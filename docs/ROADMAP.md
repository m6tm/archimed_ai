# Roadmap — Vision complète post-MVP

> Statut : **Vision** (hors MVP 1 et MVP 2). Tout ajout au produit passe par ici pour éviter le scope creep.

## Principe directeur

- **MVP 1** (`docs/MVP1.md`) prouve la boucle *décrire une maison → plan technique 2D IA*.
- **MVP 2** (`docs/MVP2.md`) prouve la boucle *plan 2D → maison 3D → meubler → visiter*.
- Cette roadmap enrichit le produit **par ordre de valeur utilisateur / effort**.

Chaque version doit rester démontrable en une phrase.

---

## V1.0 — MVP 1 : Générateur de plan technique 2D par IA

- Saisie guidée du terrain, de la forme, des étages, du programme et des contraintes
- Dessin à la main levée et import d'image
- Génération IA d'un plan technique 2D (murs, portes, fenêtres, pièces, poteaux)
- Validation des normes techniques françaises générales
- Rendu SVG éditable avec plans côte à côte pour les niveaux
- Export vers le MVP 2

> Voir `docs/MVP1.md` pour le détail.

---

## V2.0 — MVP 2 : De l'éditeur 2D à la visite 3D

- Import du plan généré par le MVP 1
- Éditeur 2D SVG pour affiner le plan
- IA structure : nommer et classer les pièces
- Génération 3D photoréaliste avec culling BVH
- Catalogue de ~25 meubles, placement drag & drop
- Visite FPS avec collision BVH
- Profil machine auto et qualité adaptative
- PWA offline

> Voir `docs/MVP2.md` pour le détail.

---

## V2.1 — Fondations solides

- Sauvegarde cloud (Postgres + auth légère)
- Partage d'un projet **par lien** (read-only d'abord)
- Multi-projets / dashboard utilisateur
- Export **PNG** de la vue 3D (pour réseaux sociaux)

---

## V2.5 — Import pro

- Import **plan PDF/DWG** → conversion auto en murs (OCR + vectorisation)
- Calibrage d'échelle
- Templates de départ (T2, T3, maison R+1)

---

## V3.0 — IA générative

- **Style Transfer** : "scandinave" / "industriel" → l'IA réagence couleurs + objets
- **Suggestions contextuelles** : "il vous manque une table de chevet ici"
- **Ergonomie** : "ce canapé bloque la circulation" (pathfinding simple)
- Budget optimizer (pré-e-commerce)

---

## V3.5 — Vivre avant d'acheter ★ différenciateur

- **Simulation temporelle** : curseur heure/saison, soleil réaliste
- **Simulation habitants** : personnages qui se déplacent, test de circulation
- **Accessibilité** : mode fauteuil roulant / poussette (rayons de giration)
- **Encombrement** : heat-map de densité d'objets

---

## V4.0 — Social & collaboration

- **Visite 3D partagée** temps réel (WebRTC / Liveblocks)
- **Commentaires** post-it sur les murs
- **Votes** famille (quelle couleur pour le salon)
- **Galerie publique** + notation

---

## V4.5 — E-commerce intelligent

- Catalogue **IKEA / Maisons du Monde** (API partenaires)
- **Essai virtuel** : voir le vrai canapé chez soi
- **Devis auto** : liste de courses + prix + magasins
- Occasion : intégration Leboncoin / Emmaüs

---

## V5.0 — Immersion

- **VR** : Meta Quest / WebXR
- **AR** : voir les objets réels chez soi (mobile)
- **Acoustique spatiale** : résonance par pièce
- **Thermique visuelle** : ponts thermiques, isolation

---

## V5.5 — Construction

- **Devis artisans** : appel d'offres depuis le plan
- Export **DXF/DWG** normé pour architectes
- Métrés auto (surfaces, linéaires de murs, volumes)

---

## V6.0 — Communauté & IA compagnon

- **Marketplace** : vendre ses créations (objets 3D, textures, plans)
- **Challenges** : "redécore en 500 €"
- **IA compagnon** : assistant virtuel qui guide, alerte, inspire
- **Prédiction** : "cette déco sera dépassée dans 5 ans"

---

## Matrice vision → version

| Idée originale | Statut |
|----------------|--------|
| Générateur de plan 2D IA | V1.0 |
| Éditeur 2D/3D/visite | V2.0 |
| Multi-étages, escaliers | V2.1+ |
| Extérieur (jardin, piscine) | V3.5 |
| Ray tracing | V5.0 |
| Acoustique | V5.0 |
| Thermique | V5.0 |
| Style Transfer | V3.0 |
| Budget Optimizer | V3.0 |
| Feng Shui / Vastu | V3.0 |
| Ergonomie | V3.0 |
| Simulation habitants | V3.5 |
| Accessibilité | V3.5 |
| Partage 3D temps réel | V4.0 |
| Commentaires / Votes | V4.0 |
| Architecte online | V4.0 |
| Scan réel | V4.5 |
| Essai virtuel IKEA | V4.5 |
| Devis auto | V4.5 |
| Onboarding questionnaire | V3.0 |
| Modes (Rapide/Guidé/Libre/Pro) | V2.5 |
| Feedback "ça marche / c'est beau / c'est cher" | V3.0 |
| Simulation temporelle / saisons | V3.5 |
| Galerie / challenges / marketplace | V6.0 |
| IA compagnon | V6.0 |
| Import PDF/DWG | V2.5 |
| AR | V5.0 |
| VR | V5.0 |
| Domotique | (non planifié) |

---

## Noms de produit (à arbitrer)

- **Habiter** — simple, verbe d'action (recommandé pour le MVP)
- Maison Vivante — émotionnel
- Inside — court, international
- Dessine-Moi une Maison — culturel FR
