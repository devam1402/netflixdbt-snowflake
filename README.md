# 🎬 netflix-dbt-snowflake

> A production-style **dbt** project that models Netflix titles data inside **Snowflake** — raw ingestion through staging, dimensional tables, fact tables, and analytical queries, all version-controlled and test-covered.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Data Model](#data-model)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Snowflake Setup](#snowflake-setup)
  - [dbt Profile Configuration](#dbt-profile-configuration)
  - [Install Dependencies](#install-dependencies)
- [Running the Project](#running-the-project)
- [Testing](#testing)
- [Seeds](#seeds)
- [Snapshots](#snapshots)
- [Analyses](#analyses)
- [Macros](#macros)
- [Contributing](#contributing)

---

## Overview

`netflix-dbt-snowflake` is an **end-to-end analytics engineering project** that transforms raw Netflix content data into clean, analytics-ready dimensional and fact tables using **dbt Core** on top of **Snowflake**.

It follows the **medallion / layered modeling** pattern:

- **Staging** — light cleaning and renaming of raw source data
- **Dim (Dimension Tables)** — slowly-changing descriptive entities (titles, genres, countries, ratings)
- **Fct (Fact Tables)** — measurable events and metrics (releases by year, content by type)
- **Analyses** — ad-hoc SQL explorations and business questions answered directly in dbt

Key highlights:
- 🏗️ **Layered dbt models** — staging → dim → fct separation of concerns
- ❄️ **Snowflake** as the cloud data warehouse with warehouse, database, and schema isolation
- 🌱 **Seeds** for small reference/lookup data loaded directly via dbt
- 📸 **Snapshots** for tracking slowly-changing dimension history (SCD Type 2)
- 🧪 **dbt Tests** — `not_null`, `unique`, `accepted_values`, and `relationships` out of the box
- 🔁 **Macros** for reusable SQL logic across models
- 📦 **dbt packages** (e.g. `dbt_utils`) for extended test and utility coverage

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Data Source                            │
│           Netflix Titles Dataset (CSV / S3)                 │
└──────────────────────────┬──────────────────────────────────┘
                           │  COPY INTO / Seeds
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    Snowflake Data Warehouse                  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  RAW / SOURCE Layer                                  │   │
│  │  (raw Netflix data tables)                           │   │
│  └────────────────────┬────────────────────────────────┘   │
│                        │  dbt run (staging models)          │
│  ┌─────────────────────▼──────────────────────────────┐    │
│  │  STAGING Layer  (views)                             │    │
│  │  stg_netflix__titles, stg_netflix__...              │    │
│  └────────────────────┬───────────────────────────────┘    │
│                        │  dbt run (dim / fct models)        │
│  ┌─────────────────────▼──────────────────────────────┐    │
│  │  MARTS Layer  (tables)                              │    │
│  │  dim_titles  │  dim_genres  │  dim_countries        │    │
│  │  fct_releases │  fct_content_by_type               │    │
│  └────────────────────┬───────────────────────────────┘    │
│                        │                                    │
│  ┌─────────────────────▼──────────────────────────────┐    │
│  │  ANALYSES  (ad-hoc SQL insights)                    │    │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
              📊 BI Tool / Dashboards
         (Metabase, Tableau, Snowsight, etc.)
```

---

## Tech Stack

| Layer | Tool |
|---|---|
| Transformation | [dbt Core](https://docs.getdbt.com/) |
| Data Warehouse | [Snowflake](https://www.snowflake.com/) |
| Language | SQL + Jinja2 |
| Package Manager | dbt packages (`packages.yml`) |
| Source Data | Netflix Titles Dataset |
| Version Control | Git + GitHub |

---

## Project Structure

```
netflixdbt-snowflake/
├── analyses/                  # Ad-hoc analytical SQL queries
├── macros/                    # Reusable Jinja2 SQL macros
├── models/                    # Core dbt transformation models
│   ├── staging/               # Stg models — light cleaning of raw sources
│   │   └── stg_netflix__*.sql
│   ├── dim/                   # Dimension tables (materialized: table)
│   │   ├── dim_titles.sql
│   │   ├── dim_genres.sql
│   │   ├── dim_countries.sql
│   │   └── dim_ratings.sql
│   └── fct/                   # Fact tables (materialized: table)
│       ├── fct_releases.sql
│       └── fct_content_by_type.sql
├── seeds/                     # Static CSV reference data
├── snapshots/                 # SCD Type 2 snapshot definitions
├── tests/                     # Custom singular dbt tests
├── .gitignore
├── dbt_project.yml            # Project config (name: netflix)
├── packages.yml               # dbt package dependencies
└── package-lock.yml           # Locked package versions
```

---

## Data Model

### Materialization Strategy

| Layer | Materialization | Why |
|---|---|---|
| Staging | `view` | Lightweight; no storage cost, always fresh |
| Dim | `table` | Stable reference data; fast joins |
| Fct | `table` | Aggregate metrics; query performance |

### Key Models

**Dimension Tables**

| Model | Description |
|---|---|
| `dim_titles` | Cleaned Netflix title records — title, type, release year, rating, duration |
| `dim_genres` | Exploded genre/category per title |
| `dim_countries` | Countries where each title is available |
| `dim_ratings` | Distinct content ratings (PG, TV-MA, etc.) with descriptions |

**Fact Tables**

| Model | Description |
|---|---|
| `fct_releases` | Count of new titles added to Netflix per year |
| `fct_content_by_type` | Breakdown of Movies vs TV Shows over time |

### Lineage (DAG)

```
raw_netflix_titles
       │
       ▼
stg_netflix__titles
       │
       ├──────────────────────────────────┐
       ▼                                  ▼
dim_titles          dim_genres      dim_countries
       │                  │
       └──────────────────┘
                │
                ▼
         fct_releases
         fct_content_by_type
```

---

## Getting Started

### Prerequisites

- Python 3.8+
- [dbt Core](https://docs.getdbt.com/docs/core/installation) with the Snowflake adapter:

```bash
pip install dbt-snowflake
```

- A [Snowflake](https://signup.snowflake.com/) account (free trial works)

---

### Snowflake Setup

Run the following in a Snowflake worksheet to set up the required objects:

```sql
-- Create a dedicated warehouse
CREATE WAREHOUSE netflix_wh
  WITH WAREHOUSE_SIZE = 'X-SMALL'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE;

-- Create database and schemas
CREATE DATABASE netflix_db;

CREATE SCHEMA netflix_db.raw;       -- Raw source data
CREATE SCHEMA netflix_db.staging;   -- dbt staging models
CREATE SCHEMA netflix_db.marts;     -- dbt dim / fct models

-- Create a dbt service user
CREATE USER dbt_user
  PASSWORD = '<your_password>'
  DEFAULT_WAREHOUSE = netflix_wh
  DEFAULT_ROLE = dbt_role;

CREATE ROLE dbt_role;
GRANT USAGE ON WAREHOUSE netflix_wh TO ROLE dbt_role;
GRANT ALL ON DATABASE netflix_db TO ROLE dbt_role;
GRANT ROLE dbt_role TO USER dbt_user;
```

Then load the raw Netflix CSV into `netflix_db.raw`:

```sql
CREATE OR REPLACE TABLE netflix_db.raw.netflix_titles (
  show_id       VARCHAR,
  type          VARCHAR,
  title         VARCHAR,
  director      VARCHAR,
  cast          VARCHAR,
  country       VARCHAR,
  date_added    VARCHAR,
  release_year  NUMBER,
  rating        VARCHAR,
  duration      VARCHAR,
  listed_in     VARCHAR,
  description   VARCHAR
);

-- Load from a stage or use a COPY INTO from S3
```

---

### dbt Profile Configuration

Add a `netflix` profile to your `~/.dbt/profiles.yml`:

```yaml
netflix:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: <your_account_identifier>     # e.g. xy12345.us-east-1
      user: dbt_user
      password: <your_password>              # or use key-pair / SSO
      role: dbt_role
      warehouse: netflix_wh
      database: netflix_db
      schema: marts
      threads: 4
      client_session_keep_alive: false
```

> 💡 Use environment variables to avoid hardcoding credentials:
> ```yaml
> password: "{{ env_var('DBT_SNOWFLAKE_PASSWORD') }}"
> ```

---

### Install Dependencies

```bash
# Clone the repo
git clone https://github.com/devam1402/netflixdbt-snowflake.git
cd netflixdbt-snowflake

# Install dbt packages (e.g. dbt_utils)
dbt deps

# Verify your Snowflake connection
dbt debug
```

---

## Running the Project

**Run all models:**

```bash
dbt run
```

**Run a specific model:**

```bash
dbt run --select dim_titles
```

**Run models by layer:**

```bash
dbt run --select staging.*      # All staging models
dbt run --select dim.*          # All dimension tables
dbt run --select fct.*          # All fact tables
```

**Run with full refresh (re-build tables from scratch):**

```bash
dbt run --full-refresh
```

**Compile SQL without executing:**

```bash
dbt compile
```

**Generate and serve the docs site:**

```bash
dbt docs generate
dbt docs serve
```

---

## Testing

dbt tests validate data quality at every layer.

**Run all tests:**

```bash
dbt test
```

**Run tests for a specific model:**

```bash
dbt test --select dim_titles
```

**Common tests applied:**

| Test | Models | What it checks |
|---|---|---|
| `not_null` | All dim/fct models | Primary and foreign keys are never null |
| `unique` | `dim_titles.show_id` | No duplicate title records |
| `accepted_values` | `dim_titles.type` | Only `Movie` or `TV Show` |
| `relationships` | `fct_releases` | Every fact row has a matching dim record |

**Run tests + models together:**

```bash
dbt build
```

> `dbt build` runs seeds → snapshots → models → tests in the correct dependency order.

---

## Seeds

Seeds are small static CSV files loaded directly into Snowflake via dbt.

```bash
dbt seed
```

Seeds are stored in the `seeds/` directory. Example use cases:
- Mapping content rating codes to descriptions
- Country ISO code lookups
- Genre category normalizations

After loading, seeds are available as source tables in your models:

```sql
SELECT *
FROM {{ ref('ratings_lookup') }}
```

---

## Snapshots

Snapshots in the `snapshots/` directory capture historical changes to dimension records using **SCD Type 2**.

```bash
dbt snapshot
```

Example — tracking when a title's rating or category changes over time:

```sql
{% snapshot netflix_titles_snapshot %}

{{
  config(
    target_schema='snapshots',
    unique_key='show_id',
    strategy='check',
    check_cols=['rating', 'listed_in']
  )
}}

SELECT * FROM {{ source('raw', 'netflix_titles') }}

{% endsnapshot %}
```

---

## Analyses

The `analyses/` directory contains exploratory SQL queries that are compiled by dbt but not materialized as tables.

Example analyses included:
- Top 10 most prolific directors on Netflix
- Content release trend by year (Movies vs TV Shows)
- Countries with the most Netflix originals
- Average duration by genre

Run with:

```bash
dbt compile --select analyses.*
```

Compiled SQL is written to `target/compiled/netflix/analyses/`.

---

## Macros

Reusable Jinja2 macros are stored in `macros/`. These eliminate repetition across models.

Example — a macro to safely split comma-separated genres:

```sql
{% macro split_column(column_name, delimiter=',') %}
  TRIM(value)
FROM TABLE(FLATTEN(INPUT => SPLIT({{ column_name }}, '{{ delimiter }}')))
{% endmacro %}
```

Usage in a model:

```sql
SELECT
  show_id,
  {{ split_column('listed_in') }} AS genre
FROM {{ ref('stg_netflix__titles') }}
```

---

## Contributing

Contributions are welcome!

1. Fork the repository
2. Create a feature branch: `git checkout -b feat/new-model`
3. Add your model/test/macro with proper documentation
4. Run `dbt build` to verify everything passes
5. Open a Pull Request with a clear description

**Model documentation standard** — every model should have a `.yml` schema file:

```yaml
models:
  - name: dim_titles
    description: "Cleaned and deduplicated Netflix title dimension."
    columns:
      - name: show_id
        description: "Unique identifier for each title."
        tests:
          - unique
          - not_null
```

---

## Resources

- [dbt Documentation](https://docs.getdbt.com/)
- [dbt Discourse Community](https://discourse.getdbt.com/)
- [Snowflake dbt Adapter Docs](https://docs.getdbt.com/docs/core/connect-data-platform/snowflake-setup)
- [dbt Utils Package](https://github.com/dbt-labs/dbt-utils)
- [Netflix Titles Dataset (Kaggle)](https://www.kaggle.com/datasets/shivamb/netflix-shows)
- https://www.youtube.com/watch?v=zZVQluYDwYY by darshil parmar.

---

<div align="center">

Built with ❤️ by [devam1402](https://github.com/devam1402)

</div>
