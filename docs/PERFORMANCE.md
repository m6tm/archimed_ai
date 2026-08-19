# Performance — Spécifications techniques approfondies

> Statut : **Référence**. Compagnon de `MVP1.md` et `MVP2.md`. Détaille le *comment* des optimisations.
> Dernière mise à jour : 2026-08-19

## Périmètre

- **MVP 1** (générateur de plan 2D IA) : interface principalement SVG + appels au LLM via Tauri. Les optimisations 3D ne s'appliquent pas.
- **MVP 2** (éditeur 2D/3D/visite) : c'est ici que les optimisations GPU/render deviennent critiques.

Ce document se concentre sur le MVP 2. Les budgets spécifiques au MVP 1 figurent en §0.1.

## Philosophie

> **N'afficher que ce qui est dans le champ de la caméra** — et qui n'est pas caché par un mur.

Une maison en intérieur est un cas paradisiaque pour le culling : la plupart des pièces sont cachées derrière des murs. Une approche naïve (tout rendre) gaspille 60-90% du travail GPU. Notre stratégie en cinq piliers :

```
1. CULLING      — ne rendre que le visible (frustum + occlusion)
2. BATCHING     — regrouper les draw calls (InstancedMesh, BatchedMesh)
3. ON-DEMAND    — ne rendre que quand ça bouge (render loop pilotée)
4. ASSETS       — compresser et budgeter (meshopt, KTX2, < 80Ko)
5. ADAPTIVE     — se dégrader gracieusement (qualité auto)
```

---

## 0.1 Performance du MVP 1

Le MVP 1 est une interface desktop 2D (SVG) avec des appels au LLM via le backend Tauri. Les contraintes GPU du MVP 2 ne s'appliquent pas.

| Métrique | Cible MVP 1 |
|----------|-------------|
| Bundle JS gz (initial) | < 120 Ko |
| Premier affichage formulaire | < 1 s après lancement |
| Temps de génération IA | < 15 s (LLM + validation) |
| Temps de rendu SVG | < 500 ms après génération |
| Mémoire JS | < 80 Mo |
| Appels API LLM | 1 à 3 par génération (prompt + retry) |

> Le MVP 1 doit rester fluide sur desktop bas de gamme, sans WebGL.

---

## 1. Culling — le pilier n°1

### 1.1 Frustum culling (niveau objet)

Activé par défaut dans three.js (`mesh.frustumCulled = true`). Tout objet dont la bounding box est hors du cône de vision de la caméra est **skip** du rendu.

```ts
// Automatique, mais vérifier qu'aucun mesh n'a frustumCulled=false par erreur
scene.traverse(o => { if (o.isMesh) o.frustumCulled = true; });
```

⚠️ Pour InstancedMesh/BatchedMesh, fournir une **bounding box englobante correcte** sinon le culling râte.

### 1.2 Occlusion culling — la technique clé (three-mesh-bvh)

Dans un intérieur, le frustum culling ne suffit pas : une armoire dans la pièce d'à-côté est *dans le frustum* mais cachée par un mur. L'**occlusion culling** l'élimine.

**Principe :**

```
Frame N:
  1. Render un depth-pass LOW-RES (ex: 256×144) des OCCLUDEURS (murs)
  2. Pour chaque objet candidat (meubles, murs lointains):
       - Projeter sa bounding box dans l'espace écran
       - Tester ses samples contre le depth buffer
       - Si entièrement occludé → CULL (skip draw)
  3. Render la scène complète (objets visibles seulement)
```

**Bibliothèque :** [`three-mesh-bvh`](https://github.com/gkjohnson/three-mesh-bvh) de gkjohnson — mature, utilisée en production, démo BatchedMesh+BH à 1M instances fluide sur mobile ([forum](https://discourse.threejs.org/t/batchedmesh-1-milion-instances-using-a-bvh/82028)).

**Implémentation squelette :**

```ts
import { MeshBVH, MeshBVHHelper, computeBatchedBoundsTree } from 'three-mesh-bvh';

// 1. BVH sur les murs (occluders statiques)
const wallBvh = new MeshBVH(wallMergedGeometry);
wallsMesh.geometry.boundsTree = wallBvh;

// 2. Occlusion culling pass (chaque frame en mode visite)
function occlusionCull(camera, candidates: Mesh[]) {
  // Pour chaque candidat: raycast BVH depuis caméra vers ses coins d'AABB
  // Si tous les rayons sont bloqués avant d'atteindre l'objet → occludé
  for (const obj of candidates) {
    obj.visible = !isOccludedByBVH(obj, camera, wallsBvh);
  }
}
```

**Hystérésis (anti-scintillement) :** un objet qui devient caché reste visible 2-3 frames avant d'être cull. Évite le clignotement aux bordures de murs.

**Gain attendu :** dans un appartement T3, **60-80% des triangles sont éliminés** dès qu'on traverse une porte.

### 1.3 Portails de pièces (V1.1, hors MVP)

Évolution : découper la maison en **cellules** (pièces) connectées par des **portails** (portes). On ne rend que la cellule courante + celles visibles au travers des portails ouverts. Technique des moteurs de jeu (Source/Unreal). Plus précis que l'occlusion BVH pour les grands plans. Programmé en V1.1.

---

## 2. Batching — réduire les draw calls

Un T3 meublé naïf = **200-400 draw calls** → goulot CPU (chaque draw = overhead).

### 2.1 InstancedMesh (objets répétés)

Tout meuble répété N fois devient **1 draw call** :

```ts
// Ex: 4 chaises identiques autour d'une table
const chairInstanced = new THREE.InstancedMesh(chairGeo, chairMat, 4);
chairInstanced.setMatrixAt(i, matrix);   // position/rotation/scale par instance
```

**Catalogue MVP concerné :** chaises (×4), chevets (×2), placards, plantes, spots lumineux.

### 2.2 BatchedMesh (three r16x+, statique)

Les murs d'un étage, tous de même matériau → **1 BatchedMesh = 1 draw call** :

```ts
const wallBatch = new THREE.BatchedMesh(WALL_COUNT, vertexCount, indexCount, wallMaterial);
const geomId = wallBatch.addGeometry(wallGeometry);
const instanceId = wallBatch.addInstance(geomId, matrix);
```

Compatible avec le frustum culling et BVH ([BatchedMesh BVH](https://discourse.threejs.org/t/batchedmesh-1-milion-instances-using-a-bvh/82028)).

### 2.3 Règle de budget

| Type | Draw calls max |
|------|----------------|
| Murs (par étage) | 1 (BatchedMesh) |
| Sol / plafond | 1 chacun |
| Meubles uniques | 1 / objet |
| Meubles instanciés | 1 / type |
| **Total cible (MVP)** | **< 50** (Performant), **< 120** (Photoréaliste) |

---

## 3. Render-on-demand

90% du temps en édition, la scène ne change pas. Rendre 60fps pour une image fixe = gaspillage pur.

### 3.1 Loop pilotée

```ts
class Engine {
  private needsRender = false;
  private animating = false;

  // Appelé par TOUTE mutation UI
  invalidate() { this.needsRender = true; }

  loop = () => {
    requestAnimationFrame(this.loop);
    if (!this.needsRender && !this.animating) return;
    this.needsRender = false;
    this.preRender();        // culling update, contrôles
    this.renderer.render(this.scene, this.camera);
  };

  setMode(mode: 'edit' | 'visit') {
    this.animating = (mode === 'visit');
  }
}
```

### 3.2 Déclencheurs d'invalidation

| Événement | Invalide ? |
|-----------|------------|
| Ajout/suppression/modif mur | ✅ |
| Placement meuble | ✅ |
| Drag meuble (en cours) | ✅ chaque move |
| Changement qualité | ✅ |
| Mode visite (FPS) | ✅ continu (`animating=true`) |
| Slider heure du jour | ✅ |
| Idle (rien ne bouge) | ❌ → **GPU à 0%** |

**Gain :** en édition, le laptop ne chauffe plus, batterie préservée.

---

## 4. Assets & mémoire GPU

### 4.1 Budgets

| Asset | Budget |
|-------|--------|
| Meuble (geo + textures) | **< 80 Ko** |
| Texture mur / sol | 1024×1024 max |
| Texture meuble | 512×512 max |
| Total textures GPU | < 50 Mo |
| HDRI (.env PMREM) | 256 cubemap |

### 4.2 Compression

| Asset | Format | Pourquoi |
|-------|--------|----------|
| Géométrie GLB | **meshopt** (via `gltfpack`) | Meilleur ratio + decode plus rapide que DRACO, support hardware |
| Textures | **KTX2/Basis** | Transcodé GPU-native, ~70% plus petit que PNG/JPG, mipmaps inclus |
| HDRI | `.env` 256 PMREM pré-calculé | Pas de filtrage runtime |

**Pipeline build assets :**

```bash
# Géométrie
gltfpack -i sofa.glb -o sofa.glb \
  -cc -si 0.5        # meshopt, simplifier 50%

# Textures
toktx --bcmp sofa_basecolor.png    # BC compression GPU
```

### 4.3 Chargement progressif

Les meubles lointains (via occlusion culling) peuvent être chargés en lazy : on ne charge le GLB depuis le disque local que quand l'objet entre dans le frustum pour la première fois. (Optimisation V1.1 — MVP charge tout au démarrage, budget total < 2 Mo.)

---

## 5. Ombres (poste de coût n°1)

| Réglage | Photoréaliste | Performant |
|---------|---------------|------------|
| Actif ? | ✅ | ❌ |
| Type | PCFSoft | — |
| Map size | 1024 | — |
| Casters | Murs + soleil | — |
| Receivers | Sol | — |

En « Performant », **supprimer les ombres = +40-60% FPS** sur GPU intégrés. Compensation : AmbientLight + HemisphereLight (look correct sans coût).

---

## 6. Post-processing conditionnel

```ts
const composer = new EffectComposer(renderer);
if (quality === 'high') {
  composer.addPass(new SSAOEffect(...));   // demi-résolution
  composer.addPass(new BloomEffect({ luminanceThreshold: 0.8 }));  // subtil
  composer.addPass(new VignetteEffect());
}
```

- `gl={{ toneMapping: NoToneMapping }}` **obligatoire** avec `postprocessing` ([pmndrs/postprocessing](https://github.com/pmndrs/postprocessing)).
- DoF exclu du MVP (trop coûteux, peu de valeur en visite FPS).

---

## 7. Qualité adaptative

### 7.1 Détection GPU au démarrage

```ts
function estimateGpuTier(): 'low' | 'high' {
  const dbg = renderer.getContext().getExtension('WEBGL_debug_renderer_info');
  const name = dbg ? renderer.getContext().getParameter(dbg.UNMASKED_RENDERER_WEBGL) : '';
  // Heuristique : "Intel" / "AMD Radeon(TM) Graphics" (intégrés) → low
  // "RTX" / "Radeon RX" / "Apple M" (dédiés) → high
  return /Intel|Intel|HD Graphics|UHD|Radeon\(TM\)|Adreno 6|Adreno 7[0-4]/i.test(name) ? 'low' : 'high';
}
```

### 7.2 Downgrade auto runtime

Fenêtre glissante de 60 frames :

```ts
if (avgFps < 30 && quality === 'high') {
  setQuality('low');          // désactive ombres + post-FX
  showToast('Qualité ajustée pour la fluidité');
}
```

Le produit **ne lag jamais** — il se dégrade plutôt que de ramer.

---

## 7bis. Côté UI (SolidJS) — garde l'UI maigre

L'UI reste volontairement **mince** : la 3D est la star, l'UI ne fait qu'éditer un store. Règles :

- Pas de librairie de composants lourde (pas de MUI/AntD). Composants maison + CSS modules, ~5 Ko.
- Pas de router client (un seul écran), `@solidjs/router` non requis au MVP.
- Signaux Solid = réactivité fine-grained, **pas de re-render global** : déplacer un meuble ne re-render pas le catalogue.
- Bundle UI cible : **< 30 Ko gz** (hors three.js).

---

## 8. Mesure (obligatoire)

HUD dev (toggle F3) :

| Métrique | Source |
|----------|--------|
| FPS (moyenne 60f) | `requestAnimationFrame` timestamps |
| Draw calls | `renderer.info.render.calls` |
| Triangles | `renderer.info.render.triangles` |
| Textures | `renderer.info.memory.textures` |
| Geometries | `renderer.info.memory.geometries` |
| Temps GPU/frame | `EXT_disjoint_timer_query` (si dispo) |
| **Heap JS** | `performance.memory.usedJSHeapSize` (Chrome) |
| **Jank** (frames > 16ms) | PerformanceObserver `longtask` |

Sans mesure, pas d'opti. On compare avant/après chaque optimisation.

---

## 9. Synthèse des budgets

| Métrique | Ultra-low | Performant | Photoréaliste |
|----------|-----------|------------|---------------|
| FPS (visite) | > 30 (GPU intégré ancien) | > 30 (GPU intégré) | > 60 (GPU dédié) |
| Draw calls | < 30 | < 50 | < 120 |
| Triangles rendus (post-culling) | < 40k | < 80k | < 300k |
| Résolution render | **50 % + FXAA** | 75 % | 100 % |
| Ombres | Off | Off | PCFSoft 1024 |
| Post-FX | Off | Off | SSAO+Bloom+Vignette |
| Bundle JS gz (hors three) | < 60 Ko | < 60 Ko | < 100 Ko |
| Textures GPU | < 15 Mo | < 30 Mo | < 50 Mo |
| RAM WASM/JS | < 100 Mo | < 120 Mo | < 200 Mo |
| GPU idle en édition | ~0% | ~0% | ~0% |
| **Premier rendu 3D (assets locaux)** | **< 3 s** | < 3 s | < 3 s |
| **Jank > 16 ms** | **0** | 0 | 0 |
| **Taille binaire installateur** | < 20 Mo | < 20 Mo | < 20 Mo |
| **Pack assets initiaux** | ~1,5 Mo | ~1,5 Mo | ~1,5 Mo |

> **Niveau « Ultra-low »** ajouté pour machines faibles (GPU intégré ancien, peu de RAM, disque lent). Activé automatiquement par le profil machine (voir LOWEND.md §6).

---

## 10. Roadmap des optimisations (post-MVP)

| Version | Opti | Gain attendu |
|---------|------|--------------|
| V1.1 | **Portails de pièces** (cellules + portails) | Élimine pièces entières, précision > BVH |
| V1.1 | **LOD** sur meubles (LOD3/LOD1 selon distance) | -50% triangles lointains |
| V1.1 | **WebGPU** optionnel (compute shaders soft shadows) | Ombres douces GPU |
| V1.1 | Lazy-loading des GLB lointains | Premier chargement < 1s |
| V1.2 | **Worker** pour le culling per-frame (off-main-thread) | Décharge le main thread |
| V1.2 | Variants de tuiles de sol (texture bombing) | Tiling invisible |
| V1.2 | **Impostors/billboards** pour objets très lointains | 1 quad au lieu de milliers de triangles |
| V1.3 | **Atlas de textures** partagé (1 bind pour tous meubles) | -draw calls |
| V1.3 | Streaming de scènes (gros projets multi-étages) | Scale |

> **Techniques low-end au MVP** (détails dans `LOWEND.md`) : code-splitting chunks, stockage local, Asset Store, GLB paresseux depuis le disque, object pooling, BVH build en worker, dynamic resolution scaling, FXAA, distance culling, progressive enhancement, profil machine auto.

---

## Références

- [three-mesh-bvh (gkjohnson)](https://github.com/gkjohnson/three-mesh-bvh) — occlusion culling, raycast accéléré
- [BatchedMesh + BVH, 1M instances sur mobile](https://discourse.threejs.org/t/batchedmesh-1-milion-instances-using-a-bvh/82028)
- [pmndrs/postprocessing](https://github.com/pmndrs/postprocessing) — EffectComposer (SSAO/Bloom/DoF)
- [The Complete Guide to Three.js Post-Processing 2026](https://threejsroadmap.com/blog/the-complete-guide-to-threejs-post-processing-in-2026)
- [WebGPU regression r182 (discourse)](https://discourse.threejs.org/t/webgpu-significant-performance-drop-and-shadow-quality-regression-in-r182-vs-webgl-r170/89322)
- [WebGPU UBO bottleneck (GitHub #30560)](https://github.com/mrdoob/three.js/issues/30560)
- [The State of Solid.js 2026](https://listiak.dev/blog/the-state-of-solid-js-in-2026-signals-performance-and-growing-influence) — perf Solid, bundle ~7-20 Ko
- [React Three Fiber vs Three.js 2026](https://www.creativedevjobs.com/blog/react-three-fiber-vs-threejs) — bundle three.js ~155 Ko gz
- [Collision BVH (Medium)](https://medium.com/@pablobandinopla/collision-detection-in-threejs-made-easy-using-bvh-1ce6012199e8)
