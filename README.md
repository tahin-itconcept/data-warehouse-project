# data-warehouse-project

# Contexte du Projet
Entreprise
Société e-commerce fictive spécialisée dans la vente de matériels agricoles (inspirée de mon expérience chez KUHN Group) avec deux branches d'activité :

E-commerce B2C : Vente directe aux particuliers et exploitants agricoles
Distribution B2B : Fourniture de matériels aux revendeurs professionnels

# Organisation

- 5 Unités de Production (France, Allemagne, Espagne, USA)
- 10 Unités de Distribution à travers l'Europe et les USA
-  200,000+ références produits au catalogue
-  7+ million de transactions clients sur 2 ans
- Expéditions multi-transporteurs vers 30+ pays

# État Actuel du Système Data

Situation problématique :
- Données dispersées dans 8 systèmes sources différents sans consolidation
- Qualité des données incertaine : doublons massifs (~10%), valeurs NULL (~5%), incohérence
- Aucune vision temps réel des performances commerciales
- Rapports manuels Excel prenant plusieurs jours à produire
- Impossibilité de croiser les données (clients × produits × ventes × logistique)
- Historique incomplet limitant les analyses de tendances

# Conséquence : 
Les décisions stratégiques se basent sur des données partielles, obsolètes et peu fiables.

## Problématique Métier
Situation Déclenchante
La Direction Générale a identifié plusieurs problèmes critiques nécessitant une intervention data urgente :
1️ - Performance Commerciale
"Baisse inexpliquée de 15% des ventes en Q4 2024"
Questions sans réponse :

- Quels produits/catégories sont en baisse ?
- Quels segments de clients sont affectés ?
- Y a-t-il une corrélation avec les avis clients (qualité) ?
- Est-ce lié à des problèmes logistiques ?
- Comment se compare-t-on à l'année dernière ?

## Impact : Impossible de réagir rapidement sans données consolidées et fiables.

- Gestion du Catalogue
"200,000+ références produits dont beaucoup ne génèrent aucune vente"
Problèmes :

- Pas de visibilité sur les produits à faible rotation
Impossible de calculer la rentabilité réelle (pas de notion de marge)
Stock non optimisé → coûts de stockage élevés
Pas d'analyse de complémentarité produits (market basket)

- Impact : Catalogue inefficace, coûts de stockage excessifs, opportunités de cross-sell manquées.
Rétention Client
"Perte de 30% des clients chaque année"
Défis :

- Impossible d'identifier les clients à risque de churn
Pas de segmentation client pour campagnes ciblées
Customer Lifetime Value (CLV) non mesurée
Impact de la qualité de service (livraison, SAV) non quantifié

- Impact : Coût d'acquisition client élevé sans stratégie de rétention.
Efficacité Logistique
"Coûts de livraison +25% en 6 mois, délais dégradés"
Problématiques :

- Pas de KPIs par transporteur (performance, coût, délai)
Zones géographiques problématiques non identifiées
Routage des commandes non optimisé
Impossibilité de prévoir les volumes pour planifier

- Impact : Explosion des coûts, insatisfaction client, perte de compétitivité.
 Sécurité et Fraude
"Suspicions de fraudes : transactions suspectes, faux avis"
Signaux faibles non détectés :

- Patterns de commandes annulées en masse
Paiements rejetés récurrents
Avis clients suspects (review bombing)
Incohérences montants paiements vs commandes

# Architecture Technique
Modèle Médaillon (Lakehouse Architecture)
┌─────────────────────────────────────────────────────────────┐
│                     SOURCES DE DONNÉES                       │
│  CSV Files / API REST / Base de données / Event Streams     │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                  COUCHE BRONZE (Raw)                         │
│  • Lakehouse: lh_bronze_ecommerce                           │
│  • Format: Delta Lake                                        │
│  • Données brutes non modifiées                             │
│  • Partitionnement: par date d'ingestion                    │
│  • Outil: Data Factory (Copy Activity)                      │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                 COUCHE SILVER (Cleaned)                      │
│  • Lakehouse: lh_silver_ecommerce                           │
│  • Transformations:                                          │
│    - Dédoublonnage                                          │
│    - Nettoyage espaces/casse                                │
│    - Validation formats                                      │
│    - Correction valeurs négatives                           │
│    - Enrichissement avec référentiels                       │
│  • Outil: Notebooks PySpark + Dataflows Gen2                │
│  • Tests: Great Expectations                                │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                  COUCHE GOLD (Business)                      │
│  • Data Warehouse: dw_gold_ecommerce                        │
│  • Modèle: Star Schema                                      │
│  • Dimensions:                                               │
│    - dim_customers (SCD Type 2)                             │
│    - dim_products (SCD Type 2)                              │
│    - dim_date                                                │
│    - dim_suppliers                                           │
│  • Faits:                                                    │
│    - fact_sales                                              │
│    - fact_reviews                                            │
│    - fact_shipments                                          │
│  • Agrégats:                                                 │
│    - agg_sales_daily                                         │
│    - agg_customer_metrics (RFM)                             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              SEMANTIC MODEL & REPORTING                      │
│  • Power BI Semantic Model                                  │
│  • Mesures DAX (Revenue, AOV, CLV, YoY...)                  │
│  • Dashboards:                                               │
│    - Performance Ventes                                      │
│    - Analyse Produits                                        │
│    - Expérience Client                                       │
│    - Logistique                                              │
│    - Détection Fraude                                        │
└─────────────────────────────────────────────────────────────┘

# Objectifs du Projet
1. Data Engineering
- Construire un pipeline ETL/ELT automatisé Bronze → Silver → Gold
- Implémenter le nettoyage et validation des données brutes
- Assurer la qualité des données avec tests automatisés
- Créer un modèle dimensionnel optimisé (Star Schema)
-  Gérer l'historisation avec SCD Type 2
2. Data Quality
  - Détecter et corriger 100% des doublons
  - Gérer les valeurs NULL selon les règles métier
  - Valider l'intégrité référentielle entre tables
  - Implémenter des tests automatisés (Great Expectations)
  - Créer un dashboard de qualité des données
3. Analytics Engineering
- Développer 20+ mesures DAX (Revenue, AOV, CLV, YoY...)
- Créer 5 dashboards Power BI thématiques
- Implémenter la segmentation RFM des clients
- Calculer les KPIs e-commerce standards
- Analyser les tendances et saisonnalité
4. Business Intelligence
  - Répondre aux 5 problématiques métier identifiées
  - Fournir des insights actionnables à la Direction
  - Permettre l'analyse self-service pour les équipes métier
  - Créer des alertes automatiques sur KPIs critiques


