# MVP 2 — De l'éditeur 2D à la visite 3D

> Statut : **À valider avant codage**  
> Suite logique du MVP 1. Le MVP 2 transforme un plan technique 2D (généré par IA dans le MVP 1 ou dessiné manuellement) en maison 3D photoréaliste, meublable et visitable.

---

## 0. Objectif et philosophie

**Démontrer UNE seule chose** : à partir d'un plan 2D existant, le geste *« j'affine, je génère la 3D, je meuble, je visite »* est magique et fonctionne de manière fluide.

Le plan 2D d'entrée peut provenir :
- du **MVP 1** (plan IA généré et exporté)
- d'un **dessin manuel** dans l'éditeur SVG (mode fallback / création libre)

**Performance = feature de rang 0.** L'objectif reste de réduire au maximum le matériel requis et d'augmenter la fluidité, surtout en visite FPS.

> Principes directeurs :
> 1. **N'afficher que ce qui est dans le champ de la caméra** (+ non caché par un mur).
> 2. **Ne télécharger que ce qui est utilisé** (code-splitting + GLB paresseux).
> 3. **Ne rendre que quand ça bouge** (render-on-demand).
> 4. **Adapter la charge au matériel et à la connexion** (profil auto).

---

## 0.1 Positionnement par rapport au MVP 1

```
MVP 1 : Assistant IA → Plan technique 2D éditable
            ↓
MVP 2 : Éditeur 2D → IA structure → Maison 3D → Meubles → Visite FPS
```

Le MVP 2 ne redemande pas à l'utilisateur de dessiner un plan from scratch. Il part d'un plan existant qu'il peut affiner, enrichir de sémantique (noms de pièces), convertir en 3D, meubler et visiter.

---

## 1. Stack technique

Identique à celle définie dans l'ancien `MVP.md` (r17x / r18x, SolidJS, Vite, TypeScript strict, three-mesh-bvh, meshoptimizer, postprocessing, Workbox).

> Voir `docs/MVP1.md` pour la justification des choix. Les raisons de ne pas utiliser React/R3F/Threlte et de rester sur WebGL2 par défaut restent valables.

---

## 2. Architecture du monorepo

L'architecture reste centrée sur la séparation UI (SolidJS) / Moteur 3D (vanilla three.js).

### Adaptation par rapport au MVP 1

- Le `planStore` peut être initialisé soit par un import depuis le MVP 1, soit vide (mode dessin manuel).
- L'éditeur 2D SVG devient un outil d'**affinage** plutôt que le point d'entrée principal.
- Le bouton **« Générer 3D »** est désormais le moment clé qui transforme le plan validé en scène three.js.

### Principe de séparation

```
┌────────────────────────────────────────────────────────────────┐
│  COUCHE UI (SolidJS, signaux)                                   │
│  Toolbar, catalogue, plan 2D SVG, panneaux                      │
│         │  createSignal / createStore                           │
│         │  → mutation → déclenche rebuild ciblé du moteur 3D     │
└─────────┬──────────────────────────────────────────────────────┘
          │  planSnapshot (immutable) + engine.setPlan(snapshot)
          ▼
┌────────────────────────────────────────────────────────────────┐
│  COUCHE MOTEUR (vanilla three.js, impératif)                    │
│  Engine.ts : WebGLRenderer + render-on-demand + stats          │
│  World.ts  : reconstruit la scène uniquement sur diff          │
│  Culling   : BVH + occlusion → ne rend que le visible          │
└────────────────────────────────────────────────────────────────┘
```

---

## 3. Modèle de données

Le modèle de données reste compatible avec le MVP 1. Le type `Plan` est le contrat central.

```ts
export interface Plan {
  walls: Wall[];
  openings: Opening[];
  rooms: Room[];
  objects: PlacedObject[];
  meta: {
    name: string;
    createdAt: number;
    updatedAt: number;
    source?: 'manual' | 'ai-generated-plan';   // ← héritage MVP 1
    levels?: LevelMeta[];                       // ← héritage MVP 1
  };
}
```

> ⭐ Le modèle **Référent / Instance** des objets (`AssetDefinition` / `PlacedObject`) reste inchangé. Voir `docs/OBJECTS.md`.

---

## 4. Pipeline 2D → IA structure → 3D

La pipeline s'applique maintenant au plan importé du MVP 1 :

```
   Plan (MVP 1)  ──▶  Étape A — GÉOMÉTRIE (déterministe)  ──▶  Étape B — SÉMANTIQUE (LLM)
   (walls + openings)  detectRooms() : polygones fermés      enrichWithLLM() : noms + types
                       + aire + adjacence                    + suggestions (Zod-validé)
```

### Différence avec l'ancien MVP

- Dans l'ancien MVP, l'utilisateur dessinait tout manuellement.
- Dans le MVP 2, le plan est déjà structuré (murs, ouvertures, pièces). L'IA structure sert surtout à **vérifier et nommer** les pièces, et à proposer des ajustements.

### 4.1 Règle de fidélité 3D au plan 2D

> **Le plan 3D est une extrusion géométriquement fidèle du plan 2D du MVP 1.**

Cette règle est non négociable :

- Chaque mur du plan 2D devient un mur 3D de même emprise, longueur et épaisseur.
- Chaque ouverture (porte/fenêtre) conserve sa position et ses dimensions.
- L'enveloppe extérieure de chaque niveau 3D correspond exactement au polygone du niveau 2D.
- Les poteaux du plan 2D sont matérialisés en 3D à la même position.
- Le MVP 2 ne réinterprète pas la géométrie : il l'extrude avec une hauteur d'étage paramétrable (défaut 2,50 m).

Seuls les éléments ajoutés par le MVP 2 sont de nouvelles données :

- hauteur des étages
- épaisseur des sols/plafonds
- matériaux et textures
- éclairage et post-processing
- meubles et objets

---

## 5. Spécifications performance

Inchangées par rapport à l'ancien `MVP.md` :

- **> 30 FPS** sur GPU intégré en visite FPS
- **> 60 FPS** sur GPU dédié
- **GPU idle** en édition
- Culling BVH, batching, render-on-demand, assets compressés, qualité adaptative

> Voir `docs/PERFORMANCE.md` pour le détail.

---

## 6. Catalogue d'objets

Le catalogue de ~25 référents reste identique. Voir `docs/OBJECTS.md`.

---

## 7. Plan de tâches ordonné (6 sprints)

Chaque sprint reste un livrable démontrable, mais le point de départ change : on part d'un plan importé plutôt que d'une page blanche.

### Sprint 0 — Setup

- pnpm workspace + `apps/web` (Vite + SolidJS-TS)
- `Engine.ts` : renderer + render-on-demand + HUD fps
- **Démo :** cube qui ne rend que si on le fait bouger

### Sprint 1 — Import et édition 2D

- Chargement d'un plan depuis le MVP 1 (import JSON)
- `PlanEditor` SVG + grille snap 20cm
- `WallTool`, `DoorTool`, `WindowTool`, `DimensionLabel`
- `planStore` (Solid store) + persistance localStorage
- Tests `detectRooms`
- **Démo :** on charge un plan MVP 1 et on l'affine

### Sprint 2 — 3D + extrusion + culling

- `WallBuilder` : extrusion → BatchedMesh
- `FloorBuilder`, `OpeningBuilder` (mesh segmenté)
- `World.ts` + reconcile (diff par id)
- `BvhWorld` + `OcclusionCuller`
- `Lighting` HDRI + OrbitController
- Bouton **« Générer 3D »**
- **Démo :** le plan devient maison 3D, culling visible au HUD

### Sprint 3 — IA structure (LLM)

- `detectRooms()`, `llm/client.ts`, `enrichWithLLM` + Zod + fallback
- `StructurePanel` : pièces nommées éditables
- **Démo :** les pièces s'appellent "Salon", "Cuisine", "Chambre"

### Sprint 4 — Catalogue + placement

- Catalogue ~25 référents optimisés
- `CatalogPanel` drag&drop, `FurnitureBuilder` (InstancedMesh)
- TransformControls move/rotate, collision AABB
- **Démo :** on meuble le plan

### Sprint 5 — Visite + photoréalisme + polish

- `WalkController` : PointerLock + ZQSD + collision BVH
- `PostFX` (SSAO+Bloom+Vignette, qualité haute only)
- `QualityToggle` + détection auto + downgrade auto
- Slider heure du jour
- `ProjectStore` : save/load multi-projets, export JSON
- **Démo finale :** la boucle complète, fluide

> Le détail ordonné avec critères de sortie phase par phase figure dans `docs/REALIZATION.md`.

---

## 8. Definition of Done

Le MVP 2 est livré quand, en **< 2 minutes**, un bêta-testeur peut :

1. ✅ Charger un plan provenant du MVP 1
2. ✅ L'affiner si besoin (ajouter/supprimer un mur, une ouverture)
3. ✅ Cliquer **« Analyser »** → pièces nommées par l'IA
4. ✅ Cliquer **« Générer 3D »** → maison photoréaliste fluide
5. ✅ Glisser 5-6 meubles et les positionner
6. ✅ Cliquer **« Visiter »** → FPS > 30 même sur GPU intégré
7. ✅ Recharger → projet intact

**Tests perf obligatoires (HUD dev) :**
- GPU intégré + qualité "Performant" : **> 30 FPS** en visite
- GPU dédié + qualité "Photoréaliste" : **> 60 FPS**
- Mode édition : **GPU idle** entre actions

---

## 9. Risques et mitigations

Les risques sont identiques à ceux de l'ancien MVP, avec un risque supplémentaire :

| Risque | Prob. | Impact | Mitigation |
|--------|-------|--------|------------|
| Plan MVP 1 incompatible avec le moteur 3D | Moyen | Élevé | Schéma `Plan` strict, validation à l'import |
| Clé LLM exposée côté client | Élevé | Critique | Proxy backend minimal en prod |
| Occlusion culling mal réglé | Moyen | Moyen | Hystérésis + conservateur |
| Photoréalisme = fps bas sur GPU intégrés | Élevé | Moyen | Toggle qualité + downgrade auto |
| Scope creep | Élevé | Élevé | Ce doc fige le périmètre |

---

## 10. Ce qui est explicitement EXCLU du MVP 2

Multi-étages avancés (hors gestion simple du MVP 1) · Ray tracing · Acoustique · Thermique · Style transfer · Budget optimizer · Feng Shui · Scan réel · IKEA · Marketplace · Social · Collaboration · VR/AR · Import PDF/DWG · Simulation habitants · E-commerce · WebGPU (V1.1) · Portails de pièces (V1.1).

> Règle : si une feature n'est pas dans la boucle §0, elle n'entre pas.

---

## 11. Prochaine étape

1. Valider ce plan et sa compatibilité avec `docs/MVP1.md`
2. Sprint 0 → squelette compilable avec render-on-demand
3. Attaquer Sprint 1 (import et édition 2D)
