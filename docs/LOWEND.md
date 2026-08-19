# Low-End & Machines faibles — Spécifications techniques

> Statut : **Référence**. Compagnon de `PERFORMANCE.md`, focalisé sur un seul objectif :
> **fonctionner correctement sur machines faibles (4-8 Go RAM, GPU intégré, disque lent) en mode desktop Tauri.**
> Dernière mise à jour : 2026-08-19

## Périmètre

- **MVP 1** : interface 2D + appels au LLM via Tauri. Les contraintes sont le poids du JS initial, la latence LLM et la consommation disque.
- **MVP 2** : WebGL + assets 3D. Les contraintes GPU/RAM/deviennent critiques.

Ce document couvre les deux MVP, avec des sections spécifiques quand nécessaire.

## Philosophie

Le document `PERFORMANCE.md` optimise la **vitesse de rendu** (GPU). Ce document attaque ce que l'autre ne couvre pas :

```
1. FIRST-RUN — télécharger et installer les assets initiaux même sur machine / réseau faibles
2. RAM        — tenir dans 4-8 Go avec d'autres apps ouvertes
3. CPU/GC     — éviter les micro-freezes (jank) fatals en visite FPS
4. RENDU      — adapter dynamiquement la charge GPU
5. PERÇU      — masquer la lenteur par la perception (feedback immédiat)
```

> **Cibles chiffrées (objectifs) :**
> - Premier écran affiché < 1 s après lancement de l'application
> - First-run (téléchargement + installation pack initial) < 30 s sur connexion modeste
> - RAM consommée < 150 Mo
> - **0 jank > 16 ms** en visite FPS (GC maîtrisé)
> - App **100 % fonctionnelle hors-ligne** après téléchargement initial des assets

---

## 0.1 Spécificités MVP 1

Le MVP 1 ne charge pas three.js ni de gros assets 3D. Ses contraintes low-end sont différentes :

| Métrique | Cible MVP 1 | Pourquoi |
|----------|-------------|----------|
| Bundle JS initial | < 120 Ko gz | Formulaires + SVG léger |
| Premier écran | < 1 s après lancement | Premier formulaire affiché rapidement |
| Temps de réponse LLM | < 15 s | Feedback utilisateur pendant l'attente |
| Appels LLM | 1–3 appels par génération | Latence maîtrisée |
| Stockage local | SQLite + filesystem | Projets et brouillons sauvegardés localement |
| Mode offline | Formulaires + historique sauvegardés | Pas de dépendance au réseau pour reprendre un brouillon |

### Adaptations pour le MVP 1

- **Génération IA désactivée si hors-ligne** ; proposition de reprendre un brouillon local.
- **Compresser les prompts** envoyés au LLM (JSON compact, pas de coordonnées brutes).
- **Mise en cache locale** du brief utilisateur et du plan généré (SQLite).
- **Thumbnails et assets légers** : pas de GLB, pas de HDRI.

---

## 1. FIRST-RUN ET ASSET STORE — installer les assets sur machine faible

### 1.1 Principe

L'application Tauri est distribuée avec un **binaire léger** (< 20 Mo). Les assets 3D initiaux sont téléchargés une seule fois lors du first-run, puis stockés localement. Cette section définit comment rendre ce first-run robuste sur machine ou connexion faibles.

### 1.2 Budgets du first-run

| Élément | Cible | Raison |
|---------|-------|--------|
| Binaire installateur | < 20 Mo | Téléchargement rapide même en ADSL |
| Pack assets initial | ~1,5 Mo compressé | meshopt + KTX2, ~25 référents |
| HDRI | < 200 Ko (.env 256) | Inclus dans le pack initial |
| Installation sur disque | < 5 s | Décompression + écriture locale |
| **Premier rendu 3D interactif** | **< 3 s** après installation des assets | Tout est local |

> La règle d'or : **le binaire démarre immédiatement** ; les assets arrivent en arrière-plan ou sur demande.

### 1.3 Code-splitting agressif (Vite)

L'app reste découpée en **chunks** chargés à la demande via `import()` dynamique :

```ts
// main.ts — chunk INITIAL (critique, ~120 Ko gz)
import { render } from 'solid-js/web';
import { App } from './ui/App';                    // shell + toolbar + plan 2D
render(() => <App />, root);
// three.js n'est PAS importé ici !

// ui/App.tsx — chunk 3D chargé quand l'utilisateur veut la 3D
async function enter3D() {
  const { Engine } = await import('./engine/Engine');      // ← dynamic import three.js
  engine = new Engine(...);
}
```

| Chunk | Quand | Contenu |
|-------|-------|---------|
| `core` | Immédiat | Solid shell, plan 2D SVG, store |
| `engine` | Clic "Générer 3D" | three.js + World + BVH |
| `visit` | Clic "Visiter" | WalkController + collision |
| `postfx` | Qualité "Photoréaliste" | postprocessing (SSAO/Bloom) |
| `catalog-*` | Drag d'un meuble | GLB du meuble spécifique (depuis le disque) |
| `llm` | Clic "Analyser" | Client LLM |

Résultat : un utilisateur qui **dessine juste un plan** ne charge **jamais three.js** (~155 Ko gz économisés).

### 1.4 Préchargement intelligent (prefetch)

Pendant que l'utilisateur dessine (CPU idle), on **prefetch** silencieusement le chunk 3D depuis le disque local :

```ts
// Dès que l'utilisateur a tracé son 1er mur fermé
const prefetch = import('./engine/Engine');  // non awaité
// → quand il cliquera "Générer 3D", le chunk est déjà en mémoire
```

### 1.5 Compression des assets

| Asset | Format | Gain |
|-------|--------|------|
| JS/CSS | Brotli (niveau 11) | -20 % vs gzip |
| GLB | meshopt | -60 % vs non-compressé |
| Textures | KTX2/Basis | -70 % vs PNG |
| HDRI | .env (256) | pré-filtré, pas de runtime |

### 1.6 Catalogue d'objets en paresse

On ne charge **que les meubles réellement utilisés** depuis le disque local, au moment du drag :

```
[Catalogue local]  [Drag canapé] → read sofa.glb (80 Ko)  → instance 3D
                                        ↑ depuis le cache local
```

L'aperçu du catalogue utilise des **thumbnails WebP statiques** (~2 Ko), pas les GLB.

### 1.7 Asset Store local

Les assets sont gérés par le backend Tauri :

| Ressource | Emplacement | Stratégie |
|-----------|-------------|-----------|
| Binaire app | `Program Files/Habiter/` ou `.app` | Installé une fois |
| Pack assets initial | `AppData/Roaming/Habiter/assets/` | Téléchargé au first-run, cache permanent |
| Assets communautaires | Même dossier | Téléchargés à la demande |
| API LLM | — | Network-only, pas de cache |

> **Résultat :** après le first-run, l'app tourne **100 % hors-ligne**. La 2e ouverture est instantanée car tout est lu depuis le disque.

### 1.8 Adaptation au first-run lent

Si le téléchargement du pack initial est lent ou impossible :

- **Téléchargement différé** : proposer "Télécharger plus tard" et utiliser un mode limité (édition 2D sans meubles 3D).
- **Pack allégé** : proposer un pack minimal (< 500 Ko) avec seulement les assets essentiels.
- **Reprise de téléchargement** : si l'app est fermée pendant le téléchargement, elle reprend à l'endroit de l'interruption.
- **Message clair** : barre de progression, vitesse, temps restant estimé.

---

## 2. RAM — tenir dans 4-8 Go

### 2.1 Disposer correctement (fuite mémoire = mort lente)

Sur machines faibles, une fuite = l'onglet se fait tuer par l'OS. Règle stricte : **tout objet three.js créé doit être disposé**.

```ts
function disposeMesh(mesh: THREE.Mesh) {
  mesh.geometry?.dispose();
  if (Array.isArray(mesh.material)) mesh.material.forEach(m => m.dispose());
  else mesh.material?.dispose();
  // dispose des textures aussi !
  traverseTextures(mesh, t => t.dispose());
}
```

Checklist système (hook `engine.dispose()` sur démontage UI) :
- [ ] Toute `geometry` créée → `dispose()`
- [ ] Tout `material` → `dispose()`
- [ ] Toute `texture` → `dispose()` (surtout KTX2)
- [ ] Toute `WebGLRenderTarget` (post-FX) → `dispose()`
- [ ] Tout `EventListener` → `removeEventListener`
- [ ] Tout `requestAnimationFrame` → `cancelAnimationFrame`

Réf : [Dispose things correctly (three.js discourse)](https://discourse.threejs.org/t/dispose-things-correctly-in-three-js/6534).

### 2.2 Pas de double résident des GLB

Quand on retire un meuble de la scène : **disposer sa géométrie**, mais **garder en cache le GLTFResource parsé** (sans sa géométrie GPU). Sinon re-drag = re-download + re-parse.

```ts
// Cache application-level : ArrayBuffer (petit) gardé, GPU resources disposées
const gltfBufferCache = new Map<string, ArrayBuffer>();
```

### 2.3 Textures partagées

Une seule texture de bois sert pour tous les meubles en bois → 1 residency GPU au lieu de 25. **Atlas de textures** (V1.2) pousse plus loin : 1 texture pour tout le catalogue.

### 2.4 Budget RAM

| Catégorie | Cible |
|-----------|-------|
| JS heap total | < 150 Mo |
| Textures GPU résidentes | < 50 Mo |
| Géométrie GPU | < 30 Mo |
| Catalogue en cache (ArrayBuffer) | < 2 Mo |

---

##  machines faibles : éviter le jank

### 3.1 Le problème du GC

JS garbage-collecte les objets temporaires. Si la boucle de rendu crache des milliers de `Vector3`/`Matrix4` par frame, le GC se déclenche toutes les ~1 s → **micro-freeze visible** (les fameux saccades périodiques en FPS).

> Témoignages : pauses WebGL ~1/s attribuées au GC ([SO GC pauses](https://stackoverflow.com/questions/4827174/javascript-garbage-collection-pauses), [GitHub three.js #9525](https://github.com/mrdoob/three.js/issues/9525)).

### 3.2 Object pooling (zéro alloc par frame)

**Principe :** pré-allouer les objets réutilisés, ne jamais `new` dans la boucle.

```ts
// engine/pools.ts
const _v3a = new THREE.Vector3();
const _v3b = new THREE.Vector3();
const _m4  = new THREE.Matrix4();
const _quat = new THREE.Quaternion();
const _ray = new THREE.Raycaster();

// Dans la boucle : on réutilise
function update(camera) {
  camera.getWorldPosition(_v3a);          // pas de new Vector3()
  const hit = raycastWalls(_ray, _v3a);   // _ray réutilisé
  // ...
}
```

**Règle ferme : zéro `new` dans `requestAnimationFrame`.** Linter ESLint custom pour le détecter.

### 3.3 Pré-typage des tableaux typés

Préférer `Float32Array` aux tableaux d'objets pour les données chaudes (instances, positions de meubles) → cache-friendly, pas de boxing GC.

### 3.4 BVH / culling off-main-thread

La construction du BVH et le culling par frame coûtent du CPU. Sur machines monocore/dualcore, ils rivalisent avec le rendu. Solution : **déporter dans un Web Worker**.

`three-mesh-bvh` supporte nativement la sérialisation worker-friendly :

```ts
// worker.ts
import { MeshBVH } from 'three-mesh-bvh';
self.onmessage = (e) => {
  const bvh = new MeshBVH(geometryFromBuffer(e.data.geometry));
  const serialized = MeshBVH.serialize(bvh);
  self.postMessage({ bvh: serialized }, [serialized.root.buffer]); // transferable
};

// main thread
const { bvh } = await callWorker(geometry);
wallsMesh.geometry.boundsTree = MeshBVH.deserialize(bvh);
```

Le main thread reste libre pour le rendu. ([three-mesh-bvh worker pattern](https://discourse.threejs.org/t/three-mesh-bvh-a-plugin-for-fast-geometry-raycasting-and-spatial-queries/26394), [Evil Martians OffscreenCanvas](https://evilmartians.com/chronicles/faster-webgl-three-js-3d-graphics-with-offscreencanvas-and-web-workers)).

> **MVP :** worker uniquement pour le build BVH (au chargement). Le culling per-frame reste main-thread (assez léger). Worker culling per-frame = V1.2.

### 3.5 Éviter les `console.log` en prod

`console.log` d'objets complexes (matrices, scènes) = sérialisation coûteuse + GC. En prod, on les strip via `vite-plugin` ou `console.clear()` en build.

---

## 4. RENDU — adapter dynamiquement la charge GPU

### 4.1 Dynamic Resolution Scaling ⭐

**Le levier le plus puissant pour GPU faibles.** On rend à une fraction de la résolution, puis on upscale. FPS quasi doublé pour qualité acceptable.

```ts
class DynamicResolution {
  private scale = 1.0;        // 1.0 = plein écran, 0.5 = moitié
  private fpsHistory: number[] = [];

  onFrame(fps: number) {
    this.fpsHistory.push(fps);
    if (this.fpsHistory.length < 60) return;
    const avg = average(this.fpsHistory);
    this.fpsHistory = [];

    if (avg < 30 && this.scale > 0.5) this.scale -= 0.1;   // saccade → baisser
    else if (avg > 55 && this.scale < 1.0) this.scale += 0.05;  // fluide → remonter

    renderer.setPixelRatio(baseDpr * this.scale);
  }
}
```

Réf : [Utsubo 100 tips 2026](https://www.utsubo.com/blog/threejs-best-practices-100-tips) (*"rendering at half resolution, then upscaling, can double frame rate"*), [Temporal Upscaling forum](https://discourse.threejs.org/t/temporal-upscaling-webgpu/89989).

### 4.2 FXAA cheap (remplace le MSAA coûteux)

Désactiver le MSAA natif (`antialias: false`) → remplacer par **FXAA** en post-process : bien moins cher, lisse les artefacts d'upscaling.

```ts
renderer = new WebGLRenderer({ antialias: false });   // pas de MSAA
// + FXAA pass dans le pipeline post-FX (qualité haute)
```

### 4.3 Distance culling / fog cutoff

Au-delà d'une distance, on **cull** les objets (pas besoin de l'armoire 20 m derrière). Combiné au fog, c'est invisible :

```ts
const FOG_CUTOFF = 25;  // m
// objets au-delà de 25 m de la caméra → visible = false
obj.visible = distanceToCamera > FOG_CUTOFF ? false : obj.visible;
```

Dans un intérieur, le fog couvre le pop-in.

### 4.4 LOD (Level of Detail) — V1.1

Chaque meuble = 3 versions (hi/med/low poly). On affiche la basse selon la distance. Gain : -50 % triangles sur objets lointains. Hors MVP (préparation des assets coûteuse), mais l'`FurnitureBuilder` est conçu pour.

### 4.5 Impostors / billboards — V1.2

Pour les objets très lointains (multi-étages), on remplace le mesh 3D par une **image plane** pré-rendue. 1 quad au lieu de milliers de triangles. Idéal pour la vue d'ensemble.

### 4.6 Qualité des ombres adaptive

Déjà couvert dans PERFORMANCE.md, mais version low-end renforcée :

| Réglage | Photoréaliste | Performant | **Ultra-low** (3G/GPU <2018) |
|---------|---------------|------------|------------------------------|
| Shadows | PCFSoft 1024 | Off | Off |
| Lights | HDRI + sun | Ambient+Hemi | Ambient only |
| Post-FX | SSAO+Bloom | Off | Off |
| Résolution | 100 % | 75 % | **50 %** + FXAA |

Le preset **Ultra-low** est un vrai sous-niveau activé automatiquement sur détection machine faible + connexion lente.

---

## 5. PERÇU — masquer la lenteur

Quand on ne peut pas être plus rapide, on **fait sentir** rapide.

### 5.1 Feedback immédiat (0-latency perçue)

| Action | Feedback immédiat |
|--------|-------------------|
| Clic "Générer 3D" | Spinner + progression % immédiatement |
| Drag meuble | Outline/placeholder affiché avant le GLB téléchargé |
| Chargement 3D | Plan 2D reste visible derrière, se "plie" en 3D (transition) |

> Règle : **toute action a un feedback < 100 ms**, même si le vrai travail prend 2 s.

### 5.2 Progressive enhancement de la scène

1. **T0** : sol + murs bas-poly (sans textures) — instant
2. **T+200 ms** : textures basses résolution apparaissent
3. **T+1 s** : ombres / post-FX si qualité haute
4. **T+2 s** : textures haute résolution (si idle)

L'utilisateur **voit sa maison tout de suite**, elle s'enrichit progressivement. Sensation de rapidité bien supérieure à un chargement bloquant.

### 5.3 Skeleton screens

Au démarrage : un squelette de l'UI (toolbar grise, zone plan) s'affiche avant même le JS. CSS critique inliné dans le HTML (< 14 Ko) → **premier paint < 1 s même sur 3G**.

### 5.4 Pas de blocking UI

Tous les chargements (LLM, GLB, 3D) sont **asynchrones et non bloquants**. L'utilisateur peut continuer à dessiner pendant que l'analyse IA tourne.

---

## 6. Détection du profil machine

Au démarrage, on classe la machine en **Low / Mid / High** et on applique un preset :

```ts
function detectProfile(): 'low' | 'mid' | 'high' {
  const ram = (navigator as any).deviceMemory;        // Go (Chrome)
  const cores = navigator.hardwareConcurrency;
  const gpu = estimateGpuTier();

  let score = 0;
  if (ram && ram >= 8) score += 2; else if (ram) score += 1;
  if (cores && cores >= 8) score += 2; else if (cores) score += 1;
  if (gpu === 'high') score += 3;

  if (score <= 2) return 'low';
  if (score <= 5) return 'mid';
  return 'high';
}
```

| Profil | Preset appliqué |
|--------|------------------|
| **low** | Ultra-low (50 % res, no shadows, no post-FX), pas de prefetch, thumbs 64px |
| **mid** | Performant (75 % res, no shadows, culling BVH), prefetch idle |
| **high** | Photoréaliste (100 % res, SSAO+Bloom, shadows 1024), prefetch agressif |

L'utilisateur peut **toujours surcharger** via le QualityToggle.

---

## 7. Plan de validation low-end

Checklist de tests à exécuter avant chaque release :

- [ ] **Premier lancement** → écran affiché < 1 s, first-run proposé clairement
- [ ] **First-run sur connexion modeste** → pack initial téléchargé et installé < 30 s
- [ ] **2e ouverture** (assets locaux) → app ouvre instant, fonctionne offline
- [ ] **Mode limité** (assets non installés) → édition 2D fonctionne, message IA offline clair
- [ ] **RAM** : app ouverte 10 min → heap < 200 Mo (pas de fuite)
- [ ] **Visite FPS 5 min** → 0 jank > 16 ms détecté (Performance tab)
- [ ] **GPU intégré (Intel UHD)** → > 30 FPS qualité Ultra-low
- [ ] **deviceMemory = 4** → pas de crash OOM
- [ ] **Layout / function `dispose()`** : scène reconstruite 20× → heap stable
- [ ] **WebView2 Windows** → app démarre sans erreur sur version récente

> Sans ces tests passés, pas de release. Ce sont les **acceptance criteria low-end**.

---

## 8. Synthèse des nouvelles optis vs PERFORMANCE.md

| Opti | Dans PERFORMANCE.md ? | Action |
|------|----------------------|--------|
| Code-splitting chunks | non couvert | **Ajouté ici §1.3** |
| Asset Store / first-run | non couvert | **Ajouté ici §1** |
| Stockage local | non couvert | **Ajouté ici §1.7** |
| Lazy catalog GLB depuis disque | mentionné V1.1 | **Promu MVP ici §1.6** |
| Object pooling / zero-alloc | non couvert | **Ajouté ici §3.2** |
| BVH build en worker | non couvert | **Ajouté ici §3.4** |
| Dynamic resolution | non couvert | **Ajouté ici §4.1** ⭐ |
| FXAA vs MSAA | non couvert | **Ajouté ici §4.2** |
| Distance culling + fog | non couvert | **Ajouté ici §4.3** |
| Progressive enhancement | non couvert | **Ajouté ici §5.2** |
| Profil machine auto | partiel (GPU) | **Étendu ici §6** (RAM+cores) |
| LOD / Impostors | roadmap | Confirmés V1.1/V1.2 |

---

## Références

### Asset Store / Tauri
- [Tauri 2 Documentation](https://tauri.app/)
- [Tauri File System Plugin](https://tauri.app/plugin/file-system/)
- [Tauri SQL Plugin](https://tauri.app/plugin/sql/)
- [Vite dynamic imports discussion](https://github.com/vitejs/vite/discussions/17730)

### CPU / GC / Workers
- [three-mesh-bvh (worker pattern)](https://discourse.threejs.org/t/three-mesh-bvh-a-plugin-for-fast-geometry-raycasting-and-spatial-queries/26394)
- [OffscreenCanvas + Workers (Evil Martians)](https://evilmartians.com/chronicles/faster-webgl-three-js-3d-graphics-with-offscreencanvas-and-web-workers)
- [Dispose things correctly in three.js](https://discourse.threejs.org/t/dispose-things-correctly-in-three-js/6534)
- [Object Pooling in Three.js](https://kingdavvid.hashnode.dev/introduction-to-object-pooling-in-threejs)
- [Low-garbage real-time JS (Construct)](https://www.construct.net/en/blogs/construct-official-blog-1/write-low-garbage-real-time-761)
- [GitHub three.js GC pauses #9525](https://github.com/mrdoob/three.js/issues/9525)

### Rendu dynamique
- [100 Three.js Tips 2026 (Utsubo)](https://www.utsubo.com/blog/threejs-best-practices-100-tips)
- [Render Half Size Then Upscale (forum)](https://discourse.threejs.org/t/render-half-size-then-upscale/13228)
- [Temporal Upscaling (forum)](https://discourse.threejs.org/t/temporal-upscaling-webgpu/89989)
- [Building Efficient Three.js Scenes (Codrops 2025)](https://tympanus.net/codrops/2025/02/11/building-efficient-three-js-scenes-optimize-performance-while-maintaining-quality/)
