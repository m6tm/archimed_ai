# MVP 1 — Générateur de plan technique 2D par IA

> Statut : **Plan d'implémentation détaillé**  
> Objectif : permettre à un utilisateur non-expert de décrire une maison et d'obtenir un plan technique 2D éditable, multi-niveaux, respectant les normes françaises générales.

---

## 0. Philosophie produit

Le MVP 1 répond à une frustration simple : **dessiner un plan technique correct est difficile**.  
L'utilisateur décrit sa maison en langage naturel et via quelques gestes visuels ; l'IA produit un plan structuré qu'il peut immédiatement affiner.

**Principes directeurs :**

1. **Décrire avant de dessiner** — l'utilisateur exprime son besoin, l'IA propose un plan.
2. **Toujours éditable** — le plan généré est un brouillon technique, pas une fin en soi.
3. **Normes en guide, pas en cage** — le validateur signale les écarts, l'utilisateur garde le contrôle.
4. **Multi-niveaux natif** — dès la conception, chaque étage est un plan à part entière, aligné avec les autres.
5. **Passage fluide vers le MVP 2** — le plan produit alimente directement l'éditeur 2D/3D existant.

---

## 1. Boucle utilisateur

```
Décrire le terrain
        ↓
Dessiner ou importer la forme de la maison
        ↓
Indiquer le nombre d'étages et la forme de chaque niveau
        ↓
Décrire le programme des pièces par étage
        ↓
Ajouter des contraintes de style (optionnel)
        ↓
IA génère un plan technique 2D
        ↓
Validation technique + rendu SVG éditable
        ↓
Affinage manuel (facultatif) → export vers MVP 2
```

---

## 2. Entrées utilisateur

### 2.1 Terrain

L'utilisateur renseigne les informations d'implantation :

- **Largeur** et **profondeur** du terrain (en mètres)
- **Orientation** principale (Nord, Sud, Est, Ouest)
- **Contraintes d'implantation** (optionnel) : reculs avant/arrière/côtés, mitoyenneté, pente, accès route

Ces données servent à positionner l'enveloppe de la maison et à orienter les pièces de vie.

### 2.2 Forme de la maison

Deux modes d'entrée sont proposés :

| Mode | Description | Usage |
|------|-------------|-------|
| **Dessin à la main levée** | Tracé libre à la souris ou au doigt dans une zone SVG | Croquis rapide, forme personnalisée |
| **Import d'image** | Téléchargement d'une photo ou d'un scan de croquis | Relevé existant, dessin sur papier |

La forme est normalisée en un polygone fermé simplifié. L'application propose une correction automatique si le tracé est ouvert ou trop irrégulier.

### 2.3 Nombre d'étages et forme par niveau

- L'utilisateur indique le **nombre d'étages** (RDC, R+1, etc.).
- Par défaut, chaque niveau reprend la **forme de référence** dessinée.
- L'utilisateur peut **modifier la forme d'un niveau spécifique** (reduction d'emprise, décalage, étage partiel).
- L'IA utilise ces formes comme contrainte stricte pour positionner les pièces et les éléments structurels.

### 2.4 Programme des pièces par étage

L'utilisateur décrit le programme en langage naturel structuré :

- "1 salon de 25 m²"
- "1 cuisine ouverte"
- "3 chambres avec salle de douche"
- "1 WC invités"
- "1 garage"

L'application parse cette saisie en un ensemble de `RoomSpec` :

- type de pièce
- surface cible (si précisée) ou fourchette par défaut
- contraintes attachées (fenêtre obligatoire, accès salle d'eau, proximité d'une entrée, etc.)

### 2.5 Contraintes de style (champ libre)

Champ libre pour exprimer des intentions qualitatives :

- "maison espacée, beaucoup de lumière"
- "entrées de soleil au sud"
- "pièces traversantes"
- "chambres à l'écart du salon"
- "accessible PMR"
- "style contemporain avec patio central"

L'IA interprète ces contraintes comme des objectifs de placement et d'orientation.

---

## 3. Sortie attendue

### 3.1 Plan technique 2D

Le plan généré contient :

- **Murs extérieurs** : suivant l'enveloppe du niveau, avec épaisseur réaliste
- **Murs intérieurs** : délimitant les pièces
- **Portes** : positionnées selon la circulation et les usages
- **Fenêtres** : dimensionnées et orientées en fonction de la lumière naturelle
- **Poteaux / murs porteurs** : placés de manière structurellement cohérente
- **Pièces nommées** : avec surface calculée
- **Cotes et dimensions** : affichées sur le plan

### 3.2 Affichage multi-niveaux

Lorsque la maison compte plusieurs étages, l'interface affiche les plans **côte à côte** :

- chaque niveau dans son propre cadre SVG
- la même échelle pour tous les niveaux
- les éléments structurels alignés verticalement (poteaux, cage d'escalier)
- possibilité de zoom/pan indépendant ou synchronisé

### 3.3 Édition

Le plan est un artefact interactif :

- sélectionner un mur, une pièce, une ouverture
- déplacer, redimensionner, supprimer
- ajouter une ouverture ou une pièce
- modifier les dimensions affichées
- voir les alertes normatives en temps réel

### 3.4 Compatibilité et fidélité avec le MVP 2

Le plan est exporté sous la forme d'un objet `Plan` (voir `MVP2.md` §3) :

- Les coordonnées 2D des murs sont conservées **à l'identique** pour l'extrusion 3D.
- L'enveloppe de chaque niveau dans le MVP 1 devient l'enveloppe exacte du niveau correspondant dans le MVP 2.
- Les épaisseurs de murs, positions de portes/fenêtres et poteaux sont transmis sans approximation.
- Le MVP 2 n'a pas le droit de réinterpréter la géométrie : il extrude et matérialise le plan tel quel.

```
Plan {
  walls: Wall[],
  openings: Opening[],
  rooms: Room[],
  objects: PlacedObject[],
  meta: { name, createdAt, updatedAt, source: 'ai-generated-plan', levels: LevelMeta[] }
}
```

---

## 4. Modèle de données

### 4.1 Entités principales

| Entité | Rôle |
|--------|------|
| `Terrain` | dimensions, orientation, contraintes d'implantation |
| `BuildingEnvelope` | forme de référence de la maison + formes par niveau |
| `Level` | un étage avec son identifiant, sa hauteur, son programme |
| `LevelPlan` | murs, ouvertures, pièces, poteaux d'un niveau |
| `RoomSpec` | pièce demandée par l'utilisateur (type, surface, contraintes) |
| `ConstraintSet` | contraintes qualitatives (style, orientation, accessibilité) |
| `Pillar` | poteau structurel avec position et section |

### 4.2 Format pivot

Le format pivot reste **JSON sérialisable**, compatible avec le modèle `Plan` du MVP 2.

Les poteaux sont ajoutés comme une nouvelle entité temporaire dans le MVP 1 ; ils peuvent être convertis en murs porteurs ou conservés comme annotations structurelles dans le MVP 2.

---

## 5. Pipeline IA

### 5.1 Normalisation des entrées

Avant d'appeler le LLM, les entrées brutes sont normalisées :

1. **Terrain** : conversion en polygone rectangle + orientation.
2. **Forme** :
   - tracé SVG → simplification (Douglas-Peucker), fermeture automatique, conversion en polygone
   - image importée → extraction de contour par modèle de vision ou traitement local
3. **Programme** : parsing en `RoomSpec[]` avec surfaces par défaut selon le type.
4. **Contraintes** : extraction de mots-clés structurés (orientation, style, PMR, etc.).

### 5.2 Génération par LLM

Le LLM reçoit un prompt structuré contenant :

- l'enveloppe de chaque niveau
- le programme des pièces
- les contraintes utilisateur
- un extrait des normes techniques de `docs/STANDARDS.md`

Le LLM produit un JSON structuré et validé par schéma Zod :

```
{
  levels: LevelPlan[],
  rooms: Room[],
  walls: Wall[],
  openings: Opening[],
  pillars: Pillar[],
  warnings: string[]
}
```

### 5.3 Validation technique déterministe

Un validateur indépendant vérifie le plan généré :

- enveloppe fermée et sans auto-intersection
- pièces fermées et non chevauchantes
- surfaces respectant les minima de `STANDARDS.md`
- pièces d'habitation avec ouverture sur l'extérieur
- pièces humides avec ventilation possible
- largeurs de portes et circulations conformes
- poteaux alignés entre les niveaux (si multi-étages)
- portes/fenêtres entièrement sur des murs

### 5.4 Correction et fallback

- Si le plan est invalide, l'IA reçoit les erreurs et tente une correction (retry auto).
- Si le retry échoue, l'application affiche le plan avec les **alertes** et laisse l'utilisateur corriger.
- En cas d'erreur réseau ou de timeout, un fallback propose un plan minimal ou un message clair.

---

## 6. Interface utilisateur

### 6.1 Parcours en 6 étapes

1. **Accueil** : choix "Créer un plan avec l'IA" ou "Dessiner manuellement" (MVP 2).
2. **Terrain** : formulaire de dimensions et d'orientation.
3. **Forme** : zone de dessin + import image + aperçu du polygone simplifié.
4. **Étages** : nombre d'étages + ajustement de la forme par niveau.
5. **Programme** : saisie des pièces par étage.
6. **Style** : champ libre de contraintes qualitatives.

### 6.2 Écran de résultat

- Affichage du plan 2D généré, niveau par niveau, côte à côte.
- Barre d'outils : sélection, déplacement, zoom, undo/redo.
- Panneau latéral : propriétés de l'élément sélectionné, alertes normatives.
- Boutons d'action : **Regénérer**, **Modifier**, **Exporter vers MVP 2**.

### 6.3 États de chargement et feedback

- Spinner explicite pendant la génération IA.
- Messages d'étape : "Analyse de la forme", "Placement des pièces", "Vérification des normes".
- Feedback immédiat sur les actions utilisateur.

---

## 7. Gestion des niveaux

### 7.1 Principe

Chaque niveau est un plan 2D indépendant mais contraint par :

- la forme de référence de la maison
- l'alignement vertical des éléments structurels
- la continuité des circulations verticales (escalier)

### 7.2 Formes par niveau

- **Par défaut** : chaque niveau = forme de référence.
- **Personnalisé** : l'utilisateur peut modifier légèrement la forme d'un niveau (décroché, étage en retrait).
- **Contrainte** : l'IA ne peut pas dépasser l'emprise du terrain.

### 7.3 Poteaux et alignement

- L'IA place les poteaux/murs porteurs de manière à assurer une continuité verticale.
- Un poteau présent au RDC doit avoir un équivalent aux étages supérieurs.
- Le validateur signale tout désalignement structurel.

---

## 8. Normes techniques

Les normes appliquées sont documentées dans `docs/STANDARDS.md`.  
Elles couvrent au minimum :

- surfaces minimales par type de pièce
- hauteurs sous plafond
- largeurs de portes et circulations
- lumière naturelle et ventilation
- accessibilité PMR
- orientation et ensoleillement
- cohérence structurelle (poteaux, portées)

Le MVP 1 applique ces règles de manière indicative et corrective, pas bloquante. L'utilisateur peut outrepasser une alerte s'il le souhaite.

---

## 9. Ce qui est exclu du MVP 1

- Génération 3D (MVP 2)
- Placement de meubles (MVP 2)
- Visite FPS (MVP 2)
- Multi-étages avec formes très complexes (escaliers en spirale, etc.)
- Calcul structurel précis (poutres, fondations)
- Gestion des toitures et des extérieurs
- Import de plans PDF/DWG (roadmap V1.5)
- Coût de construction / devis
- Collaboratif / partage

---

## 10. Définition of Done

Le MVP 1 est livré quand un bêta-testeur peut, en **moins de 5 minutes** :

1. ✅ Renseigner son terrain et dessiner/importer la forme de sa maison
2. ✅ Indiquer le nombre d'étages et le programme des pièces
3. ✅ Obtenir un plan technique 2D avec murs, portes, fenêtres et pièces nommées
4. ✅ Voir les plans de chaque niveau côte à côte (si multi-niveaux)
5. ✅ Modifier le plan généré dans l'interface
6. ✅ Exporter le plan vers le MVP 2 sans perte de données

---

## 11. Risques et mitigations

| Risque | Impact | Mitigation |
|--------|--------|------------|
| Plan IA incohérent géométriquement | Élevé | Validateur déterministe + retry + édition manuelle |
| Non-respect des normes techniques | Moyen | Règles encodées + alertes + fallback |
| Image importée de mauvaise qualité | Moyen | Fallback vers dessin manuel + message explicite |
| Coût API élevé à chaque génération | Moyen | Prompt optimisé, modèle léger (gpt-4.1-mini / Haiku), cache des formes |
| Sortie non compatible MVP 2 | Élevé | Schéma Zod strict aligné sur `Plan` |

---

## 12. Prochaines étapes

1. Valider `docs/MVP1.md` avec les décisions ouvertes.
2. Rédiger `docs/MVP2.md` (adaptation de l'ancien `MVP.md`).
3. Rédiger `docs/STANDARDS.md` avec les règles techniques précises.
4. Mettre à jour `README.md`, `REALIZATION.md`, `ROADMAP.md`.
