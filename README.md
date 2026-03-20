# 📋 Formulaire DiagMob — Documentation

## 🎯 Objectif

**Mon Diag'Mob** est un dispositif numérique développé par le ministère des Transports pour aider les agents publics à changer leurs habitudes de déplacement domicile-travail.

Le formulaire collecte les pratiques de mobilité actuelles de chaque agent, calcule automatiquement des alternatives (vélo, covoiturage, transports en commun), puis déclenche un parcours de changement de comportement personnalisé par email.

> Projet incubé à la Fabrique Numérique du ministère de la transition écologique, porté par l'intraprenariat du minisitère des Transports — [beta.gouv.fr/startups/diag-mob.html](https://beta.gouv.fr/startups/diag-mob.html)

---

## 🏗️ Architecture générale

```
Formulaire DSFR (navigateur)
        │
        ├─ Calcul automatique des trajets (OSRM, ORS, Transitous)
        │
        ▼
    Grist (base de données)
        │
        └─ Webhook → n8n → Brevo (emails personnalisés)
                              └─ Viewer interactif (carte Leaflet)
```

---

## 📝 Fonctionnement du formulaire

### Saisie utilisateur
L'agent renseigne :
- **Adresse domicile / travail** — autocomplétion via l'API BAN (Base Adresses Nationale)
- **Mode de transport principal** — voiture, covoiturage, vélo, VAE, marche, TC, intermodalité
- ** FMD et remboursement transports en communs**
- **Jours et horaires de présence** sur site
- **Informations complémentaires** —  téléphone (facultatif)

### Calcul automatique des trajets (en arrière-plan)
Dès que les deux adresses sont renseignées, le formulaire calcule simultanément :

| Mode | API | Données récupérées |
|------|-----|--------------------|
| 🚗 Voiture | OSRM (Project OSRM) | Distance, durée (avec facteur trafic selon distance) |
| 🚴 Vélo | OpenRouteService (ORS) | Distance, durée, dénivelé +/-, types de voies, revêtements |
| 🚌 TC (à pied) | Transitous / MOTIS | Durée, correspondances, modes utilisés |
| 🚴🚌 TC + Vélo | Transitous / MOTIS | Durée intermodale vélo + TC |

Le calcul TC utilise l'**heure d'arrivée souhaitée** de l'agent pour calculer à rebours l'heure de départ optimale.

### Envoi vers Grist
À la soumission, toutes les données (saisies + calculées) sont envoyées directement à Grist via l'API REST, sans clé API côté client (accès anonyme autorisé par les Access Rules Grist).

---


## 🤖 Pipeline d'analyse et d'envoi d'emails

### 1. Script Python `analyse_matching.py`
Lance l'analyse en local de l'ensemble des répondants :
- Calcul des **clusters géographiques** de lieu de travail
- **Matching covoiturage** : identification des équipages potentiels par intersection de trajets
- **Matching vélotaf** : identification de mentors vélo proches
- Attribution d'un **profil de mobilité** à chaque agent :
  - `velo_musculaire`, `VAE`, `covoit_parfait`, `covoit_journees`, `covoit_itineraire`, `covelotaff`, `deja_mobilite_durable`, `voiture_sanssolution`

Seuls les agents avec `Profil_Mobilite` **vide** sont traités (les déjà analysés sont ignorés).

### 2. Mise à jour Grist
Le script met à jour les colonnes de profil et d'équipage dans Grist, **1 agent à la fois avec 1 seconde de délai**, pour que chaque update déclenche un webhook Grist distinct.

### 3. Webhook Grist → n8n
Configuré sur la colonne `Profil_Mobilite` (événement `update`). Chaque mise à jour déclenche une exécution n8n avec l'ensemble des données de la ligne.

### 4. n8n → Brevo
Un node **Rules** dans n8n route vers le bon template Brevo selon le profil :

| Profil | Template Brevo |
|--------|---------------|
| `velo_musculaire` | Email vélo avec carte interactive + calcul économies |
| `VAE` | Email VAE avec estimation temps -33% |
| `covoit_*` | Email covoiturage avec carte équipage + jours communs |
| `covelotaff` | Email covélotaff |
| `deja_mobilite_durable` | Email félicitations |

Les emails contiennent :
- Une **carte interactive Leaflet** (viewer hébergé sur GitHub Pages)
- Les **économies calculées** (€/mois, CO₂, calories)
- Des **informations sur le FMD** et les aides disponibles

---

## 🔧 Stack technique

| Composant | Technologie |
|-----------|-------------|
| Formulaire | HTML/CSS/JS — Design System de l'État (DSFR) |
| Itinéraires voiture | [OSRM](https://project-osrm.org/) — open source, sans clé |
| Itinéraires vélo | [OpenRouteService](https://openrouteservice.org/) — clé gratuite |
| Transports en commun | [Transitous](https://transitous.org/) — open source |
| Géocodage adresses | [API BAN](https://adresse.data.gouv.fr/api-doc/adresse) — data.gouv.fr |
| Base de données | [Grist](https://www.getgrist.com/) — instance DNUM |
| Automatisation | [n8n](https://n8n.io/) — instance DNUM |
| Emails | [Brevo](https://www.brevo.com/) |
| Cartes interactives | [Leaflet.js](https://leafletjs.com/) — hébergées sur GitHub Pages |
| Analyse Python | Pandas, GeoPandas, Shapely |

---

## 🚀 Déploiement

- **Formulaire** : hébergé sur GitHub Pages (`bapteis/diagmob-widget`)
- **Viewers carte** : hébergés sur GitHub Pages (`bapteis/diagmob-viewer`)
- **Script Python** : exécution manuelle locale (ou planifiée)

---

## 📬 Contact

Baptiste Eisele — [baptiste.eisele@developpement-durable.gouv.fr](mailto:baptiste.eisele@developpement-durable.gouv.fr)

Ministère des Transports - Ministère de la Transition écologique (FABNUM)
