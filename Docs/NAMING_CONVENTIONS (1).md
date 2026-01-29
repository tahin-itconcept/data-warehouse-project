# **Conventions de Nommage - Portfolio Data Engineering**

Ce document définit les conventions de nommage pour tous les objets du projet : schémas, tables, colonnes, notebooks, pipelines et mesures DAX dans Microsoft Fabric.

---

## **Table des Matières**

1. [Principes Généraux](#principes-généraux)
2. [Conventions Tables](#conventions-tables)
   - [Couche Bronze](#couche-bronze)
   - [Couche Silver](#couche-silver)
   - [Couche Gold](#couche-gold)
3. [Conventions Colonnes](#conventions-colonnes)
   - [Clés Surrogate](#clés-surrogate)
   - [Colonnes Techniques](#colonnes-techniques)
   - [Colonnes Métier](#colonnes-métier)
4. [Conventions Notebooks](#conventions-notebooks)
5. [Conventions Pipelines](#conventions-pipelines)
6. [Conventions Mesures DAX](#conventions-mesures-dax)
7. [Conventions Fichiers](#conventions-fichiers)

---

## **Principes Généraux**

### **Style de Nommage**
- **Format** : `snake_case` (minuscules avec underscores)
- **Langue** : Anglais uniquement
- **Lisibilité** : Noms explicites et descriptifs
- **Cohérence** : Même pattern pour objets similaires

### **Règles Communes**
✅ **À FAIRE** :
- Utiliser des noms explicites : `customer_email` plutôt que `cust_em`
- Préfixer selon la couche : `bronze_`, `silver_`, `dim_`, `fact_`
- Être cohérent : `order_date` partout (pas `ord_dt`, `order_dt`, etc.)
- Limiter à 64 caractères maximum

❌ **À ÉVITER** :
- Mots réservés SQL : `select`, `from`, `where`, `table`, `order`
- Abréviations obscures : `ctmr`, `prd`, `qty`
- Espaces ou caractères spéciaux : `customer name`, `produit@id`
- CamelCase ou PascalCase : `CustomerOrder`, `productName`

---

## **Conventions Tables**

### **Couche BRONZE (Raw Data)**

**Pattern** : `bronze_<source>_<entity>`

- `bronze_` : Préfixe obligatoire pour la couche Bronze
- `<source>` : Système source des données
- `<entity>` : Nom de l'entité (conservé tel quel de la source)

#### **Exemples Bronze**

| Table Bronze | Description | Source |
|--------------|-------------|--------|
| `bronze_ecommerce_customers` | Clients bruts du système e-commerce | CSV |
| `bronze_ecommerce_orders` | Commandes brutes | CSV |
| `bronze_ecommerce_products` | Produits bruts | CSV |
| `bronze_ecommerce_shipments` | Expéditions brutes | CSV |
| `bronze_erp_suppliers` | Fournisseurs du système ERP | API ERP |
| `bronze_crm_leads` | Prospects du CRM | API CRM |

#### **Colonnes Techniques Bronze**

Toutes les tables Bronze doivent inclure :

```sql
dwh_insert_date TIMESTAMP,          -- Date d'insertion dans Bronze
dwh_source_file STRING,              -- Fichier source (si applicable)
dwh_load_id BIGINT,                  -- ID du batch de chargement
dwh_is_deleted BOOLEAN               -- Flag de suppression logique
```

---

### **Couche SILVER (Cleaned Data)**

**Pattern** : `silver_<source>_<entity>`

- `silver_` : Préfixe obligatoire pour la couche Silver
- `<source>` : Système source (même que Bronze)
- `<entity>` : Nom de l'entité nettoyée

#### **Exemples Silver**

| Table Silver | Description | Transformation depuis Bronze |
|--------------|-------------|------------------------------|
| `silver_ecommerce_customers` | Clients nettoyés | Dédoublonnage, validation emails |
| `silver_ecommerce_orders` | Commandes validées | Correction prix négatifs, dates |
| `silver_ecommerce_products` | Produits enrichis | Prix validés, catégories normalisées |
| `silver_ecommerce_order_items` | Détails articles validés | Quantités > 0, prix cohérents |

#### **Colonnes Techniques Silver**

Ajout de colonnes de qualité :

```sql
dwh_update_date TIMESTAMP,           -- Date dernière modification
dwh_data_quality_score DECIMAL(3,2), -- Score qualité (0.00 - 1.00)
dwh_validation_status STRING,        -- VALID, WARNING, ERROR
dwh_validation_rules STRING          -- Règles appliquées (JSON)
```

---

### **Couche GOLD (Business Data)**

**Pattern** : `<category>_<entity>`

- `<category>` : Type de table (`dim`, `fact`, `agg`, `report`)
- `<entity>` : Nom métier descriptif

#### **Categories Gold**

| Catégorie | Préfixe | Usage | Exemple |
|-----------|---------|-------|---------|
| **Dimension** | `dim_` | Tables de référence, master data | `dim_customers` |
| **Fait** | `fact_` | Tables transactionnelles, métriques | `fact_sales` |
| **Agrégat** | `agg_` | Tables pré-calculées pour performance | `agg_sales_daily` |
| **Rapport** | `report_` | Vues matérialisées pour rapports | `report_customer_360` |
| **Bridge** | `bridge_` | Tables de liaison many-to-many | `bridge_product_category` |

#### **Exemples Gold - Dimensions**

| Table | Description | Type SCD |
|-------|-------------|----------|
| `dim_customers` | Dimension clients | Type 2 |
| `dim_products` | Dimension produits | Type 2 |
| `dim_suppliers` | Dimension fournisseurs | Type 1 |
| `dim_date` | Dimension calendrier | Type 0 (fixe) |
| `dim_geography` | Dimension géographique | Type 1 |

#### **Exemples Gold - Faits**

| Table | Grain | Description |
|-------|-------|-------------|
| `fact_sales` | 1 ligne par article commandé | Transactions de vente |
| `fact_shipments` | 1 ligne par expédition | Historique livraisons |
| `fact_reviews` | 1 ligne par avis client | Notes et commentaires |
| `fact_payments` | 1 ligne par transaction | Paiements reçus |

#### **Exemples Gold - Agrégats**

| Table | Périodicité | Description |
|-------|-------------|-------------|
| `agg_sales_daily` | Quotidienne | CA et volumes par jour |
| `agg_sales_monthly` | Mensuelle | Agrégats mensuels |
| `agg_customer_rfm` | Hebdomadaire | Segmentation RFM |
| `agg_product_performance` | Quotidienne | Performance produits |

---

## **Conventions Colonnes**

### **Clés Surrogate**

**Pattern** : `<table_name>_key`

Toutes les clés primaires des dimensions utilisent le suffixe `_key`.

#### **Exemples**

```sql
-- Dimension Customers
customer_key BIGINT PRIMARY KEY,        -- Clé surrogate
customer_id STRING,                     -- Clé naturelle (source)

-- Dimension Products
product_key BIGINT PRIMARY KEY,         -- Clé surrogate
product_id STRING,                      -- Clé naturelle (source)

-- Dimension Date
date_key INT PRIMARY KEY,               -- Clé surrogate (YYYYMMDD)
full_date DATE                          -- Date complète
```

### **Clés Étrangères**

**Pattern** : Même nom que la clé primaire référencée

```sql
-- Table Fact Sales
customer_key BIGINT,                    -- FK vers dim_customers
product_key BIGINT,                     -- FK vers dim_products
order_date_key INT,                     -- FK vers dim_date
```

### **Colonnes Techniques (Métadonnées)**

**Pattern** : `dwh_<purpose>`

Toutes les colonnes techniques commencent par `dwh_`.

#### **Colonnes Standard**

| Colonne | Type | Description | Couches |
|---------|------|-------------|---------|
| `dwh_insert_date` | TIMESTAMP | Date première insertion | Bronze, Silver, Gold |
| `dwh_update_date` | TIMESTAMP | Date dernière modification | Silver, Gold |
| `dwh_source_system` | STRING | Système source (ecommerce, erp) | Bronze, Silver |
| `dwh_source_file` | STRING | Fichier source d'origine | Bronze |
| `dwh_load_id` | BIGINT | ID du batch ETL | Toutes |
| `dwh_is_deleted` | BOOLEAN | Suppression logique | Toutes |
| `dwh_hash_key` | STRING | Hash pour détection changements | Silver, Gold |

#### **Colonnes SCD Type 2 (Dimensions Gold)**

```sql
dwh_valid_from DATE,                    -- Date début validité
dwh_valid_to DATE,                      -- Date fin validité (NULL = actif)
dwh_is_current BOOLEAN,                 -- TRUE pour version actuelle
dwh_version INT                         -- Numéro de version (optionnel)
```

#### **Colonnes Data Quality (Silver)**

```sql
dwh_data_quality_score DECIMAL(3,2),   -- Score qualité 0.00-1.00
dwh_validation_status STRING,          -- VALID, WARNING, ERROR
dwh_validation_rules STRING,           -- Règles appliquées (JSON)
dwh_cleansed_flag BOOLEAN              -- Données nettoyées ?
```

### **Colonnes Métier**

**Pattern** : Descriptif et explicite

#### **Dates et Timestamps**

```sql
-- Bonnes pratiques
order_date DATE,                        -- Date de commande
order_datetime TIMESTAMP,               -- Date/heure complète
shipment_date DATE,                     -- Date d'expédition
created_at TIMESTAMP,                   -- Horodatage création
updated_at TIMESTAMP                    -- Horodatage modification

-- À éviter
ord_dt, order_ts, ship_d
```

#### **Montants et Quantités**

```sql
-- Bonnes pratiques
total_amount DECIMAL(12,2),             -- Montant total
unit_price DECIMAL(10,2),               -- Prix unitaire
quantity INT,                           -- Quantité
discount_rate DECIMAL(5,4),             -- Taux remise (0.1500 = 15%)

-- Préciser la devise dans le nom si non-standard
total_amount_usd DECIMAL(12,2),
total_amount_eur DECIMAL(12,2)
```

#### **Booléens**

Utiliser préfixe `is_`, `has_`, `can_`

```sql
is_active BOOLEAN,
is_deleted BOOLEAN,
has_discount BOOLEAN,
can_ship_internationally BOOLEAN
```

#### **Catégories et Statuts**

```sql
order_status STRING,                    -- PENDING, CONFIRMED, SHIPPED
payment_method STRING,                  -- CREDIT_CARD, PAYPAL, BANK_TRANSFER
customer_segment STRING                 -- VIP, REGULAR, OCCASIONAL
```

---

## **Conventions Notebooks**

**Pattern** : `<layer>_<sequence>_<purpose>`

### **Structure**

- `<layer>` : bronze, silver, gold
- `<sequence>` : Numéro d'ordre (01, 02, 03...)
- `<purpose>` : Action réalisée

### **Exemples**

```
notebooks/
├── bronze/
│   ├── bronze_01_ingest_customers.py
│   ├── bronze_02_ingest_orders.py
│   └── bronze_03_ingest_products.py
│
├── silver/
│   ├── silver_01_clean_customers.py
│   ├── silver_02_clean_orders.py
│   ├── silver_03_validate_order_items.py
│   └── silver_04_enrich_products.py
│
└── gold/
    ├── gold_01_create_dim_customers.py
    ├── gold_02_create_dim_products.py
    ├── gold_03_create_dim_date.py
    ├── gold_04_create_fact_sales.py
    └── gold_05_create_agg_sales_daily.py
```

---

## **Conventions Pipelines**

**Pattern** : `pl_<action>_<layer>_<entity>`

### **Structure**

- `pl_` : Préfixe obligatoire (Pipeline)
- `<action>` : ingest, transform, load, refresh
- `<layer>` : bronze, silver, gold
- `<entity>` : Nom de l'entité (optionnel)

### **Exemples**

```
pipelines/
├── pl_ingest_bronze_daily.json           # Ingestion quotidienne Bronze
├── pl_transform_silver_all.json          # Transformation Silver complète
├── pl_transform_silver_customers.json    # Transformation Silver - Customers
├── pl_load_gold_dimensions.json          # Chargement dimensions Gold
├── pl_load_gold_facts.json               # Chargement faits Gold
├── pl_refresh_gold_aggregates.json       # Refresh agrégats Gold
└── pl_master_orchestration.json          # Pipeline maître (orchestration)
```

### **Activités dans Pipelines**

**Pattern** : `act_<action>_<object>`

```json
{
  "activities": [
    {
      "name": "act_copy_customers_csv",
      "type": "Copy"
    },
    {
      "name": "act_execute_clean_customers",
      "type": "ExecutePipeline"
    },
    {
      "name": "act_validate_data_quality",
      "type": "Script"
    }
  ]
}
```

---

## **Conventions Mesures DAX**

**Pattern** : Descriptif en PascalCase (standard Power BI)

### **Catégories de Mesures**

#### **Mesures Simples**

```dax
// Pattern: [Metric Name]
Total Revenue
Total Orders
Total Customers
Average Order Value
```

#### **Mesures Temporelles**

```dax
// Pattern: [Metric Name] [Time Period]
Revenue YTD                 -- Year to Date
Revenue QTD                 -- Quarter to Date
Revenue MTD                 -- Month to Date
Revenue LY                  -- Last Year
Revenue LM                  -- Last Month
```

#### **Mesures Calculées**

```dax
// Pattern: [Metric Name] [Calculation Type]
Revenue YoY %               -- Year over Year
Revenue MoM %               -- Month over Month
Revenue Growth %
Customer Retention Rate
```

#### **Mesures KPI**

```dax
// Pattern: KPI - [Metric Name]
KPI - Customer Lifetime Value
KPI - Net Promoter Score
KPI - Churn Rate
KPI - Average Delivery Time
```

### **Exemples Complets**

```dax
// Mesures de base
Total Revenue = SUM(fact_sales[total_amount])
Total Orders = DISTINCTCOUNT(fact_sales[order_id])
Total Customers = DISTINCTCOUNT(fact_sales[customer_key])

// Mesures temporelles
Revenue LY = 
    CALCULATE(
        [Total Revenue],
        SAMEPERIODLASTYEAR(dim_date[full_date])
    )

Revenue YoY % = 
    DIVIDE(
        [Total Revenue] - [Revenue LY],
        [Revenue LY]
    )

// KPIs
KPI - Customer Lifetime Value = 
    AVERAGEX(
        VALUES(dim_customers[customer_key]),
        CALCULATE([Total Revenue])
    )
```

---

## **Conventions Fichiers**

### **Scripts SQL**

**Pattern** : `<layer>_<sequence>_<purpose>.sql`

```
sql/
├── ddl/
│   ├── bronze_01_create_schema.sql
│   ├── silver_01_create_schema.sql
│   └── gold_01_create_schema.sql
│
├── dml/
│   ├── silver_01_clean_customers.sql
│   ├── gold_01_load_dim_customers.sql
│   └── gold_02_load_fact_sales.sql
│
└── procedures/
    ├── sp_load_bronze.sql
    ├── sp_load_silver.sql
    └── sp_load_gold.sql
```

### **Scripts Python**

**Pattern** : `<purpose>_<entity>.py`

```
scripts/
├── generate_sample_data.py
├── validate_data_quality.py
├── profile_customers_data.py
└── test_transformations.py
```

### **Documentation**

**Pattern** : Majuscules pour fichiers principaux

```
docs/
├── README.md
├── ARCHITECTURE.md
├── DATA_DICTIONARY.md
├── NAMING_CONVENTIONS.md
└── DEPLOYMENT_GUIDE.md
```

---

## **Exemples Complets par Couche**

### **Bronze Layer**

```sql
-- Table
bronze_ecommerce_customers

-- Colonnes
customer_id STRING,
first_name STRING,
last_name STRING,
email STRING,
phone STRING,
address STRING,
city STRING,
country STRING,
created_at TIMESTAMP,
-- Colonnes techniques
dwh_insert_date TIMESTAMP,
dwh_source_file STRING,
dwh_load_id BIGINT,
dwh_is_deleted BOOLEAN
```

### **Silver Layer**

```sql
-- Table
silver_ecommerce_customers

-- Colonnes
customer_id STRING,
first_name STRING,
last_name STRING,
email STRING,
email_valid BOOLEAN,              -- Ajouté après validation
phone STRING,
phone_formatted STRING,           -- Ajouté après normalisation
address STRING,
city STRING,
country STRING,
country_code STRING,              -- Ajouté (ISO code)
created_at TIMESTAMP,
-- Colonnes techniques
dwh_insert_date TIMESTAMP,
dwh_update_date TIMESTAMP,
dwh_source_system STRING,
dwh_data_quality_score DECIMAL(3,2),
dwh_validation_status STRING,
dwh_is_deleted BOOLEAN
```

### **Gold Layer - Dimension**

```sql
-- Table
dim_customers

-- Colonnes
customer_key BIGINT PRIMARY KEY,  -- Clé surrogate
customer_id STRING,               -- Clé naturelle
first_name STRING,
last_name STRING,
full_name STRING,                 -- Calculé
email STRING,
phone_formatted STRING,
address STRING,
city STRING,
country STRING,
country_code STRING,
customer_segment STRING,          -- RFM segment
customer_since_date DATE,
-- Colonnes SCD Type 2
dwh_valid_from DATE,
dwh_valid_to DATE,
dwh_is_current BOOLEAN,
dwh_version INT,
-- Colonnes techniques
dwh_insert_date TIMESTAMP,
dwh_update_date TIMESTAMP,
dwh_source_system STRING
```

### **Gold Layer - Fait**

```sql
-- Table
fact_sales

-- Colonnes
sale_key BIGINT PRIMARY KEY,
order_id STRING,
order_item_id STRING,
-- Foreign Keys
customer_key BIGINT,
product_key BIGINT,
supplier_key BIGINT,
order_date_key INT,
shipment_date_key INT,
-- Mesures
quantity INT,
unit_price DECIMAL(10,2),
total_amount DECIMAL(12,2),
discount_amount DECIMAL(10,2),
tax_amount DECIMAL(10,2),
shipping_cost DECIMAL(10,2),
net_amount DECIMAL(12,2),
-- Colonnes techniques
dwh_insert_date TIMESTAMP,
dwh_source_system STRING,
dwh_load_id BIGINT
```

---

## **Checklist de Validation**

Avant de créer un nouvel objet, vérifiez :

- [ ] Le nom respecte le pattern de la couche (bronze_, silver_, dim_, fact_)
- [ ] Le nom est en snake_case (minuscules + underscores)
- [ ] Le nom est en anglais
- [ ] Le nom est explicite et descriptif
- [ ] Le nom ne dépasse pas 64 caractères
- [ ] Le nom n'utilise pas de mots réservés SQL
- [ ] Les colonnes techniques commencent par `dwh_`
- [ ] Les clés primaires se terminent par `_key`
- [ ] Les colonnes SCD Type 2 sont présentes (si dimension)
- [ ] Les colonnes de qualité sont présentes (si Silver/Gold)

---

## **Glossaire des Abréviations Autorisées**

| Abréviation | Signification | Usage |
|-------------|---------------|-------|
| `dim` | Dimension | Préfixe tables Gold |
| `fact` | Fact | Préfixe tables Gold |
| `agg` | Aggregate | Préfixe agrégats Gold |
| `dwh` | Data Warehouse | Préfixe colonnes techniques |
| `pl` | Pipeline | Préfixe pipelines |
| `act` | Activity | Préfixe activités pipeline |
| `sp` | Stored Procedure | Préfixe procédures stockées |
| `rfm` | Recency Frequency Monetary | Segmentation client |
| `ytd` | Year to Date | Période cumulative |
| `yoy` | Year over Year | Comparaison annuelle |
| `clv` | Customer Lifetime Value | KPI client |
| `aov` | Average Order Value | KPI vente |
| `scd` | Slowly Changing Dimension | Type dimension |

---

**Version** : 1.0  
**Date** : Janvier 2026  
**Projet** : Portfolio Data Engineering - Microsoft Fabric  
**Auteur** : Charles (Data Engineer)

---

**📝 Note** : Ces conventions sont alignées sur les best practices Microsoft Fabric, Databricks et les standards de l'industrie data. Elles favorisent la maintenabilité, la lisibilité et la collaboration.
