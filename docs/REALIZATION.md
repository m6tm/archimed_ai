# Plan de réalisation — MVP 1 + MVP 2

> Statut : **Plan d'exécution opérationnel**.  
> Décrit l'ordre de réalisation des deux MVP : MVP 1 (plan 2D IA) puis MVP 2 (éditeur 2D/3D/visite).

---

## Comment lire ce document

Le plan est découpé en **phases numérotées** qui s'enchaînent strictement. Chaque phase a un **livrable démontrable** et des **critères de sortie**. On ne passe à la phase suivante que si les critères sont remplis.

Références croisées :
- `docs/MVP1.md` — spécification du générateur de plan 2D IA
- `docs/MVP2.md` — spécification de l'éditeur 2D/3D/visite
- `docs/STANDARDS.md` — normes techniques encodées
- `docs/OBJECTS.md` — modèle Référent/Instance
- `docs/PERFORMANCE.md` — specs performance
- `docs/LOWEND.md` — machines faibles, first-run et stockage local
- `docs/ROADMAP.md` — vision post-MVP

---

## Phase 0 — Cadrage & pré-développement

### 0.1 Validation du périmètre
- [ ] Relire et figer `MVP1.md`, `MVP2.md` et `STANDARDS.md`
- [ ] Confirmer le nom produit (« Habiter » par défaut)
- [ ] Valider les cibles perf (`PERFORMANCE.md` §9, `LOWEND.md` §6)
- [ ] Tracer une frontière nette MVP 1 / MVP 2 / V1.1

### 0.2 Environnement de travail
- [ ] Installer Node 22 LTS + pnpm 9+
- [ ] Installer [Rust](https://rustup.rs/) + [Tauri CLI](https://tauri.app/start/prerequisites/)
- [ ] Vérifier WebView2 sur Windows (Tauri le détecte / installe)
- [ ] Compte OpenAI **ou** Anthropic + clé API (dev local)
- [ ] Serveur d'assets pour les GLB du MVP 2 (ou stockage local en dev)
- [ ] (Optionnel) Compte code-signing pour le build Windows

### 0.3 Repos & conventions
- [ ] `git init` + `.gitignore`
- [ ] `.editorconfig`, `.nvmrc` (Node 22), `.gitattributes` (LF forcé)
- [ ] `LICENSE` (MIT ou propriétaire)
- [ ] Convention de commits (Conventional Commits)
- [ ] Branche principale protégée + branche de travail par phase

### 0.4 Design system minimal
- [ ] Palette + typographie (CSS variables, ~5 tokens)
- [ ] Grille & espacements (échelle 4/8px)
- [ ] Icônes (Lucide ou Heroicons, 1 set)
- [ ] Maquettes low-fi des parcours MVP 1 et MVP 2

### 0.5 Critères de sortie Phase 0
- ✅ Périmètre figé, aucun débordement accepté
- ✅ Environnement de dev opérationnel
- ✅ Design tokens définis

---

# Partie A — MVP 1 : Générateur de plan technique 2D

## Phase 1 — Interface de saisie

> **Livrable :** un formulaire guidé permettant de décrire le terrain, la forme, les étages, le programme et les contraintes de style.

### 1.1 Terrain
- [ ] `TerrainForm` : largeur, profondeur, orientation, reculs
- [ ] Visualisation simplifiée du terrain (rectangle orienté)

### 1.2 Forme de la maison
- [ ] `ShapeDrawer` : zone SVG pour dessin à la main levée
- [ ] Simplification du tracé (polygone fermé)
- [ ] `ImageUploader` : import de photo/scan
- [ ] Extraction de forme depuis l'image (modèle de vision ou traitement local)
- [ ] Aperçu du polygone simplifié et correction manuelle

### 1.3 Étages
- [ ] `LevelConfigurator` : nombre d'étages
- [ ] Par défaut : chaque niveau reprend la forme de référence
- [ ] Possibilité de modifier la forme par niveau

### 1.4 Programme des pièces
- [ ] `RoomProgramBuilder` : saisie en langage naturel par étage
- [ ] Parsing en `RoomSpec[]` (type, surface cible, contraintes attachées)

### 1.5 Style et contraintes
- [ ] `StyleConstraintsInput` : champ libre + suggestions
- [ ] Extraction de mots-clés structurés

### 1.6 State & persistance
- [ ] `briefStore` (Solid store) : terrain, envelope, levels, program, constraints
- [ ] Persistance SQLite via Tauri auto (debounced)
- [ ] Migration de schéma

### 1.7 Critères de sortie Phase 1
- ✅ Un utilisateur peut décrire une maison complète en moins de 5 minutes
- ✅ Le dessin à la main levée et l'import d'image fonctionnent
- ✅ Les données sont persistées et rechargées

---

## Phase 2 — Pipeline IA et validation

> **Livrable :** le brief utilisateur est transformé en plan technique structuré, validé par un validateur déterministe.

### 2.1 Normalisation des entrées
- [ ] Conversion du terrain en polygone orienté
- [ ] Normalisation de la forme dessinée/importée
- [ ] Parsing du programme pièces en `RoomSpec[]`
- [ ] Extraction des contraintes de style

### 2.2 LLM client
- [ ] `src-tauri/src/commands/llm.rs` : abstraction OpenAI ↔ Anthropic côté Rust
- [ ] Prompt versionné incluant les règles de `STANDARDS.md`
- [ ] Schéma Zod strict de sortie validé côté backend
- [ ] Frontend : `invoke('generate_plan', { brief, provider })`

### 2.3 Génération du plan
- [ ] `generatePlan(brief)` : appel LLM avec prompt structuré
- [ ] Sortie : `LevelPlan[]` (murs, ouvertures, pièces, poteaux)

### 2.4 Validation technique
- [ ] Validateur déterministe : fermeture, non-chevauchement, surfaces, lumière, ventilation, circulations, structure
- [ ] Génération d'alertes classées par sévérité
- [ ] Retry automatique sur erreur avec feedback au LLM
- [ ] Fallback gracieux si le LLM échoue

### 2.5 Critères de sortie Phase 2
- ✅ Le LLM produit un plan structuré et validé
- ✅ Les alertes normatives sont détectées et affichables
- ✅ Le fallback fonctionne en cas d'erreur

---

## Phase 3 — Rendu SVG éditable multi-niveaux

> **Livrable :** le plan généré est affiché en SVG, éditable, avec les niveaux côte à côte.

### 3.1 Rendu SVG
- [ ] `PlanRenderer` : murs, ouvertures, pièces, poteaux, dimensions
- [ ] Chaque élément identifiable par `data-id`
- [ ] Zoom / pan sur le plan

### 3.2 Affichage multi-niveaux
- [ ] Vue côte à côte des niveaux
- [ ] Échelle commune
- [ ] Alignement des éléments structurels

### 3.3 Édition
- [ ] Sélection d'élément
- [ ] Déplacer un mur, une ouverture, une pièce
- [ ] Redimensionner
- [ ] Ajouter/supprimer une ouverture
- [ ] Undo/redo

### 3.4 Validation en temps réel
- [ ] Recalcul des alertes lors d'une modification
- [ ] Affichage visuel des alertes sur le plan

### 3.5 Export vers MVP 2
- [ ] Conversion du plan édité au format `Plan` (compatible MVP 2)
- [ ] Bouton **« Passer en 3D »**

### 3.6 Critères de sortie Phase 3
- ✅ Le plan s'affiche correctement pour chaque niveau
- ✅ Les niveaux sont côte à côte et alignés
- ✅ L'utilisateur peut modifier le plan
- ✅ L'export vers le MVP 2 fonctionne

---

# Partie B — MVP 2 : Éditeur 2D/3D/visite

## Phase 4 — Squelette technique Tauri

> **Livrable :** une fenêtre desktop qui affiche un cube three.js, ne rend que quand il bouge, avec un HUD fps.

- [ ] pnpm workspace + `apps/desktop` (Vite + SolidJS + TypeScript strict) + `src-tauri` (Rust)
- [ ] Installer `three`, `three-mesh-bvh`, `postprocessing`, `meshoptimizer`
- [ ] ESLint + Prettier + `tsconfig` strict
- [ ] Configurer `tauri.conf.json` (permissions `fs`, `sql`, `http`, `dialog`)
- [ ] Code-splitting : `engine` chunk séparé de `core`
- [ ] `Engine.ts` : WebGLRenderer + render-on-demand + HUD fps
- [ ] Object pooling de base
- [ ] `main.ts` : monteur Solid + chargement paresseux du moteur
- [ ] Test : le cube ne rend que sur `engine.invalidate()`, GPU idle entre

### 4.1 Critères de sortie Phase 4
- ✅ `pnpm tauri dev` affiche un cube
- ✅ HUD fps visible
- ✅ Profiler : GPU ~0% quand statique
- ✅ Bundle `core` < 120 Ko gz (sans three)

---

## Phase 5 — Import et affinage 2D

> **Livrable :** le plan du MVP 1 est chargé et peut être affiné dans l'éditeur SVG.

### 5.1 Import de plan
- [ ] Chargement d'un plan au format `Plan` (MVP 1)
- [ ] Détection du nombre de niveaux
- [ ] Gestion du mode "dessin manuel" si aucun plan importé

### 5.2 Éditeur 2D
- [ ] `PlanEditor` SVG + grille snap 20 cm
- [ ] `WallTool`, `DoorTool`, `WindowTool`, `DimensionLabel`
- [ ] Pan/zoom du plan
- [ ] Undo/redo

### 5.3 State & persistance
- [ ] `planStore` (Solid store)
- [ ] Persistance SQLite via Tauri auto
- [ ] Migration de schéma

### 5.4 Fondations geometry
- [ ] `lib/geometry/types.ts`
- [ ] Tests `detectRooms`

### 5.5 Critères de sortie Phase 5
- ✅ Un plan MVP 1 se charge et s'affiche
- ✅ L'utilisateur peut l'affiner
- ✅ Recharger la page → plan intact
- ✅ Tests `detectRooms` verts

---

## Phase 6 — 3D, extrusion & culling

> **Livrable :** le plan devient une maison 3D, le culling BVH est visible au HUD.

### 6.1 Builders
- [ ] `WallBuilder` : extrusion 2D→3D → BatchedMesh
- [ ] `FloorBuilder`
- [ ] `OpeningBuilder` (mesh segmenté, pas de CSG)
- [ ] `CeilingBuilder`

### 6.2 Moteur
- [ ] `World.ts` + reconcile (diff par id)
- [ ] `BvhWorld` : construction BVH des murs
- [ ] `OcclusionCuller`
- [ ] Hystérésis anti-scintillement
- [ ] BVH build en Worker

### 6.3 Rendu & lumières
- [ ] `Lighting` : HDRI intérieur
- [ ] `OrbitController`
- [ ] Matériaux PBR, textures KTX2
- [ ] Bouton **« Générer 3D »**

### 6.4 Critères de sortie Phase 6
- ✅ Plan 2D → maison 3D cohérente
- ✅ **Fidélité géométrique** : vue de dessus de la maison 3D superposable au plan 2D du MVP 1 (mêmes emprises, mêmes ouvertures)
- ✅ HUD : triangles rendus après culling << avant
- ✅ Pas de scintillement en orbite
- ✅ GPU idle en mode inspection statique

---

## Phase 7 — IA structure

> **Livrable :** les pièces détectées sont nommées/classées par le LLM.

- [ ] `detectRooms()` : polygones fermés + aire + adjacences
- [ ] `llm/client.ts`, `enrichWithLLM` + Zod + fallback
- [ ] `StructurePanel` : pièces nommées éditables
- [ ] Bouton **« Analyser »**

### 7.1 Critères de sortie Phase 7
- ✅ Sur un T2 : pièces nommées « Salon », « Cuisine », « Chambre »…
- ✅ Fallback actif si le LLM échoue

---

## Phase 8 — Asset Store + catalogue + placement

> **Livrable :** on meuble le plan en glissant-déposant des objets, avec collision.

### 8.1 Modèle Référent/Instance
- [ ] `lib/assets/types.ts`
- [ ] `AssetRegistry`
- [ ] `packages/assets-registry/builtin-catalog.json` : ~25 référents

### 8.2 Asset Store backend
- [ ] `src-tauri/src/storage/asset_store.rs` : manifeste, téléchargement, cache local
- [ ] `src-tauri/src/commands/assets.rs` : `install_asset`, `list_installed_assets`, `get_asset_path`
- [ ] Écran first-run : téléchargement du pack initial (~1,5 Mo)
- [ ] Panneau Asset Store : assets communautaires (browse / install / update)

### 8.3 Asset pipeline
- [ ] Sourcer ~25 GLB libres
- [ ] Vérifier les licences
- [ ] `gltfpack` : compression meshopt
- [ ] `AssetNormalizer`
- [ ] Thumbnails WebP

### 8.4 Placement
- [ ] `CatalogPanel` drag&drop
- [ ] `FurnitureBuilder` : chargement paresseux depuis le disque local, clone (partage VRAM)
- [ ] InstancedMesh pour objets répétés
- [ ] TransformControls move/rotate
- [ ] Collision AABB

### 8.5 Critères de sortie Phase 8
- ✅ On glisse 6 meubles, on les positionne/rotationne
- ✅ 4 chaises = 1 draw call
- ✅ Pas de recollision GPU au drag
- ✅ Catalogue filtre par catégorie
- ✅ Les assets sont chargés depuis le cache local

---

## Phase 9 — Visite, photoréalisme, profil auto

> **Livrable :** la boucle complète fonctionne, fluide même sur machine faible.

### 9.1 Visite FPS
- [ ] `WalkController` : PointerLock + ZQSD + collision BVH
- [ ] Hauteur d'yeux 1,6 m
- [ ] Boutons **« Visiter »** / **« Quitter visite »**

### 9.2 Photoréalisme conditionnel
- [ ] `PostFX` : SSAO + Bloom + Vignette (haute only)
- [ ] FXAA
- [ ] Dynamic Resolution Scaling
- [ ] Ombres PCFSoft 1024 (haute only)

### 9.3 Profil machine & qualité adaptative
- [ ] `detectProfile()`
- [ ] Preset auto appliqué au démarrage
- [ ] Downgrade auto runtime
- [ ] `QualityToggle` manuel

### 9.4 Détection offline
- [ ] Commande Rust `is_online()` pour vérifier la connectivité
- [ ] Message clair si IA indisponible hors-ligne
- [ ] Asset Store communautaire accessible uniquement en ligne

### 9.5 Soleil dynamique
- [ ] Slider heure du jour
- [ ] HDRI jour/nuit

### 9.6 Critères de sortie Phase 9
- ✅ Visite FPS > 30 sur GPU intégré, > 60 sur GPU dédié
- ✅ Aucun jank > 16 ms en 5 min
- ✅ Downgrade auto se déclenche sous charge

---

## Phase 10 — Robustesse, low-end & finition

> **Livrable :** l'app tient sur machines faibles et gère les erreurs gracieusement.

### 10.1 Asset Store & offline
- [ ] Écran first-run : téléchargement du pack initial robuste (reprise, annulation)
- [ ] Cache local des assets dans le dossier utilisateur
- [ ] 2e ouverture offline → app fonctionne
- [ ] Mode limité si assets non installés
- [ ] Gestion des mises à jour d'assets

### 10.2 Gestion d'erreur & états
- [ ] Error boundary Solid
- [ ] États vides, états de chargement, skeleton screens
- [ ] Gestion échec LLM et échec chargement GLB

### 10.3 Mémoire & GC
- [ ] Checklist `dispose()` systématique
- [ ] Test : reconstruire la scène 20× → heap stable
- [ ] Strip `console.log` en prod

### 10.4 Progressive enhancement
- [ ] Premier rendu 3D progressif
- [ ] Préchargement idle du chunk engine
- [ ] Splash screen / first-run UI léger

### 10.5 Onboarding minimal
- [ ] Bulles de découverte au 1er lancement
- [ ] Skip + ne pas réafficher

### 10.6 Critères de sortie Phase 10
- ✅ Validation `LOWEND.md` §7 complète
- ✅ App offline fonctionne après téléchargement initial des assets
- ✅ Heap stable après 20 rebuilds
- ✅ Aucune erreur non gérée à l'écran

---

## Phase 11 — Tests, validation & livraison

> **Livrable :** MVP 1 et MVP 2 déployés, testés, documentés, prêts pour bêta-testeurs.

### 11.1 Tests automatisés
- [ ] Tests unitaires (Vitest) : parsing, geometry, LLM Zod schema
- [ ] Tests composants : formulaires, plan editor, catalogue
- [ ] Tests e2e légers (Playwright optionnel)
- [ ] **Test de fidélité MVP 1 → MVP 2** : comparaison géométrique plan 2D / vue de dessus 3D

### 11.2 Validation performance
- [ ] Premier écran < 1 s après lancement
- [ ] First-run < 30 s sur connexion modeste
- [ ] GPU intégré → > 30 FPS
- [ ] GPU dédié → > 60 FPS
- [ ] Bundle check
- [ ] Taille binaire installateur < 20 Mo

### 11.3 Validation low-end
- [ ] Assets locaux présents → premier rendu 3D < 3 s
- [ ] `deviceMemory = 4` simulé
- [ ] First-run sur connexion modeste < 30 s
- [ ] Offline après téléchargement initial des assets

### 11.4 Compatibilité desktop
- [ ] Windows 10+ avec WebView2
- [ ] macOS 12+ (Apple Silicon / Intel)
- [ ] Linux (WebKitGTK)
- [ ] Détection WebGL2

### 11.5 Sécurité
- [ ] Clé LLM stockée dans les préférences locales chiffrées (jamais dans le bundle)
- [ ] Appels LLM via le backend Rust Tauri
- [ ] Rate-limit côté backend
- [ ] Pas de secrets dans le bundle

### 11.6 Métadonnées app
- [ ] Nom, icône, version dans `tauri.conf.json`
- [ ] Favicon

### 11.7 Documentation livrée
- [ ] README utilisateur
- [ ] README développeur
- [ ] `CHANGELOG.md`
- [ ] Docs `docs/` à jour et cohérentes

### 11.8 Déploiement
- [ ] Build prod desktop (`pnpm tauri build`)
- [ ] Installateur Windows (.msi / .exe)
- [ ] (Optionnel) Installateur macOS (.dmg)
- [ ] Test de fumée sur machine vierge

### 11.9 Bêta-test & retour
- [ ] 3-5 bêta-testeurs sans explication
- [ ] Mesure du taux de complétion
- [ ] Backlog priorisé

### 11.10 Critères de sortie Phase 11
- ✅ MVP 1 : boucle complète fonctionnelle en < 5 min
- ✅ MVP 2 : boucle complète fonctionnelle en < 2 min
- ✅ Toutes les cibles perf tenues
- ✅ Tests verts
- ✅ Installateur desktop généré et testé
- ✅ 3+ bêta-testeurs ont complété chaque boucle

---

## Synthèse

| Phase | MVP | Livrable |
|-------|-----|----------|
| 0 | — | Cadrage |
| 1 | 1 | Interface de saisie complète |
| 2 | 1 | Pipeline IA + validation technique |
| 3 | 1 | Plan SVG éditable multi-niveaux |
| 4 | 2 | Squelette three.js + render-on-demand |
| 5 | 2 | Import et affinage 2D |
| 6 | 2 | Maison 3D + culling BVH |
| 7 | 2 | IA structure (pièces nommées) |
| 8 | 2 | Catalogue + placement |
| 9 | 2 | Visite FPS + photoréalisme + profil auto |
| 10 | 2 | Robustesse / low-end / finition |
| 11 | 1+2 | Tests, validation, livraison |

> Le projet est mené en solo, sans contrainte de planning. Les phases s'enchaînent dans l'ordre ; on passe à la suivante quand les critères de sortie de la précédente sont remplis.
