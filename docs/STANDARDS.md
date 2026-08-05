# Normes techniques — Référentiel de conception assistée

> Statut : **Référence** pour le MVP 1.  
> Ce document liste les règles techniques encodées dans l'assistant de plan 2D.  
> ⚠️ **Avertissement** : il s'agit d'un référentiel heuristique pour guider une conception préliminaire. Il ne remplace pas l'avis d'un architecte, d'un ingénieur structure ou des règlements d'urbanisme locaux.

---

## 0. Principes généraux

L'assistant de plan vise à produire des plans **cohérents, habitables et conformes aux usages courants** de la construction individuelle en France. Les règles ci-dessous servent à :

- guider le LLM dans la génération du plan
- alimenter le validateur déterministe post-génération
- signaler à l'utilisateur les écarts éventuels

Toute règle peut être **outrepassée par l'utilisateur** s'il le souhaite explicitement. L'assistant ne bloque jamais la création, il alerte.

---

## 1. Surfaces minimales

Surfaces utiles recommandées pour une pièce déclarée habitable.

| Type de pièce | Surface minimale | Notes |
|---------------|------------------|-------|
| Chambre | **9 m²** | Au moins 2 m de largeur utile |
| Salon / Séjour | **12 m²** | Peut être combiné à la cuisine |
| Cuisine | **5 m²** | 7 m² recommandés si cuisine fermée |
| Salle de bains | **3 m²** | Doit permettre une douche ou une baignoire |
| Salle d'eau (douche) | **2,5 m²** | Doit permettre une douche, un lavabo, des WC ou non |
| WC indépendant | **1,2 m²** | Minimum 0,80 m de largeur |
| Bureau | **7 m²** | Optionnel |
| Entrée / Circulation | **3 m²** | Selon configuration |
| Garage | **12 m²** | Selon usage (1 voiture) |

> Une pièce est considérée comme **habitable** si elle dispose d'une ouverture sur l'extérieur et d'une hauteur sous plafond suffisante.

---

## 2. Hauteurs sous plafond

| Élément | Hauteur minimale | Recommandée |
|---------|------------------|-------------|
| Pièces d'habitation | **2,50 m** | 2,60–2,80 m |
| Salle de bains / WC | **2,30 m** | 2,50 m |
| Garage / local technique | **2,20 m** | 2,40 m |
| Sous combles (surface habitable) | **1,80 m** sous rampant sur au moins la moitié | — |

---

## 3. Largeurs de portes et circulations

### Portes intérieures

| Type | Largeur minimale | Recommandée |
|------|------------------|-------------|
| Porte de passage standard | **0,80 m** | 0,83 m |
| Porte de chambre | **0,80 m** | 0,83 m |
| Porte de salle de bains | **0,70 m** | 0,80 m |
| Porte de WC | **0,70 m** | 0,80 m |
| Porte d'entrée | **0,90 m** | 1,00 m |
| Porte de garage | **2,40 m** | — |

### Circulations

| Élément | Largeur minimale | Notes |
|---------|------------------|-------|
| Couloir | **0,90 m** | 1,00 m recommandé |
| Passage entre meubles | **0,60 m** | — |
| Escalier (largeur) | **0,80 m** | 0,90 m recommandé |
| Escalier (profondeur marche) | **0,24 m** | — |
| Hauteur de marche | **0,17–0,20 m** | — |

---

## 4. Lumière naturelle et ventilation

### Lumière naturelle

- Toute **pièce d'habitation** (salon, séjour, chambre, bureau) doit disposer d'une ouverture sur l'extérieur.
- La surface vitrée doit représenter au moins **1/6 de la surface au sol** de la pièce (règle courante, non réglementaire stricte).
- Les pièces humides (salle de bains, cuisine) doivent idéalement avoir une ouverture ou un système de ventilation mécanique.

### Ventilation

- Une **salle de bains** ou une **cuisine** doit pouvoir être ventilée (fenêtre ou VMC).
- Les WC doivent disposer d'une ventilation directe ou via la VMC.

---

## 5. Accessibilité PMR

Si l'utilisateur active l'option PMR ou "plain-pied accessible", les règles suivantes s'appliquent :

| Élément | Critère |
|---------|---------|
| Largeur de porte | **≥ 0,90 m** |
| Couloir | **≥ 1,40 m** (demi-tour fauteuil) ou **≥ 1,50 m** entre deux portes |
| Seuil | **≤ 2 cm** de hauteur |
| WC accessibles | espace de manoeuvre **≥ 1,50 m × 1,80 m** |
| Douche à l'italienne | **≥ 0,90 m × 1,20 m** |
| Escaliers | éviter ou prévoir un espace pour monte-escalier / futur ascenseur |

---

## 6. Orientation et ensoleillement

L'IA tient compte de l'orientation du terrain pour positionner les pièces :

| Orientation | Pièces recommandées |
|-------------|---------------------|
| **Sud** | Salon, séjour, cuisine (pièces de vie) |
| **Est** | Chambres (lumière du matin) |
| **Ouest** | Bureau, pièces de vie secondaires |
| **Nord** | Salle de bains, WC, cellier, garage, local technique |

### Contraintes de style courantes

| Expression utilisateur | Traduction technique |
|------------------------|----------------------|
| "beaucoup de lumière" | surfaces vitrées augmentées, pièces de vie au sud/est |
| "pièces traversantes" | pièces avec ouvertures sur deux façades opposées |
| "chambres à l'écart" | chambres séparées du salon par un couloir |
| "maison espacée" | pièces généreuses, circulations larges |
| "entrées de soleil au sud" | pièces de vie orientées sud, baies vitrées au sud |

---

## 7. Cohérence structurelle

### Poteaux et murs porteurs

- Les **poteaux** sont placés de manière à assurer une continuité verticale entre les niveaux.
- Un poteau au RDC doit avoir un équivalent aux étages supérieurs.
- La portée entre deux appuis verticaux ne dépasse pas **5 m** en règle générale (pour une dalle béton classique).
- Les murs porteurs extérieurs suivent l'enveloppe du niveau.

### Alignement entre niveaux

- Les plans de chaque niveau doivent être **alignés** lors de l'affichage côte à côte.
- Les éléments structurels communs ( cage d'escalier, gaines techniques) doivent être superposables d'un niveau à l'autre.

---

## 8. Forme et implantation

### Empreinte au sol

- L'emprise de la maison ne doit pas dépasser les dimensions du terrain.
- Les **reculs** (distance à la limite de propriété) sont respectés si l'utilisateur les a indiqués :
  - recul avant : souvent **3 à 5 m**
  - reculs latéraux : souvent **3 m** minimum
  - recul arrière : variable selon les PLU locaux

### Formes simplifiées

L'IA privilégie les formes simples et constructibles :

- rectangle
- carré
- L régulier
- U régulier
- forme avec décrochements simples

Les formes trop complexes (nombreux angles aigus, contours non fermés) sont simplifiées ou signalées.

---

## 9. Règles de validation appliquées

Le validateur déterministe vérifie systématiquement :

1. **Fermeture** : chaque niveau forme un polygone fermé.
2. **Non-chevauchement** : les pièces ne se superposent pas.
3. **Surfaces** : chaque pièce respecte le minimum de son type.
4. **Lumière** : chaque pièce d'habitation a une ouverture sur l'extérieur.
5. **Ventilation** : chaque pièce humide a une ouverture ou une indication VMC.
6. **Circulations** : les portes et couloirs respectent les largeurs minimales.
7. **Portes/fenêtres** : chaque ouverture est entièrement sur un mur.
8. **Structure** : les poteaux sont alignés entre niveaux.
9. **Implantation** : l'emprise ne dépasse pas le terrain.

Chaque échec génère une **alerte** classée par sévérité :

- 🔴 **Bloquante** : plan géométriquement invalide (chevauchement, forme non fermée)
- 🟡 **Normative** : écart aux règles de conception (surface, lumière, circulation)
- 🟢 **Suggestion** : amélioration possible (orientation, optimisation des circulations)

---

## 10. Évolutivité

Ce référentiel est volontairement restreint au MVP 1. Les évolutions futures peuvent inclure :

- intégration des règles de **RT 2012 / RE 2020** (bilan énergétique)
- gestion des **règlements d'urbanisme locaux** (PLU, COS, hauteurs)
- contraintes **sismiques** selon la zone
- règles d'**accessibilité renforcées** (ERP, logements adaptés)
- normes **électriques et sanitaires** (position des prises, arrivées d'eau)

---

## Références

- DTU (Documents Techniques Unifiés) — normes de construction
- NF P 01-012 — Critères généraux d'accessibilité
- RT 2012 / RE 2020 — réglementation thermique
- PLU et règlements d'urbanisme locaux
