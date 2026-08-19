# Habiter — `archimed_ai`

> Imagine. Crée. Visite. Vis.

Une application desktop **performance-first** pour décrire une maison → obtenir un plan technique 2D généré par IA → affiner → obtenir une maison 3D photoréaliste visitable → la meubler → s'y promener.

## Stack (édition performance-first)

| Couche | Techno |
|--------|--------|
| Shell desktop | **Tauri 2** (Rust) — binaire natif, accès système, offline par défaut |
| UI shell | **SolidJS** (signals, ~7-20 Ko, pas de VDOM) |
| Moteur 3D | **vanilla three.js** (WebGL2, r17x) — contrôle total perf |
| Backend local | **Rust** (Tauri commands) — stockage, proxy IA, Asset Store |
| Persistence | **SQLite** local + système de fichiers |
| Asset Store | Manifestes JSON + cache local — pack initial + communautaire |
| Perf 3D | **three-mesh-bvh** (occlusion culling), InstancedMesh, BatchedMesh |
| Build | Tauri CLI + Vite + TypeScript (strict) + tree-shaking |
| State | solid-js stores + signals |

> **Pas de React / R3F / Threlte.** La 3D est pilotée en impératif direct (contrôle maximal de la perf) ; SolidJS ne gère que l'UI mince. Voir `docs/PERFORMANCE.md` pour la justification.

## Philosophie performance

> **N'afficher que ce qui est dans le champ de la caméra** — et qui n'est pas caché par un mur.
> **Ne télécharger que ce qui est utilisé.** **Ne rendre que quand ça bouge.** **Adapter la charge au matériel et à la connexion.**

Cinq piliers : **Culling · Batching · Render-on-demand · Assets compressés · Qualité adaptative.**

Objectifs : **> 30 FPS sur GPU intégré**, **> 60 FPS sur GPU dédié**, **GPU idle en édition**, **100 % offline après téléchargement initial des assets**, **premier rendu 3D < 3 s une fois les assets locaux installés**.

## Démarrage rapide

```bash
pnpm install
pnpm tauri dev
```

> Nécessite [Rust](https://rustup.rs/) et le [Tauri CLI](https://tauri.app/start/prerequisites/) installés.

Au premier lancement, l'application propose de télécharger le pack d'assets initiaux (~1,5 Mo). Une fois installé, tout fonctionne hors-ligne.

## Documentation

### MVP

- [`docs/MVP1.md`](./docs/MVP1.md) — **MVP 1** : générateur de plan technique 2D par IA (terrain, forme, étages, pièces, normes)
- [`docs/MVP2.md`](./docs/MVP2.md) — **MVP 2** : de l'éditeur 2D à la visite 3D (affinage, 3D, meubles, FPS)
- [`docs/STANDARDS.md`](./docs/STANDARDS.md) — Référentiel des normes techniques françaises encodées pour le MVP 1

### Référence technique

- [`docs/OBJECTS.md`](./docs/OBJECTS.md) — ⭐ **Modèle Référent / Instance des objets** (AssetDefinition, GLB pivot, LODs, IA/import/export)
- [`docs/PERFORMANCE.md`](./docs/PERFORMANCE.md) — Spécifications perf GPU/render (culling, batching, budgets, mesure)
- [`docs/LOWEND.md`](./docs/LOWEND.md) — Spécifications machines faibles, first-run et stockage local (RAM/GC, dynamic resolution, profil auto)
- [`docs/ROADMAP.md`](./docs/ROADMAP.md) — Vision complète post-MVP (V1.1 → V5)

## La boucle produit

```
MVP 1 : Décrire la maison  →  IA génère un plan technique 2D  →  Plan éditable
          (terrain, forme,        (murs, portes, fenêtres,         (SVG multi-niveaux
           étages, pièces)          pièces, poteaux, normes)          modifiable)

                                    ↓

MVP 2 : Affiner le plan  →  IA structure  →  Murs 3D  →  Placer objets  →  Visiter (FPS)
          (éditeur 2D)      (noms pièces)    (batched)   (instanced)       (culling BVH)
```
