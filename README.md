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
6. [Déploiement](#-déploiement)
7. [Utilisation](#-utilisation)
8. [Gestion des erreurs](#-gestion-des-erreurs)
9. [Fichiers de sortie](#-fichiers-de-sortie)
10. [Maintenance](#-maintenance)
11. [Support](#-support)

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
| **Fréquence** | Mensuelle (15e jour à 9h UTC) |
| **Données source** | 500+ produits |
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
│ • clean_erp      │ • clean_web      │ • validate_load_link     │
│ • dedup_erp      │ • dedup_web      │ • log_link               │
│ • test_erp       │ • test_web       │                          │
│ • validate_erp ✓ │ • validate_web ✓ │                          │
│ • log_erp        │ • log_web        │                          │
└──────────────────┴──────────────────┴──────────────────────────┘
                           ⬇️ (MERGE)\n┌─────────────────────────────────────────────────────────────────┐
│  MERGE & ANALYSIS                                                │
├─────────────────────────────────────────────────────────────────┤
│ • merge_files          (INNER JOIN 3 sources)                   │
│ • test_merge + validate_merge  (Quality checks)                 │
│ • sales_total          (SUM calcul)                              │
│ • sales_products       (Excel export)                            │
│ • z_score              (Detection produits premium)              │
│ • premium_csv / ordinary_csv  (Classification)                   │
└─────────────────────────────────────────────────────────────────┘
                           ⬇️ (OUTPUTS)\n┌─────────────────────────────────────────────────────────────────┐
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
├── data/
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
│       ├── sales_product.xlsx
│       ├── premium.csv
│       └── ordinary.csv
│
└── _flows/                     # Flows Kestra (YAML)
    ├── ocde_p10.yml            # Pipeline principal (flux ETL)
    ├── error_notification.yml  # Notifications d'erreurs (trigger)
    ├── git_push.yml            # Push YAML vers GitHub
    └── test.yml                # Flow test/debug
```

---

## 🚀 Installation

### Prérequis

- **Docker** & **Docker Compose** (v2.0+)
- **Git**
- **Compte SMTP** (Gmail, OVH, Sendgrid, etc.) – optionnel pour notifications
- **Compte GitHub** (optionnel) – pour `git_push.yml`

### 1. Cloner le projet

```bash
git clone https://github.com/day811/OCDE-P10.git
cd OCDE-P10
```

Le clonage inclut **automatiquement**:
- ✅ Les 3 fichiers Excel sources
- ✅ La structure `data/sources/`, `data/tmp/`, `data/output/`
- ✅ Les flows YAML dans `_flows/`

### 2. Configurer les variables d'environnement

Créer un fichier `.env` **à la racine du projet** avec les secrets et authentification:

```bash
# ===== AUTHENTIFICATION KESTRA (première connexion) =====
KESTRA_BASIC_AUTH_USERNAME=admin@kestra.io
KESTRA_BASIC_AUTH_PASSWORD=Admin1234

# ===== SMTP (pour notifications d'erreur) =====
ENV_SMTP_HOST=smtp.gmail.com
ENV_SMTP_PORT=587
ENV_SMTP_USER=your-email@gmail.com
ENV_SMTP_PASSWORD=your-app-password
ENV_SMTP_FROM=your-email@gmail.com

# ===== GIT (pour git_push.yml - sauvegarde YAML vers GitHub) =====
GIT_ACCESS_USER=your-github-username
GIT_ACCESS_TOKEN=your-github-token

# ===== POSTGRESQL (par défaut, optionnel à modifier) =====
POSTGRES_DB=kestra
POSTGRES_USER=kestra
POSTGRES_PASSWORD=k3str4
```

**⚠️ Important – Authentification Kestra:**

Le mot de passe `KESTRA_BASIC_AUTH_PASSWORD` doit contenir:
- **Minimum 8 caractères**
- **Au moins 1 majuscule**
- **Au moins 1 chiffre**

Exemple valide: `MyPass123` ✅

**⚠️ Important – Secrets Kestra:**

Les secrets (SMTP, GIT) sont définis dans `.env` (variables d'environnement Docker), **pas dans l'UI**.
Ils deviennent accessibles dans les flows via:

```yaml
password: "{{ secret('SMTP_PASSWORD') }}"
username: "{{ secret('SMTP_USER') }}"
```

> **Note:** La gestion des secrets via l'UI Kestra est réservée à la version **Enterprise**. 
> En version open-source, les secrets utilisent les variables d'environnement Docker.

### 3. Démarrer les services

```bash
docker-compose up -d
```

Attendre ~30-40 secondes que PostgreSQL soit prêt. Vérifier la santé:

```bash
docker-compose ps
```

Accéder à l'interface Kestra: **http://localhost:8080**

**Première connexion:**
- **Email:** Valeur de `KESTRA_BASIC_AUTH_USERNAME` (ex: `admin@kestra.io`)
- **Password:** Valeur de `KESTRA_BASIC_AUTH_PASSWORD` (ex: `Admin1234`)

---

## ⚙️ Configuration

### Variables du pipeline

Les chemins des fichiers sources et outputs sont définis dans `ocde_p10.yml`:

```yaml
variables:
  sourceErp: /app/data/sources/Fichier_erp.xlsx
  sourceWeb: /app/data/sources/Fichier_web.xlsx
  sourceLink: /app/data/sources/fichier_liaison.xlsx
  tempDir: /app/data/tmp
  outputDir: /app/data/output
```

À adapter si vous modifiez la structure des répertoires.

### Configuration SMTP (optionnel)

Si vous souhaitez **activer les notifications d'erreur**:

1. **Gmail:**
   ```bash
   ENV_SMTP_HOST=smtp.gmail.com
   ENV_SMTP_PORT=587
   ENV_SMTP_USER=your-email@gmail.com
   ENV_SMTP_PASSWORD=your-app-password  # Générer via https://myaccount.google.com/apppasswords
   ```

2. **OVH:**
   ```bash
   ENV_SMTP_HOST=ssl0.ovh.net
   ENV_SMTP_PORT=465
   ENV_SMTP_USER=your-email@domain.com
   ENV_SMTP_PASSWORD=your-password
   ```

3. **Sendgrid:**
   ```bash
   ENV_SMTP_HOST=smtp.sendgrid.net
   ENV_SMTP_PORT=587
   ENV_SMTP_USER=apikey
   ENV_SMTP_PASSWORD=your-sendgrid-key
   ```

Après modification de `.env`, redémarrer:
```bash
docker-compose down
docker-compose up -d
```

### Configuration GitHub (optionnel)

Pour **sauvegarder automatiquement les flows vers GitHub**:

1. Créer un **Personal Access Token** sur GitHub:
   - Settings → Developer settings → Personal access tokens → Tokens (classic)
   - Sélectionner le scope `repo` (accès complet)
   - Copier le token

2. Ajouter au `.env`:
   ```bash
   GIT_ACCESS_USER=your-github-username
   GIT_ACCESS_TOKEN=your-token-from-github
   ```

3. Redémarrer Kestra:
   ```bash
   docker-compose down
   docker-compose up -d
   ```

---

## 📖 Déploiement

### Charger les flows dans Kestra

Les flows sont présents dans `_flows/` après le `git clone`.

**Option 1: Via l'UI Kestra (manuel)**

1. Accéder à http://localhost:8080
2. Cliquer sur **"New Flow"**
3. Copier-coller le contenu de `_flows/ocde_p10.yml`
4. Nommer: `ocde_p10`
5. Sauvegarder (bouton en haut à droite)

Répéter pour:
- `_flows/error_notification.yml`
- `_flows/git_push.yml`
- `_flows/test.yml` (optionnel, pour debug)

**Option 2: Via API Kestra (script)**

```bash
# Charger ocde_p10.yml
curl -X POST http://localhost:8080/api/v1/flows \
  -H "Content-Type: application/yaml" \
  -d @_flows/ocde_p10.yml

# Charger error_notification.yml
curl -X POST http://localhost:8080/api/v1/flows \
  -H "Content-Type: application/yaml" \
  -d @_flows/error_notification.yml

# Charger git_push.yml
curl -X POST http://localhost:8080/api/v1/flows \
  -H "Content-Type: application/yaml" \
  -d @_flows/git_push.yml
```

### Sauvegarder les flows vers GitHub

Une fois `git_push.yml` déployé:

1. UI Kestra → Flows → `git_push` → "Execute"
2. Les flows sont pushés vers: `https://github.com/your-username/OCDE-P10/_flows/`

---

## 🔄 Utilisation

### Exécuter le pipeline

**Manuelle:**
- UI Kestra → Flows → `ocde_p10` → "Execute"

**Automatique:**
Le pipeline s'exécute **le 15e jour de chaque mois à 9h UTC**.

Trigger défini dans `ocde_p10.yml`:
```yaml
triggers:
  - id: monthly_schedule
    type: io.kestra.plugin.core.trigger.Schedule
    cron: "0 9 15 * *"
```

### Surveiller l'exécution

1. **Logs en temps réel:**
   - UI Kestra → `ocde_p10` → Dernière exécution → Logs

2. **Statut des tâches:**
   - Chaque tâche affiche: ✅ SUCCESS | ⚠️ WARNING | ❌ FAILED

3. **Notifications d'erreur:**
   - Email automatique si un pipeline ÉCHOUE (voir `error_notification.yml`)

### Télécharger les outputs

Après l'exécution, les fichiers sont dans `/app/data/output/`:

**Depuis votre machine locale:**
```bash
# Copier depuis le conteneur Docker
docker-compose cp kestra:/app/data/output/sales_product.xlsx ./sales_product.xlsx
docker-compose cp kestra:/app/data/output/premium.csv ./premium.csv
docker-compose cp kestra:/app/data/output/ordinary.csv ./ordinary.csv
```

**Depuis l'UI Kestra:**
- Flows → `ocde_p10` → Dernière exécution → Logs
- Chaque fichier généré est listé dans le log final

---

## ⚠️ Gestion des erreurs

### Points de validation critiques

Le pipeline s'arrête si une condition critique échoue:

| Validation | Arrêt ? | Condition |
|-----------|--------|-----------|
| `validate_load_erp` | ⭐ OUI | Fichier vide OU colonnes manquantes |
| `validate_erp_quality` | ⭐ OUI | `product_id IS NULL` OU duplicates détectés |
| `validate_load_web` | ⭐ OUI | Fichier vide OU colonnes manquantes |
| `validate_web_quality` | ⭐ OUI | `sku IS NULL` OU duplicates détectés |
| `validate_load_link` | ⭐ OUI | Fichier vide OU colonnes manquantes |
| `validate_merge` | ⭐ OUI | Perte de données lors du merge |
| `validate_sales` | ⭐ OUI | Prix invalide (NULL, ≤0) OU qty (NULL, <0) |
| `validate_zscore` | ⭐ OUI | Z-score manquant OU incohérent |

### Retry automatique

Les tâches I/O ont une stratégie de retry:

```yaml
retry:
  type: constant
  maxAttempts: 3           # Max 3 tentatives
  interval: PT3M           # Attendre 3 minutes
  maxDuration: PT5M        # Timeout total 5 minutes
```

Appliqué à:
- `load_erp`, `load_web` (lecture Excel)
- `test_erp`, `test_web` (scripts Python)
- `validate_*_quality` (validations)

### Notifications d'erreur

**Flow: `error_notification.yml`**

Si un pipeline ÉCHOUE:
- ✉️ Email automatique à: `day811@laposte.net`
- 📝 Subject: `🔴 ocde_p10 FAILED: {executionId}`
- 🔗 Contenu: lien vers les logs Kestra

> **Important:** Pour activer les notifications, s'assurer que `ENV_SMTP_*` est configuré dans `.env`.

### Déboguer un pipeline échoué

1. **Vérifier les logs:**
   ```bash
   UI Kestra → ocde_p10 → Dernière exécution → Logs
   ```

2. **Vérifie les fichiers temporaires:**
   ```bash
   docker-compose exec kestra ls -lh /app/data/tmp/
   ```

3. **Exécuter le flow test:**
   ```bash
   UI Kestra → test → Execute
   # Ce flow teste uniquement le load_erp pour debug rapide
   ```

---

## 📊 Fichiers de sortie

### Localisation

Tous les fichiers sont dans: `/app/data/output/`

### Format & Contenu

#### 1. **sales_product.xlsx**

Tous les produits vendus, **trié par volume de ventes (DESC)**.

| Colonne | Type | Exemple |
|---------|------|---------|
| product_id | string | `3847-7338` |
| sku | string | `BORDEAUX-2019` |
| post_title | string | `Bordeaux 2019 - Château Palmer` |
| onsale_web | boolean | `true` |
| stock_quantity | int | `150` |
| total_sales | int | `1250` |
| price | float | `45.99` |
| sales_product | float | `57,487.50` |

#### 2. **premium.csv**

Produits **haut de gamme** (z-score ≥ 2 = prix > moyenne + 2×écart-type).

```csv
z_score,product_id,sku,post_title,total_sales,price,sales_product
3.45,5001,GRAND-CRU,Château Lafite Rothschild,500,180.00,90000.00
2.12,5002,PREMIUM,Pomerol Ancien,300,120.50,36150.00
```

#### 3. **ordinary.csv**

Produits **standard** (z-score < 2 = prix ≤ moyenne + 2×écart-type).

```csv
z_score,product_id,sku,post_title,total_sales,price,sales_product
0.50,1001,ROUGE-STD,Vin de Table Rouge,5000,12.50,62500.00
-0.30,1002,BLANC-STD,Vin de Table Blanc,4500,9.99,44955.00
```

### Statistiques d'exécution

À chaque exécution, le log final affiche:

```
════════════════════════════════════════════════════════════
              PIPELINE EXECUTION COMPLETED
════════════════════════════════════════════════════════════
📅 Started: 2026-01-15 09:00:15
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

### Nettoyer les anciennes exécutions

Kestra accumule les logs et métrics au fil du temps.

**Activer le purge automatique:**

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

Puis déployer via l'UI Kestra.

### Monitorer les performances

1. **Kestra UI:** Flows → Statistics
2. **PostgreSQL:** Monitorer la taille de la base
   ```bash
   docker-compose exec postgres psql -U kestra -d kestra -c "SELECT pg_size_pretty(pg_database_size('kestra'));"
   ```
3. **Disque:** Vérifier `/app/data/`
   ```bash
   docker-compose exec kestra du -sh /app/data/
   ```

### Sauvegarder les outputs

Les fichiers Excel/CSV restent dans `/app/data/output/` indéfiniment.

**Recommandation:** Implémenter une sauvegarde mensuelle vers un serveur external (AWS S3, Google Drive, etc.).

### Arrêter les services

```bash
docker-compose down
```

### Supprimer les conteneurs et données

⚠️ **Attention:** Cela supprime tout (base de données, logs, fichiers).

```bash
docker-compose down -v
```

---

## 📞 Support

### Liens utiles

- **Kestra Documentation:** https://kestra.io/docs
- **GitHub Repo:** https://github.com/day811/OCDE-P10
- **Projet:** OpenClassrooms - P10 Data Engineer

### Troubleshooting

#### Q: Le pipeline s'arrête avec "FICHIER VIDE"
**R:** Vérifier que les fichiers Excel existent dans `/app/data/sources/`.

#### Q: "Colonnes manquantes" – Quel fichier?
**R:** Vérifier les noms de colonnes dans le fichier Excel source.

#### Q: Les notifications d'erreur ne s'envoient pas
**R:** Vérifier que `ENV_SMTP_*` est configuré dans `.env` et `error_notification.yml` est déployé.

#### Q: La base PostgreSQL ne démarre pas
**R:** Attendre 30-40 secondes, vérifier les logs:
```bash
docker-compose logs postgres
```

#### Q: Accès à l'UI Kestra refusé
**R:** Vérifier le mot de passe KESTRA_BASIC_AUTH_PASSWORD (8+ chars, 1 majuscule, 1 chiffre).

---

## 📝 Changelog

| Version | Date | Changements |
|---------|------|-------------|
| 1.1 | 2026-01-12 | Correction descriptions, ajout gestion secrets .env |
| 1.0 | 2026-01-09 | Release initiale - Pipeline ETL complet |

---

## 📄 Licence

Projet OpenClassrooms - Usage personnel.

---

**Dernière mise à jour:** 12 janvier 2026 – Kestra v0.50+