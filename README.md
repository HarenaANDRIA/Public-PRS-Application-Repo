# PRS – Module de Reconnaissance Automatique de la Conformité des PPM

> Système intelligent d'analyse automatique de la conformité des Plans de Passation des Marchés Publics (PPM), développé dans le cadre d'un stage au **Ministère de l'Economie et des Finances (MEF) de Madagascar**, en tant que module du **Procurement Review System (PRS)** utilisé par le CNM.

Mémoire de fin d'études du premier cycle — Licence en Informatique et Télécommunications, parcours **Electronique, Système Informatique et Intelligence Artificielle (ESIIA)** — Institut Supérieur Polytechnique de Madagascar (ISPM), Novembre 2025.

**Auteurs :** ANDRIANTOVOSOA Aina Harentsoa, NARIVONY Tamby Nomena Miarizo
**Encadreur pédagogique :** RAKOTOMALALA Herimiarantsoa Avotraniaina
**Encadreur professionnel :** ANDRIANAIVOLALA Haingonandrianina

---

## Table des matières

- [Contexte et problématique](#contexte-et-problématique)
- [Objectifs du projet](#objectifs-du-projet)
- [Description fonctionnelle](#description-fonctionnelle)
- [Architecture générale](#architecture-générale)
- [Stack technique](#stack-technique)
- [Modèle d'intelligence artificielle](#modèle-dintelligence-artificielle)
- [Modélisation des données](#modélisation-des-données)
- [Fonctionnalités de l'application](#fonctionnalités-de-lapplication)
- [Flux de traitement d'un document](#flux-de-traitement-dun-document)
- [Structure du projet](#structure-du-projet)
- [Environnement de développement](#environnement-de-développement)
- [Résultats obtenus](#résultats-obtenus)
- [Limites connues](#limites-connues)
- [Pistes d'amélioration](#pistes-damélioration)
- [Méthodologie](#méthodologie)
- [Références](#références)

---

## Contexte et problématique

Au sein du MEF malgache, le traitement des Plans de Passation des Marchés Publics (PPM) reste largement **manuel**, ce qui alourdit les délais de validation, augmente le risque d'erreurs ou d'omissions et limite les capacités de contrôle a posteriori.

**Question centrale du projet :** comment évaluer automatiquement la conformité des PPM en s'appuyant sur les critères réglementaires et les données disponibles, afin de moderniser ce processus clé au sein du MEF ?

## Objectifs du projet

**Objectif principal :** concevoir et implémenter un modèle d'intelligence artificielle permettant de détecter automatiquement la conformité des documents PPM à partir des critères définis par la réglementation nationale.

**Objectifs spécifiques :**
- Extraire automatiquement les informations clés des documents PPM (nature du marché, mode de passation, dates, seuils, montants, etc.)
- Classifier chaque document comme **conforme** ou **non conforme** selon les remarques du responsable de vérification
- Concevoir une interface simple permettant de téléverser des documents et de visualiser les résultats

## Description fonctionnelle

Le système repose sur quatre composantes principales :

| Composante | Rôle |
|---|---|
| **Interfaces (web/mobile)** | Inscription, connexion, import de documents, déclenchement de l'analyse, visualisation/correction des résultats, réentraînement du modèle, consultation de l'historique |
| **API** | Communication entre interfaces et base de données ; déclenchement du moteur d'IA |
| **Moteur d'IA** | Extraction de texte, prétraitement, analyse par apprentissage automatique, classification de conformité |
| **Base de données** | Stockage des utilisateurs et de leur historique d'utilisation |

## Architecture générale

L'application suit une **architecture n-tiers** (Web API), séparant clairement la couche présentation, la couche métier et la couche d'accès aux données :

```
┌─────────────────────┐     ┌─────────────────────┐     ┌──────────────────────┐
│   Frontend Web       │     │   Backend            │     │   Base de données     │
│   (Angular CLI)       │◄──►│   (Spring Boot /      │◄──►│   (PostgreSQL 16)      │
│                       │     │   API RESTful)        │     └──────────────────────┘
│   Frontend Mobile     │     │                       │
│   (Flutter)           │     │   ProcessBuilder      │     ┌──────────────────────┐
└─────────────────────┘     │   ──────────────────►│────►│   Moteur d'IA (Python) │
                              └─────────────────────┘     │   CamemBERT + scripts  │
                                                            │   d'extraction/analyse │
                                                            └──────────────────────┘
```

La communication entre le backend Java et le moteur d'IA Python se fait via la classe **`ProcessBuilder`**, qui déclenche des scripts Python externes en leur transmettant dynamiquement des arguments et en gérant les retours d'exécution, les erreurs et la traçabilité.

## Stack technique

Les technologies ont été choisies pour être **identiques à celles déjà utilisées par le ministère**, afin de faciliter l'intégration au projet PRS existant.

| Couche | Technologie | Usage |
|---|---|---|
| Backend | **Java / Spring Boot** | API RESTful, logique métier, sécurité (Spring Security), authentification JWT, journalisation des actions |
| Base de données | **PostgreSQL 16** | Stockage relationnel, intégrité transactionnelle, robustesse |
| Frontend Web | **Angular CLI** | Interfaces web dynamiques et réactives, composants réutilisables |
| Frontend Mobile | **Flutter (Dart)** | Application mobile multiplateforme (Android/iOS), Hot Reload |
| IA / Traitement | **Python 3.8** | Extraction, prétraitement, entraînement et inférence du modèle |
| Bibliothèques IA | **TensorFlow, Keras, scikit-learn, spaCy, pdfplumber** | Réseaux de neurones, NLP, extraction de tableaux PDF |
| Modèle NLP | **CamemBERT** (Transformer, fine-tuné) | Classification conforme / non conforme |
| IDE | **Visual Studio Code** | Développement frontend et moteur d'IA |
| Runtime JS | **Node.js** | Exécution et build d'Angular |

**Environnement matériel de développement :** ordinateur portable Lenovo — CPU Intel Core i7 (11ᵉ génération), GPU intégré Intel Iris Xe (non exploité pour l'entraînement intensif), 8 Go de RAM, disque SSD. L'entraînement du modèle s'exécute sur **CPU**, pas de GPU CUDA dédié.

## Modèle d'intelligence artificielle

- **Architecture :** Transformer
- **Modèle de base :** `camembert-base` (12 couches, 12 têtes d'attention, ~110 millions de paramètres), fine-tuné pour le français
- **Tâche :** classification binaire (conforme / non conforme)
- **Hyperparamètres :** taux d'apprentissage `2e-5`, taille de batch `16`, `20` époques d'entraînement
- **Optimiseur :** Adam
- **Deux scripts principaux :**
  - **Script d'entraînement** : charge `camembert-base`, fine-tune sur les données annotées de PPM, utilise des callbacks pour suivre la progression et éviter le surapprentissage, sauvegarde le modèle appris côté backend.
  - **Script d'analyse (inférence)** : charge le modèle appris, prétraite les nouveaux documents (nettoyage, tokenisation, padding) et prédit leur conformité.

**Pipeline de traitement d'un document :**
1. Extraction des tableaux et des données de chaque cellule via `pdfplumber` (seuillage, détection de lignes et de contours)
2. Conversion des tableaux extraits en **JSON**, puis structuration en **JSONL**
3. Le modèle fine-tuné génère les résultats de conformité par comparaison avec les données labellisées

Le système intègre également une **fonctionnalité de réentraînement continu**, permettant d'améliorer progressivement la précision du modèle à partir des nouvelles données collectées et validées par les utilisateurs.

## Modélisation des données

Conception réalisée selon la méthode **MERISE** (MCD → MLD).

**Entités principales du MCD :**
- **Utilisateur** — identifiant, nom d'utilisateur, mot de passe, rôle
- **Document** — identifiant, nom, chemin fichier, date d'importation, format, statut de conformité
- **Remarque** — identifiant, contenu, date de création
- **ModèleIA** — nom, date d'entraînement, date d'analyse
- **Historique** — horodatage, description de l'action
- **Action** — identifiant, nom, horodatage
- **LotDocuments** — identifiant, date de traitement, nombre de documents

**Principales relations :**
- Un utilisateur peut effectuer plusieurs actions et importer plusieurs documents
- Un document peut être analysé par le modèle d'IA et associé à des remarques
- Les actions des utilisateurs sont tracées dans l'historique
- Les documents peuvent être regroupés en lots (`LotDocuments`)

## Fonctionnalités de l'application

| Interface | Description |
|---|---|
| **Inscription** | Création de compte avec validation en temps réel, choix de rôle (Utilisateur / Administrateur, ce dernier protégé par mot de passe additionnel) |
| **Connexion** | Authentification sécurisée (mot de passe chiffré en transit), gestion d'erreurs explicite |
| **Conditions d'utilisation** | Affichage des droits et obligations, acceptation tacite via bouton « Suivant » |
| **Analyse de documents** *(page principale)* | Import de PDF, prévisualisation, lancement de l'analyse, barre de progression en temps réel, annulation possible, tableau de performance (confiance, temps d'analyse), résultats par fichier avec indicateurs conforme/non conforme, correction manuelle des résultats, validation/ajout aux données de réentraînement |
| **Réentraînement du modèle** *(admin)* | Lancement du réentraînement à partir des nouvelles données validées, barre de progression avec pourcentage, annulation possible |
| **Historique d'utilisation** *(admin)* | Traçabilité complète : utilisateur, rôle, date/heure, action effectuée |

## Flux de traitement d'un document

```
Import PDF → Vérification (Voir PDF) → Analyse
   → Extraction du tableau (pdfplumber) → Conversion JSON/JSONL
   → Classification par le modèle CamemBERT fine-tuné
   → Affichage des résultats (conforme / non conforme + niveau de confiance)
   → Correction manuelle éventuelle par l'utilisateur
   → Validation → Ajout aux données de réentraînement
```

## Structure du projet

```
projet-prs/
├── Backend/
│   ├── projet-spring-boot-1/     # API principale (authentification, gestion documents, historique)
│   └── projet-spring-boot-2/     # Second module Spring Boot
│       └── modele-ia/            # Modèle CamemBERT entraîné (sauvegardé après entraînement)
├── Frontend/
│   └── application-angular/      # Interface web (Angular CLI)
├── mobile/
│   └── application-flutter/      # Interface mobile (Flutter)
└── ia/
    ├── script-entrainement.py    # Fine-tuning de camembert-base
    └── script-analyse.py         # Inférence / classification de conformité
```

> Structure reconstituée à partir de la description du mémoire (« un dossier Backend contenant deux projets Spring Boot, un dossier Frontend contenant l'application Angular »). À adapter à l'arborescence réelle du dépôt de code.

## Environnement de développement

**Prérequis :**
- Java / Spring Boot (JDK compatible)
- Node.js (pour Angular CLI)
- Python **3.8**
- PostgreSQL **16**
- Flutter SDK (pour la version mobile)

**Bibliothèques Python nécessaires :**
```bash
pip install tensorflow keras pdfplumber
# + bibliothèques d'entraînement/inférence CamemBERT (transformers, torch ou tensorflow, scikit-learn, spaCy)
```

**Configuration initiale réalisée :**
- Connexion à la base de données PostgreSQL
- Initialisation du projet Angular (`ng new` / `ng serve`)
- Paramétrage du **CORS** côté backend pour autoriser les requêtes du frontend

## Résultats obtenus

### Extraction automatique des données

| Type de document | Taux de complétude | Taux d'erreur |
|---|---|---|
| **31 PDF numériques natifs** | 100 % sur tous les champs (montants, mode de passation, nature du marché, objet, compte, financement, dates prévisionnelles) | 0 % |
| **13 PDF non numériques** (scans, filigranes) | 0 % | 100 % — aucune couche OCR intégrée actuellement |

### Performance

- **Temps moyen d'analyse complet** (import → affichage des résultats) : **≈ 13 secondes**, stable jusqu'à 10 documents traités en séquence
- **Compatibilité navigateurs** : testée avec succès sur Google Chrome, Mozilla Firefox et Microsoft Edge, sans latence ni erreur d'affichage notable

## Limites connues

- **Absence de module OCR** : les documents PPM scannés ou non numériques ne sont pas exploitables par le système actuel
- Entraînement du modèle réalisé sur **CPU** uniquement (pas d'accélération GPU/CUDA)
- Volume de données d'entraînement limité (projet en phase de preuve de concept)

## Pistes d'amélioration

1. **Module OCR** *(priorité absolue)* — intégrer Tesseract (open source) ou un service cloud (Google Cloud Vision, Amazon Textract, Azure Cognitive Services) pour traiter les documents scannés ; ajouter des prétraitements d'image (redressement, désaturation, contraste, suppression de bruit/filigranes) et des mécanismes de post-correction
2. **Qualité des documents PPM** — sensibiliser à la production de documents numériques natifs, car même un OCR performant ne compense pas une mauvaise qualité source
3. **Modèle d'IA** — augmenter le volume et la diversité des données d'entraînement, appliquer des techniques d'augmentation de données, explorer d'autres architectures Transformer (BERT, RoBERTa)
4. **Interface** — afficher dans l'historique les remarques ajoutées/modifiées avec leurs références précises (ligne/colonne)
5. **Scalabilité** — envisager un déploiement cloud (GCP, AWS, Azure) pour une haute disponibilité et une scalabilité automatique

## Méthodologie

Conception réalisée selon la méthode **MERISE**, structurée en deux axes :
- **Axe des cycles de vie** (horizontal) : phases successives du projet
- **Axe des préoccupations** (vertical) : niveaux **Conceptuel** (QUOI), **Logique** (COMMENT), **Physique** (AVEC QUOI)

Modèles produits : **MCD** (Modèle Conceptuel de Données), **MLD** (Modèle Logique de Données), avec validation croisée MCD/MCT.

## Références

- Documentation officielle [Angular](https://angular.dev)
- Documentation officielle [Spring Boot](https://spring.io/projects/spring-boot)
- Documentation officielle [Flutter](https://flutter.dev)
- Documentation officielle [PostgreSQL](https://www.postgresql.org/docs/)
- Devlin, J. et al. (2018). *BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding*. arXiv:1810.04805
- Abadi, M. et al. (2016). *TensorFlow: Large-Scale Machine Learning on Heterogeneous Distributed Systems*. arXiv:1603.04467
- Martin, R. C. (2008). *Clean Code: A Handbook of Agile Software Craftsmanship*. Prentice Hall

---

*Ce README a été généré à partir du mémoire de fin d'études « Conception d'un modèle d'intelligence artificielle pour la reconnaissance de la conformité des Plans de Passation des Marchés Publics » (ISPM, Novembre 2025). Il synthétise l'architecture, les technologies et les résultats décrits dans le document ; il ne remplace pas une documentation technique issue directement du code source.*
