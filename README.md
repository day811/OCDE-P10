# 🍷 OCDE-P10: Wine Sales ETL Pipeline

**Mise en place d'un pipeline d'orchestration des flux de données avec Kestra**

Projet OpenClassrooms pour l'automatisation de l'ingestion, nettoyage, réconciliation et analyse des données de ventes de vin pour l'entreprise **BottleNeck.fr**.

---

## 📋 Table des matières

1. [Vue d'ensemble](#-vue-densemble)
2. [Architecture](#-architecture)
3. [Structure du projet](#-structure-du-projet)
4. [Installation](#-installation)
5. [Configuration](#-configuration)
6. [Utilisation](#-utilisation)
7. [Flux de données](#-flux-de-données)
8. [Gestion des erreurs](#-gestion-des-erreurs)
9. [Outputs](#-outputs)
10. [Maintenance](#-maintenance)

---

## 🎯 Vue d'ensemble

Ce pipeline **Kestra** automatise le processus ETL (Extract, Transform, Load) pour les données de ventes de vin:

- **Extraction** depuis 3 sources Excel (ERP, Web, Liaison)
- **Nettoyage** et **déduplication** en parallèle
- **Validation** de qualité avec arrêt si erreurs critiques
- **Réconciliation** via fichier de liaison (product_id ↔ sku)
- **Enrichissement** avec calcul du z-score (détection produits premium)
- **Export** en fichiers Excel et CSV

| Métrique | Valeur |
|----------|--------|
| **Durée cible** | ~10-15 minutes |
| **Fréquence** | Mensuelle (15e jour à 9h) |
| **Données source** | 500+ produits ERP + web |
| **Format output** | XLSX + CSV |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      KESTRA PIPELINE                             │
├──────────────────┬──────────────────┬──────────────────────────┤
│  ERP BRANCH      │  WEB BRANCH      │  LIAISON BRANCH          │
├──────────────────┼──────────────────┼──────────────────────────┤
│ • load_erp       │ • load_web       │ • load_link              │
│ • clean_erp      │ • clean_web      │                          │
│ • dedup_erp      │ • dedup_web      │                          │
│ • test_erp       │ • test_web       │                          │
│ • validate_erp ✓ │ • validate_web ✓ │                          │
│ • log_erp        │ • log_web        │                          │
└──────────────────┴──────────────────┴──────────────────────────┘
                           ⬇️ (MERGE)
┌─────────────────────────────────────────────────────────────────┐
│  MERGE & ANALYSIS                                                │
├─────────────────────────────────────────────────────────────────┤
│ • merge_files          (INNER JOIN 3 sources)                   │
│ • sales_total          (SUM calcul)                              │
│ • sales_products       (Excel export)                            │
│ • zscore               (Detection produits premium)              │
│ • premium_csv / ordinary_csv  (Classification)                   │
│ • log_final            (Report)                                  │
└─────────────────────────────────────────────────────────────────┘
                           ⬇️ (OUTPUTS)
┌─────────────────────────────────────────────────────────────────┐
│  OUTPUT FILES                                                    │
├─────────────────────────────────────────────────────────────────┤
│ • sales_product.xlsx   (Tous les produits + ventes)             │
│ • premium.csv          (z-score ≥ 2)                             │
│ • ordinary.csv         (z-score < 2)                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📁 Structure du projet

```
OCDE-P10/
├── docker-compose.yml          # Configuration Docker (Kestra + PostgreSQL)
├── .env                        # Variables d'environnement (secrets SMTP, GIT, auth)
├── README.md                   # Ce fichier
│
├── data/                       # ✅ Créés automatiquement par git clone
│   ├── sources/                # Fichiers sources (input)
│   │   ├── Fichier_erp.xlsx          (500+ produits, IDs, prix, stock)
│   │   ├── Fichier_web.xlsx          (WooCommerce products, sales)
│   │   └── fichier_liaison.xlsx      (Mapping product_id ↔ sku)
│   │
│   ├── tmp/                    # Fichiers temporaires (parquet)
│   │   ├── erp_raw.parquet
│   │   ├── erp_clean.parquet
│   │   ├── dedup_erp.parquet
│   │   ├── web_raw.parquet
│   │   ├── dedup_web.parquet
│   │   ├── link_raw.parquet
│   │   ├── merge.parquet
│   │   └── z_score.csv
│   │
│   └── output/                 # Fichiers finaux (exports)
│       ├── sales_product.xlsx       (Tous produits + ventes)
│       ├── premium.csv             (Produits haut de gamme)
│       └── ordinary.csv            (Produits standard)
│
└── _flows/                     # ✅ Créés automatiquement par git clone
    ├── ocde_p10.yml            # Pipeline principal (flux ETL)
    ├── error_notification.yml  # Flow notifications d'erreurs (trigger)
    ├── git_push.yml            # Flow push YAML vers GIT
    └── test.yml                # Flow test SMTP
```

---

## 🚀 Installation

### Prérequis

- Docker & Docker Compose
- Git
- Compte GitHub (optionnel, pour git_push.yml)
- Serveur SMTP (ex: Gmail, OVH, etc.)

### 1. Cloner le projet

```bash
git clone https://github.com/day811/OCDE-P10.git
cd OCDE-P10
```

Le clonage inclut **automatiquement**:
- ✅ Les 3 fichiers Excel sources (`Fichier_erp.xlsx`, `Fichier_web.xlsx`, `fichier_liaison.xlsx`)
- ✅ La structure complète des répertoires `data/sources/`, `data/tmp/`, `data/output/`
- ✅ Les flows YAML dans `_flows/`

### 2. Configurer les variables d'environnement

Créer un fichier `.env` à la racine avec les **secrets et authentification**:

```bash
# ===== AUTHENTIFICATION KESTRA (première connexion) =====
KESTRA_BASIC_AUTH_USERNAME=admin@kestra.io
KESTRA_BASIC_AUTH_PASSWORD=Admin1234

# ===== SECRETS KESTRA (définis via .env) =====

# SMTP Configuration (email notifications)
ENV_SMTP_HOST=smtp.gmail.com
ENV_SMTP_USER=your-email@gmail.com
ENV_SMTP_PASSWORD=your-app-password
ENV_SMTP_PORT=587

# GIT Configuration (pour git_push.yml - push YAML vers GitHub)
GIT_ACCESS_USER=your-github-username
GIT_ACCESS_TOKEN=your-github-token

# ===== POSTGRESQL (déjà configuré, optionnel à modifier) =====
POSTGRES_DB=kestra
POSTGRES_USER=kestra
POSTGRES_PASSWORD=k3str4
```

> **⚠️ Important - Authentification Kestra:**
> - `KESTRA_BASIC_AUTH_PASSWORD` doit contenir:
>   - **Minimum 8 caractères**
>   - **1 majuscule** et **1 chiffre**
>   - Exemple: `MyPass123` ✅

### 3. Démarrer les services

```bash
docker-compose up -d
```

Attendre ~30-40 secondes que PostgreSQL soit prêt.

Accéder à l'UI: **http://localhost:8080**

**Première connexion:**
- **Email:** valeur de `KESTRA_BASIC_AUTH_USERNAME` (ex: `admin@kestra.io`)
- **Password:** valeur de `KESTRA_BASIC_AUTH_PASSWORD` (ex: `Admin1234`)

---

## ⚙️ Configuration

### Variables d'environnement (.env)

Les **secrets** sont définis dans le fichier `.env` (variables d'environnement Docker), **pas dans l'UI**.

Vérifier que les variables suivantes sont configurées dans `.env`:

```bash
# SMTP (pour notifications d'erreur)
ENV_SMTP_HOST=smtp.gmail.com
ENV_SMTP_USER=your-email@gmail.com
ENV_SMTP_PASSWORD=your-app-password
ENV_SMTP_PORT=587

# GIT (pour git_push.yml - sauvegarde YAML vers GitHub)
GIT_ACCESS_USER=your-github-username
GIT_ACCESS_TOKEN=your-github-token
```

Ces variables deviennent automatiquement accessibles dans les flows via:

```yaml
host: "{{ secret('SMTP_HOST') }}"
password: "{{ secret('SMTP_PASSWORD') }}"
username: "{{ secret('SMTP_USER') }}"
```

> **Note:** La configuration des secrets via l'UI Kestra est une fonctionnalité **réservée à la version entreprise**.
> En version open-source, les secrets sont définis via variables d'environnement (.env).

### Variables du pipeline

Les variables de chemins sont définies dans `ocde_p10.yml`:

```yaml
variables:
  sourceErp: /app/data/sources/Fichier_erp.xlsx
  sourceWeb: /app/data/sources/Fichier_web.xlsx
  sourceLink: /app/data/sources/fichier_liaison.xlsx
  tempDir: /app/data/tmp
  outputDir: /app/data/output
```

À adapter si vous modifiez la structure des répertoires.

---

## 📖 Utilisation

### Déployer les flows

Les flows sont déjà présents dans `_flows/` après le `git clone`.

**Pour charger les flows dans Kestra:**

1. **Via l'UI Kestra:**
   - Accéder à http://localhost:8080
   - Cliquer sur "New Flow"
   - Copier le contenu de `_flows/ocde_p10.yml`
   - Sauvegarder

   Répéter pour:
   - `_flows/error_notification.yml`
   - `_flows/git_push.yml`
   - `_flows/test.yml`

2. **Sauvegarde automatique vers GitHub:**
   - Une fois `git_push.yml` déployé
   - L'exécuter pour pusher tous les flows vers `https://github.com/your-username/OCDE-P10/_flows/`

### Exécuter le pipeline

**Manuelle:**
- UI Kestra → Flows → `ocde_p10` → "Execute"

**Automatique:**
- Le pipeline s'exécute le **15e jour de chaque mois à 9h UTC**
- Trigger défini dans `ocde_p10.yml`:
  ```yaml
  triggers:
    - id: monthly_schedule
      type: io.kestra.plugin.core.trigger.Schedule
      cron: "0 9 15 * *"
  ```

### Surveiller l'exécution

1. **UI Kestra:** Flows → `ocde_p10` → Dernière exécution
2. **Logs:** Affichage détaillé de chaque tâche
3. **Notifications:** Email en cas d'erreur (déclencheur `error_notification`)

---

## 🔄 Flux de données

### Phase 1: Extraction & Nettoyage (Parallèle)

#### Branche ERP
1. **load_erp** → Charge Excel → Table DuckDB
   - Retry: 3 tentatives, 3min d'intervalle
   - Output: `erp_raw.parquet` (500+ rows)

2. **clean_erp** → Supprime `product_id IS NULL`
   - Output: `erp_clean.parquet`

3. **dedup_erp** → `DISTINCT ON(product_id)`
   - Output: `dedup_erp.parquet`

4. **test_erp** → Python script (DuckDB)
   - Compte: `NULL product_id`, duplicates
   - Output: `missings`, `duplicated` vars

5. **validate_erp_quality** → ⭐ CRITICAL
   - Stop si: `missings > 0` OU `duplicated > 0`
   - Exception levée = Pipeline FAILED

6. **log_erp** → Rapport logs

#### Branche WEB
Même flux avec colonne `sku` au lieu de `product_id`.

#### Branche LIAISON
**load_link** → Charge fichier mapping (`product_id → id_web`)

### Phase 2: Fusion (Sequential)

**merge_files**
```sql
SELECT e.*, w.* FROM
  dedup_erp AS e
  INNER JOIN link_raw AS l ON e.product_id = l.product_id
  INNER JOIN dedup_web AS w ON l.id_web = w.sku
```
- Réconcilie les 3 sources
- Output: `merge.parquet`

**sales_total** → `SUM(total_sales * price)`

**sales_products** → Export XLSX ordonné par ventes

### Phase 3: Classification (Parallèle)

**zscore** → Calcul z-score pour chaque prix

```python
z_score = (price - price.mean()) / price.std()
```

**premium_csv** → z-score ≥ 2 (haut de gamme)
**ordinary_csv** → z-score < 2 (standard)

### Phase 4: Finalisation

**log_final** → Rapport complet + stats

---

## ⚠️ Gestion des erreurs

### Retry Policy

**Automatique:**
```yaml
retry:
  type: constant
  maxAttempts: 3           # Max 3 tentatives
  interval: PT3M           # Attendre 3 minutes entre tentatives
  maxDuration: PT5M        # Timeout total 5 minutes
```

Appliqué à:
- `load_erp`, `load_web` (I/O risqué)
- `test_erp`, `test_web` (Python + DuckDB)
- `validate_erp_quality`, `validate_web_quality`

### Validation & Arrêt

Les tâches **validate_erp_quality** et **validate_web_quality** sont **CRITIQUES**:
- Si données invalides → Exception levée
- Pipeline FAILED immédiatement
- Email de notification envoyé (voir ci-dessous)

Seuils de validation:
```python
MAX_MISSINGS = 0          # 0 NULL accepté
MAX_DUPLICATED = 0        # 0 doublon accepté
```

### Notifications d'erreurs

**Flow: error_notification.yml**

Trigger automatique:
- Condition: `ExecutionStatus = FAILED`
- Namespace: `company.team` (prefix match)

Action:
- Email à `day811@laposte.net`
- Subject: `🔴 ocde_p10 FAILED: {executionId}`
- Contenu: lien vers logs

---

## 📊 Outputs

### Format & Localisation

Tous les fichiers sont générés dans `/app/data/output/`:

#### 1. **sales_product.xlsx**
Tous les produits vendus, trié par volume de ventes.

| Colonne | Type | Exemple |
|---------|------|---------|
| product_id | string | `3847-7338` |
| sku | string | `BORDEAUX-2019` |
| post_title | string | `Bordeaux 2019 - Château Palmer` |
| total_sales | int | `1250` |
| price | float | `45.99` |
| sales_product | float | `57,487.50` |

#### 2. **premium.csv**
Produits haut de gamme (z-score ≥ 2 = prix > moyenne + 2×écart-type).

```
z_score,product_id,sku,post_title,total_sales,price,sales_product
3.45,5001,GRAND-CRU,Château Lafite Rothschild,500,180.00,90000.00
2.12,5002,PREMIUM,Pomerol Ancien,300,120.50,36150.00
...
```

#### 3. **ordinary.csv**
Produits standard (z-score < 2 = prix ≤ moyenne + 2×écart-type).

```
z_score,product_id,sku,post_title,total_sales,price,sales_product
0.50,1001,ROUGE-STD,Vin de Table Rouge,5000,12.50,62500.00
-0.30,1002,BLANC-STD,Vin de Table Blanc,4500,9.99,44955.00
...
```

### Statistiques

À chaque exécution, les logs affichent:

```
✅ ════════════════════════════════════════════════════════════
              PIPELINE EXECUTION COMPLETED
════════════════════════════════════════════════════════════
📊 Merged records: 387
💰 Total sales: $1,245,678.92

📂 Output files:
  - sales_product.xlsx
  - premium.csv (z-score > 2)
  - ordinary.csv (z-score <= 2)

✅ All validations passed
✅ All exports completed
════════════════════════════════════════════════════════════
```

---

## 🧹 Maintenance

### Purger les anciennes exécutions

Kestra accumule les logs et métrics. Mettre en place un nettoyage automatique:

Créer `_flows/purge_executions.yml`:

```yaml
id: purge_executions
namespace: system

tasks:
  - id: purge_old_executions
    type: io.kestra.plugin.core.execution.PurgeExecutions
    endDate: "{{ now() | dateAdd(-30, 'DAYS') }}"
    states:
      - SUCCESS
      - WARNING
    purgeLog: true
    purgeMetric: true
    purgeStorage: true

triggers:
  - id: daily_purge
    type: io.kestra.plugin.core.trigger.Schedule
    cron: "0 2 * * *"  # Chaque jour à 2h du matin
```

### Monitorer les performances

1. **Kestra UI:** Flows → Statistics
2. **PostgreSQL:** Monitorer la taille DB
3. **Disque:** Vérifier `/app/data/tmp` (nettoyage automatique après exec)

### Sauvegarder les outputs

Les fichiers Excel/CSV restent dans `/app/data/output/` indéfiniment.
Recommandation: implémenter une sauvegarde mensuelle.

---

## 📞 Support & Documentation

- **Kestra Docs:** https://kestra.io/docs
- **GitHub Repo:** https://github.com/day811/OCDE-P10
- **Projet:** OpenClassrooms - P10 Data Engineer

---

## 📝 Changelog

| Version | Date | Changements |
|---------|------|-------------|
| 1.1 | 2026-01-09 | Correction config secrets (.env), auth Kestra, git_push |
| 1.0 | 2026-01-09 | Release initiale - Pipeline ETL complet |

---

## 📄 Licence

Projet OpenClassrooms - Usage personnel.

---

**Dernière mise à jour:** 9 janvier 2026 - Kestra v0.50+