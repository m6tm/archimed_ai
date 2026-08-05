# Objets 3D — Modèle Référent / Instance & interopérabilité

> Statut : **Référence**. Définit comment tout objet (meuble, lustre, accessoire, mur décoratif…) est **identifié, défini, instancié, optimisé et échangé**.
> Dernière mise à jour : 2026-06-27

## Périmètre

Ce modèle s'applique principalement au **MVP 2** (catalogue de meubles, rendu 3D, placement).  
Le **MVP 1** produit un plan 2D structuré (`Plan`) qui sert d'entrée au MVP 2. Les objets 3D (meubles) ne sont pas encore concernés au MVP 1.

## 0. Pourquoi ce document

Trois besoins futurs imposent un modèle propre **dès le MVP 2** :

1. **IA image→3D** — une IA construira des objets depuis une image (Meshy/Tripo/Rodin → GLB+PBR).
2. **Export** — traiter un objet dans Blender/Maya/etc.
3. **Import** — ramener un objet 3D pris ailleurs (Sketchfab, bibliothèque perso).

Si on modélise mal maintenant (ex : l'instance porte sa propre géométrie), on devra **refaire tout le catalogue et le moteur** dans 6 mois. Ce doc fige le bon modèle.

---

## 1. Le pattern Référent / Instance

> Inspiré d'Unity (Prefab/GameObject), Blender (data-block lié), Unreal (Asset/Actor), USD (prim reference).

On sépare **ce que l'objet est** (définition, partagée) de **où et comment il est posé** (instance, propre à la scène).

```
┌─────────────────────────────────────────────────────────────────────┐
│  ASSET DEFINITION  (référent — partagé, indexé, 1 fois par objet)   │
│  ─────────────────────────────────────────────────────────────────  │
│  id stable · GLB pivot · PBR · métadonnées · LODs · origine physique │
│         ▲                                                             │
│         │ référencé par (1→N)                                         │
│         │                                                             │
│  ┌──────┴───────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────┐ │
│  │  INSTANCE 1  │  │  INSTANCE 2  │  │  INSTANCE 3  │  │ ...      │ │
│  │  (PlacedObj) │  │  (PlacedObj) │  │  (PlacedObj) │  │          │ │
│  │ salon, x,y,z │  │ chambre, ... │  │ cuisine ...  │  │          │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────┘ │
│  (légère : juste position/rotation/scale + assetId)                  │
└─────────────────────────────────────────────────────────────────────┘
```

**Avantages :**
- **Indexation** : tout objet indexé par son `assetId` stable → recherche, stats, remplacement global, export.
- **Mémoire** : 1 GLB en VRAM, N instances = N matrices (voir InstancedMesh, PERFORMANCE.md §2).
- **MAJ globale** : changer le GLB du référent → toutes les instances suivent.
- **Interopérabilité** : on exporte le référent (GLB/USD) ou une instance (GLB avec transform) indépendamment.

---

## 2. Format pivot interne : **GLB (glTF 2.0)**

### Pourquoi glTF 2.0 (et pas une "3.0" qui n'existe pas)

- glTF 2.0 = **standard ISO/IEC 12113:2022**, le "JPEG du 3D" ([Khronos glTF Now and Next](https://www.khronos.org/blog/gltf-now-and-next)).
- **Pas de glTF 3.0 prévu** ; Khronos étend via **extensions** : `KHR_materials_*`, `KHR_lights_punctual`, `KHR_texture_basisu` (KTX2), `KHR_mesh_quantization`, et à venir Physics/Interactivity/LOD-streaming/glX.
- **L'IA image→3D sort du GLB par défaut** (Meshy v6, Tripo, Rodin, TRELLIS) avec PBR embarqué → absorption native dans notre pipeline.
- Supporté nativement par three.js (`GLTFLoader`), Blender, Unity, Unreal.

### Règles de stockage interne

| Règle | Valeur |
|-------|--------|
| Format pivot | **GLB** (conteneur binaire, geometry + PBR + animations embarqués) |
| Compression géométrie | **`KHR_mesh_quantization`** + **meshopt** (`EXT_meshopt_compression`) |
| Textures | **KTX2** (`KHR_texture_basisu`), PBR complet (baseColor, normal, metallicRoughness, occlusion, emissive) |
| Échelle | Unité GLB = **1 mètre** (aligné avec notre scène) |
| Origine | Pied de l'objet au point (0,0,0), axe +Y vers le haut (convention glTF) |
| Up/Forward | Y-up, -Z forward (glTF natif) → pas de conversion three.js |

> **Tous les GLB entrants (catalogue, IA, import) sont normalisés** vers ces conventions par un `AssetNormalizer` (§6). C'est ce qui garantit l'interopérabilité.

---

## 3. Schéma de données

TypeScript central dans `lib/assets/types.ts`. **Sérialisable JSON** (sauvegarde, export, partage).

### 3.1 AssetDefinition (le référent)

```ts
// lib/assets/types.ts

/** Type d'objet — gouverne les comportements (collisions, éclairage, snapper…) */
export type AssetCategory =
  | 'seating'      // canapé, chaise, fauteuil
  | 'table'        // table, bureau, chevet
  | 'storage'      // armoire, commode, étagère
  | 'bed'          // lit, matelas
  | 'appliance'    // frigo, four, lave-linge
  | 'sanitary'     // baignoire, vasque, WC
  | 'lighting'     // lustre, lampe sur pied, spot
  | 'decor'        // plante, cadre, tapis
  | 'electronics'  // TV, enceinte
  | 'structural'   // mur, colonne, escalier ( futur )
  | 'custom';      // importé / généré IA

/** Source de l'asset — traçabilité pour l'import/IA */
export type AssetSource =
  | { kind: 'builtin'; catalogVersion: string }                       // livré avec l'app
  | { kind: 'user-import'; originalFile: string; importedAt: number } // import utilisateur
  | { kind: 'ai-generated'; model: string; prompt?: string; seed?: string; generatedAt: number } // Meshy/Tripo…
  | { kind: 'external'; url: string; license?: string };              // Sketchfab etc.

/** Une version de LOD (Level of Detail) */
export interface LodLevel {
  level: 0 | 1 | 2;        // 0=high, 1=mid, 2=low
  glb: string;             // chemin ou hash du GLB
  triangleCount: number;
  sizeBytes: number;
}

/** Propriétés physiques — collisions, ergonomie, IA structure (V2) */
export interface PhysicalProps {
  dimensions: { w: number; h: number; d: number };  // bbox, mètres (Y-up)
  weightKg?: number;
  /** Formes de collision simplifiées (pas la géométrie visible) */
  colliders: Collider[];
  /** Points de préhension / d'ancrage (prises, accroches mur…) — futur */
  anchors?: Anchor[];
}

export type Collider =
  | { kind: 'box'; halfExtents: [number, number, number] }
  | { kind: 'cylinder'; radius: number; height: number }
  | { kind: 'convexHull'; points: [number, number, number][] }; // pour IA/moteur physique V2

export interface Anchor {
  id: string;
  type: 'wall' | 'floor' | 'ceiling' | 'power' | 'water';
  position: [number, number, number];
}

/** Propriétés de rendu (matières, surfaces) — pour PBR cohérent et thermique (futur) */
export interface RenderProps {
  materials: MaterialInfo[];
  /** Tag de surface pour l'IA style-transfer (V2) */
  primarySurface: 'wood' | 'fabric' | 'metal' | 'glass' | 'plastic' | 'stone' | 'plant' | 'other';
}

export interface MaterialInfo {
  name: string;
  surface: RenderProps['primarySurface'];
  baseColor?: string;     // hex, pour swatch catalogue
  pbrTextures?: boolean;  // le GLB embarque-t-il des maps PBR
}

/** Propriétés d'éclairage (pour lighting / lustres) */
export interface LightProps {
  kind: 'none' | 'point' | 'spot' | 'directional' | 'emissive';
  intensity?: number;     // lumens ou normalisé
  color?: string;         // hex (température couleur en futur)
  range?: number;
}

/** Balises sémantiques — indexation + recherche + IA structure */
export interface SemanticTags {
  rooms?: string[];            // ["salon","bureau"] où ça s'intègre
  styles?: string[];           // ["scandinave","industriel"]
  functions?: string[];        // ["assis","repos","cuisine"]
  era?: string;                // "1950" pour style retro (futur)
}

/**
 * RÉFÉRENT — 1 par objet distinct. Stable, indexé, partagé.
 */
export interface AssetDefinition {
  /** Identifiant stable et unique (voir §4 — stratégie d'ID) */
  id: string;
  /** Version de la définition (incrémentée si le GLB change) */
  revision: number;
  /** Slug lisible : "sofa_3p_scandi" */
  slug: string;
  /** Nom affiché : "Canapé 3 places scandinave" */
  name: string;
  category: AssetCategory;

  /** Format pivot + normalisé */
  format: 'glb';
  /** Hash du contenu GLB (content-addressable, voir §5) */
  contentHash: string;
  /** Chemin/cache du GLB normalisé */
  glb: string;                 // ex: "/assets/sofa_3p_scandi.glb" ou "ipfs://…" (futur)

  /** LODs — au moins 1 (level 0). Permet le LOD dynamique (PERFORMANCE §4.4) */
  lods: LodLevel[];

  /** AABB englobante en mètres (Y-up, origine pied) */
  bbox: { min: [number, number, number]; max: [number, number, number] };

  /** Propriétés détaillées — cf. blocs ci-dessus */
  physical: PhysicalProps;
  render: RenderProps;
  light?: LightProps;          // présent si category='lighting' ou émissif
  semantics?: SemanticTags;

  /** Aperçu catalogue (thumbnail WebP, ~2 Ko) */
  thumbnail: string;

  /** Source/traçabilité (builtin / import / IA / externe) */
  source: AssetSource;

  /** Licence (important pour import/externe) */
  license?: string;

  createdAt: number;
  updatedAt: number;
}
```

### 3.2 PlacedObject (l'instance)

```ts
/** Une instance d'un asset dans la scène — légère et mutable */
export interface PlacedObject {
  /** ID unique de l'instance (UUID) */
  id: string;
  /** ★ Référence au référent — JAMAIS de géométrie inline */
  assetId: string;
  /** Révision du référent au moment du placement (anti-dérive) */
  assetRevision: number;

  /** Transform de l'instance (mètres, radians) */
  transform: {
    position: [number, number, number];
    rotationY: number;          // on contraint à Y pour le MVP (pas de tilt)
    scale: [number, number, number]; // défaut [1,1,1]
  };

  /** Pièce d'appartenance (assignée par IA structure ou manuelle) */
  roomId?: string;

  /** Overrides locaux (couleur peinte par l'utilisateur, etc.) — V2 */
  overrides?: ObjectOverrides;

  /** Métadonnées d'instance */
  placedAt: number;
}

export interface ObjectOverrides {
  /** Ex: repeindre un meuble — remplace la baseColor du matériau 0 */
  materialColors?: Record<number, string>;
  // futur : swap de matériau, variantes…
}
```

### 3.3 AssetRegistry (l'index)

```ts
/** Registre central des référents — l'index */
export interface AssetRegistry {
  /** Map assetId → AssetDefinition (cache application) */
  byId: Map<string, AssetDefinition>;
  /** Index secondaires pour la recherche rapide */
  byCategory: Map<AssetCategory, Set<string>>;
  bySlug: Map<string, string>;
  byContentHash: Map<string, string>;   // déduplication
  /** GLB déjà chargés en VRAM */
  loaded: Map<string, GLTFResource>;     // assetId → resource three.js
}
```

---

## 4. Stratégie d'identifiants (le "référent" stable)

| Source | Format d'ID | Exemple | Propriété garantie |
|--------|-------------|---------|--------------------|
| **builtin** | `builtin:<slug>@<rev>` | `builtin:sofa_3p_scandi@1` | Stable, versionné, lisible |
| **user-import** | `usr:<userId>:<contentHash8>` | `usr:abc123:f3a91c02` | Déduit du contenu, dédupliqué |
| **ai-generated** | `ai:<model>:<contentHash8>` | `ai:meshy-v6:7b2e44a1` | Traçable jusqu'au modèle/prompt |
| **external** | `ext:<domain>:<remoteId>` | `ext:sketchfab:d3f2a1` | Référence stable à la source |

> **Principe :** l'ID encode la **source** + un **hash de contenu**. Cela rend chaque objet :
> - **Indexable** (recherche par catégorie, slug, hash)
> - **Dédupliqué** (deux imports du même GLB = même asset)
> - **Traçable** (on sait toujours d'où vient un objet : catalogue, IA, import)
> - **Stable** (renommer un meuble ne change pas son ID)

---

## 5. Content-addressable storage (dédup + intégrité)

Le **hash du GLB** (`contentHash`) sert de clé de stockage :

```
/assets/objects/f3/f3a91c02...glb     ← sofa_3p_scandi
/assets/objects/7b/7b2e44a1...glb     ← chaise générée par Meshy
```

- **Déduplication** : deux imports du même fichier = 1 fichier sur disque/CDN.
- **Intégrité** : si le hash au chargement ≠ hash attendu → refus (anti-corruption).
- **Cache PWA** : clé de cache stable (LOWEND.md §1.7), versionnée naturellement.
- **Futur IPFS/distribué** : le hash est déjà une adresse content-addressable.

```ts
function contentHashOf(glbBuffer: ArrayBuffer): string {
  return sha256(glbBuffer).slice(0, 16);   // 16 hex = collision négligeable
}
```

## 6. AssetNormalizer — garantir des GLB conformes

**Tout GLB entrant** (catalogue, import, IA) passe par un normalisateur qui garantit les conventions :

```ts
async function normalizeAsset(rawGlb: ArrayBuffer, meta: NormalizeMeta): Promise<NormalizedAsset> {
  // 1. Parser le GLB
  const doc = await parseGLB(rawGlb);

  // 2. Recentrer : pied de l'objet à l'origine (Y=0), axe Y-up
  centerToFloor(doc);

  // 3. Ré-échelle vers l'échelle déclarée (1 unité = 1 mètre)
  rescaleToMeters(doc, meta.declaredDimensions);

  // 4. Compresser : KHR_mesh_quantization + EXT_meshopt_compression
  await compressGeometry(doc);

  // 5. Textures → KTX2 (KHR_texture_basisu)
  await convertTexturesToKTX2(doc);

  // 6. Générer les LODs (decimate) → lods[0..2]
  const lods = await generateLods(doc, { ratios: [1.0, 0.5, 0.25] });

  // 7. Calculer bbox + colliders simplifiés (convex hull / box fit)
  const bbox = computeBBox(doc);
  const colliders = fitColliders(doc, meta.colliderHint ?? 'box');

  // 8. Générer le thumbnail (rendu hors-ligne, WebP ~2 Ko)
  const thumbnail = await renderThumbnail(doc);

  // 9. Hash + sérialisation finale
  const outGlb = serializeGLB(lods[0]);
  return {
    glbBuffer: outGlb,
    contentHash: sha256(outGlb).slice(0, 16),
    lods, bbox, colliders, thumbnail,
  };
}
```

> **Build-time pour le catalogue builtin**, **runtime (dans un Worker) pour import/IA**. Le résultat est **toujours** une `AssetDefinition` conforme — l'application ne manipule jamais de GLB "brut".

---

## 7. Propriétés d'optimisation (pour le détail maximal)

L'utilisateur veut des objets **les plus détaillés possible**. Le modèle le permet **sans sacrifier la perf** via :

### 7.1 LODs natifs (multi-résolution)

Chaque référent porte **3 niveaux** :

| LOD | Triangles | Usage |
|-----|-----------|-------|
| 0 (high) | détail max (ex: 8k) | Vue rapprochée / qualité "Photoréaliste" |
| 1 (mid) | ~50% | Distance moyenne |
| 2 (low) | ~25% | Très lointain / qualité "Ultra-low" |

→ On garde le **détail maximal en source**, le moteur choisit le bon niveau par distance + profil machine. On ne **perd** jamais de détail, on l'affiche quand on peut.

### 7.2 Mipmaps + KTX2

Textures PBR en KTX2 avec mipmaps → net en gros plan, léger au loin.

###  displacement / normal maps (V1.1)

Pour un détail géométrique perçu sans exploser le poly-count : une géométrie basse + normal map détaillée. Idéal pour les objets IA générés.

### 7.4 Compression sans perte visible

`KHR_mesh_quantization` (16-bit) + meshopt → **-60% de taille** sans perte visible.

### 7.5 Variantes matérielles (V2)

Le référent peut porter plusieurs jeux de matériaux (chêne, noyer, laqué) → l'utilisateur choisit sans recharger le GLB.

---

## 8. Import / Export — interopérabilité

### 8.1 Import (vers le référent)

```
[Fichier .glb/.gltf/.fbx/.obj fourni]
        │
        ▼
[Parseur] → [AssetNormalizer] → AssetDefinition (format pivot GLB normalisé)
        │
        ▼
[AssetRegistry] → instance-able dans la scène
```

| Format entrant | Parseur | Note |
|----------------|---------|------|
| **GLB/glTF** | `GLTFLoader` natif | Pivot, perte zéro |
| FBX | `FBXLoader` | Conversion matériaux → PBR (perte possible) |
| OBJ | `OBJLoader` (+ MTL) | Géométrie seule, matériau à recréer |
| USD/USDZ | `USDLoader` (three r16x+) | Pipeline pro (Blender/Omniverse) |

> **Contrat :** quel que soit le format entrant, le résultat est **toujours** un GLB normalisé + une `AssetDefinition`. L'app ne code que contre ce contrat.

### 8.2 Export (depuis le référent ou une instance)

| Besoin | Format | Contenu |
|--------|--------|---------|
| Vers Blender/Maya | **GLB** ou **glTF+bin** | Le référent complet (géométrie+PBR) |
| Pipeline pro/industriel | **USD** | Pour Omniverse / film (roundtrip glTF↔USD) |
| Unity/Unreal | **FBX** (+ maps externes) | Engine-friendly |
| Apple AR / Quick Look | **USDZ** | iOS |
| Sauvegarde projet | **JSON** | Référents (par référence) + instances |
| Partage web | lien (futur) | Référent sur CDN, instances dans l'URL |

```ts
async function exportObject(
  target: { kind: 'asset'; assetId: string } | { kind: 'instance'; instanceId: string },
  format: 'glb' | 'gltf' | 'usd' | 'fbx' | 'usdz'
): Promise<Blob> {
  // 1. Résoudre le référent (et appliquer les overrides d'instance si besoin)
  const asset = registry.get(target.assetId);
  const doc = applyOverrides(await loadGLB(asset.glb), target.overrides);

  // 2. Convertir vers le format demandé
  switch (format) {
    case 'glb':  return serializeGLB(doc);
    case 'gltf': return serializeGLTF(doc);     // + .bin + textures
    case 'usd':  return convertGLBtoUSD(doc);    // roundtrip glTF→USD
    case 'fbx':  return convertGLBtoFBX(doc);
    case 'usdz': return convertGLBtoUSDZ(doc);
  }
}
```

> L'export d'une **instance** applique son transform + overrides avant sérialisation → on peut exporter "ce canapé-là, repeint, dans cette position" vers Blender.

### 8.3 Roundtrip complet

```
[Image] ──IA (Meshy)──▶ [GLB] ──normalize──▶ [AssetDefinition] ──┐
                                                                  │
              ┌───────────────────────────────────────────────────┤
              ▼                                                   ▼
      [Instance placée]  ◀──  import/edit  ──  [Blender: edit]  ◀── [Export GLB/USD]
```

Le **même format pivot (GLB)** et le **même contrat (AssetDefinition)** servent l'entrée IA, l'import et l'export. C'est ce qui rend le système futur-proof.

---

## 9. Catalogue builtin (MVP) — ~25 référents

`lib/assets/builtin/*.json` — un fichier JSON par référent, ou un index central :

```json
// builtin/catalog.json
{
  "version": "1.0.0",
  "assets": [
    {
      "id": "builtin:sofa_3p_scandi@1",
      "revision": 1,
      "slug": "sofa_3p_scandi",
      "name": "Canapé 3 places scandinave",
      "category": "seating",
      "format": "glb",
      "contentHash": "f3a91c02...",
      "glb": "/assets/objects/f3/f3a91c02.glb",
      "lods": [
        { "level": 0, "glb": "/assets/objects/f3/f3a91c02.glb",        "triangleCount": 8200, "sizeBytes": 78000 },
        { "level": 1, "glb": "/assets/objects/f3/f3a91c02_lod1.glb",   "triangleCount": 4100, "sizeBytes": 41000 },
        { "level": 2, "glb": "/assets/objects/f3/f3a91c02_lod2.glb",   "triangleCount": 1800, "sizeBytes": 22000 }
      ],
      "bbox": { "min": [0, 0, 0], "max": [2.1, 0.85, 0.9] },
      "physical": {
        "dimensions": { "w": 2.1, "h": 0.85, "d": 0.9 },
        "weightKg": 42,
        "colliders": [{ "kind": "box", "halfExtents": [1.05, 0.425, 0.45] }]
      },
      "render": {
        "materials": [
          { "name": "tissu", "surface": "fabric", "baseColor": "#9CA89B", "pbrTextures": true },
          { "name": "pieds", "surface": "wood",  "baseColor": "#6B4A2B", "pbrTextures": true }
        ],
        "primarySurface": "fabric"
      },
      "semantics": {
        "rooms": ["salon", "bureau"],
        "styles": ["scandinave", "minimaliste"],
        "functions": ["assis", "repos"]
      },
      "thumbnail": "/assets/thumbs/sofa_3p_scandi.webp",
      "source": { "kind": "builtin", "catalogVersion": "1.0.0" },
      "license": "CC-BY-4.0",
      "createdAt": 1719500000000,
      "updatedAt": 1719500000000
    }
  ]
}
```

**Sélection MVP (25)** — couvre les pièces types (cf. MVP.md §8) :
- **Seating** : canapé 3p, fauteuil, chaise (instanced ×4)
- **Table** : table basse, table + 4 chaises, bureau, chevet (instanced ×2)
- **Storage** : armoire, commode, étagère, placard
- **Bed** : lit 160, lit 90
- **Appliance** : frigo, lave-linge
- **Sanitary** : baignoire, vasque, WC
- **Lighting** : lustre, lampe sur pied, spot (instanced)
- **Decor** : plante (instanced), tapis, cadre
- **Electronics** : TV + meuble

> Les **lustres/accessoires** que tu mentionnes entrent dans `category: 'lighting'` / `'decor'` — le modèle les couvre nativement.

---

## 10. Cycle de vie d'un asset (etat machine)

```
                ┌─────────────────────────────────────────────┐
                │  SOURCE (1 seul bloc d'entrée)               │
                │  builtin | import | IA | external            │
                └────────────────────┬────────────────────────┘
                                     ▼
                          [AssetNormalizer]  (Worker)
                                     │
                          GLB normalisé + LODs + bbox + colliders + thumb
                                     ▼
                ┌─────────────────────────────────────────────┐
                │  ASSET DEFINITION  (référent)                │
                │  id stable · hash · GLB · props              │
                └────────────────────┬────────────────────────┘
                                     ▼
                          [AssetRegistry]  (index)
                                     │
                          ┌──────────┴──────────┐
                          ▼                     ▼
                  [PlacedObject]          [Export]
                  (instance légère)       (GLB/USD/FBX/USDZ)
```

Un seul point d'entrée, un seul type de référent, un seul index. **Tout passe par le même tuyau.**

---

## 11. Implémentation côté moteur three.js

```ts
// engine/builders/FurnitureBuilder.ts
class FurnitureBuilder {
  constructor(private registry: AssetRegistry, private scene: THREE.Group) {}

  /** Instancie (ou réutilise) un référent dans la scène */
  async place(obj: PlacedObject): Promise<THREE.Object3D> {
    // 1. Charger le GLB du référent si pas en cache
    let resource = this.registry.loaded.get(obj.assetId);
    if (!resource) {
      const asset = this.registry.byId.get(obj.assetId)!;
      resource = await this.loadGLB(asset.glb, asset.lods);   // LOD chooser intégré
      this.registry.loaded.set(obj.assetId, resource);
    }

    // 2. Instancier (clone du template, partage la géométrie VRAM)
    const node = resource.cloneWithLOD();
    node.position.fromArray(obj.transform.position);
    node.rotation.y = obj.transform.rotationY;
    node.scale.fromArray(obj.transform.scale);

    // 3. Overrides (couleur peinte, etc.)
    applyOverrides(node, obj.overrides);

    // 4. Collision (colliders simplifiés, pas la géométrie visible)
    node.userData.colliders = this.registry.byId.get(obj.assetId)!.physical.colliders;
    node.userData.assetId = obj.assetId;     // ← lien retour vers le référent
    node.userData.instanceId = obj.id;

    this.scene.add(node);
    return node;
  }
}
```

**Points clés :**
- **VRAM partagée** : N instances d'un même référent partagent **1 GLB** (clone des matériaux/géométrie via three.js `SkeletonUtils.clone` ou InstancedMesh).
- **LOD auto** : `node.cloneWithLOD()` attache un `THREE.LOD` qui choisit selon la distance.
- **Lien retour** : `node.userData.assetId` permet de retrouver le référent depuis la scène (clic → fiche objet).
- **Dédup** : le `registry.loaded` évite de recharger un GLB déjà en VRAM.

---

## 12. Impact sur les docs existants

| Document | Impact |
|----------|--------|
| `MVP.md` §3 (types) | `PlacedObject` devient référence vers `assetId` (au lieu de `catalogId`), ajout `AssetDefinition` |
| `MVP.md` §8 (catalogue) | Réécrit en `AssetDefinition[]`, ~25 référents |
| `PERFORMANCE.md` §2 (batching) | InstancedMesh piloté par `registry` : instances du même `assetId` |
| `PERFORMANCE.md` §4.4 (LOD) | LOD porté par le référent, choix par distance |
| `LOWEND.md` §1.6 (GLB paresseux) | Chargement paresseux par `assetId`, cache content-addressed |
| `ROADMAP.md` | IA image→3D + import/export deviennent naturels (V2/V3) |

---

## 13. Décisions ouvertes (à valider)

| # | Décision | Recommandation | Pourquoi |
|---|----------|----------------|---------|
| O1 | Format pivot | **GLB (glTF 2.0)** | Standard ISO, sortie native de l'IA, support three.js |
| O2 | Hash de contenu | **SHA-256 16 hex** | Dédup + intégrité + future IPFS |
| O3 | Compression | **meshopt + KTX2** | Meilleur ratio que DRACO, decode GPU |
| O4 | Conversion import (FBX/OBJ/USD) | **côté build pour builtin, Worker runtime pour import** | Évite de plomber le main thread |
| O5 | Stockage import/IA (V2) | **IndexedDB** (cache) + CDN (futur) | Offline + scalable |
| O6 | LODs générés | **3 niveaux (1.0/0.5/0.25)** au build pour builtin | Détail max préservé |
| O7 | Overrides d'instance | **V2** (couleur peinte) | MVP = pas d'override, instanciation pure |

---

## Références

- [Khronos — glTF: Now and Next](https://www.khronos.org/blog/gltf-now-and-next) — roadmap extensions (Physics, Interactivity, LOD-streaming, glX, Gaussian Splatting ~Q2 2026)
- [Khronos glTF (GitHub)](https://github.com/khronosgroup/gltf) — spec ISO/IEC 12113:2022
- [SIGGRAPH 2025 — USD & glTF Interoperability](https://www.youtube.com/watch?v=_quZ1QRysSU) — roundtrip glTF↔USD, alignement PBR
- [Meshy — 3D File Formats](https://www.meshy.ai/blog/3d-file-formats) — formats d'export IA
- [Meshy Docs — Retexture API](https://docs.meshy.ai/en/api/retexture) — entrées GLB/glTF/OBJ/FBX/STL
- [Best AI 3D Generators 2026](https://medium.com/ideas-with-wings/best-ai-3d-model-generators-in-2026-tripo-ai-vs-meshy-rodin-kaedim-and-more-7eea7b05eb11) — Meshy/Tripo/Rodin/TRELLIS sortent GLB+PBR
- [pmndrs/postprocessing](https://github.com/pmndrs/postprocessing) — pour thumbnails/rendus
