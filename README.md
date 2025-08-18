Marketing common data modelling challenge — README

Quick links

Looker Studio report:[ https://lookerstudio.google.com/reporting/5947fbbe-9611-43b2-9f54-9cc30138036b](https://lookerstudio.google.com/reporting/5218607d-06b9-4c8b-a353-8acc5dea73fe)

Project overview

This repo standardizes marketing/ad-platform data into a single unified model so analysts and marketers can answer questions like “Where do clicks perform better — Facebook or TikTok?”.

The canonical table used by Looker Studio is ads_basic_performance (built in BigQuery via dbt). Raw platform CSVs live in seeds/ and are transformed with staging and marts models.

Repo layout (high level)

seeds/ — raw CSVs (one file per source / platform). These are loaded using dbt seed.

models/staging/ — platform-specific staging models (stg_src_<platform>.sql) that standardize column names and types.

models/marts/ — business-level marts and the main ads_basic_performance.sql which joins/unifies staging tables.

models/macros/, tests/, etc. — helpers and tests.

Prerequisites

GCP project with BigQuery and permissions to create datasets/tables

dbt (compatible version for this repo)

Access to the Looker Studio report (viewer/editor as needed)

Optionally, credentials or API access to pull CSVs directly from source platforms

Adding new data (step-by-step)

Prepare CSV

Name convention: src_<platform>.csv (e.g. src_facebook.csv, src_tiktok.csv). Keep the file in seeds/.

Include raw columns as exported by the platform. Don’t worry about renaming — staging models will remap.

Load seed into dbt

Run: dbt seed --select src_<platform> or dbt seed to load all seeds.

Create a staging model

Add models/staging/stg_src_<platform>.sql using an existing staging file as a template.

Standardize column names to the canonical fields used across platforms (see canonical schema below).

Cast types explicitly and add dbt_utils tests for non-null / accepted ranges where appropriate.

Integrate into the mart

Update models/marts/ads_basic_performance.sql to include the new staging model: map fields from stg_src_<platform> into the unified schema and union with other platforms.

Run models and validate

Run: dbt run --models marts.ads_basic_performance.

Run tests: dbt test --models marts.ads_basic_performance.

Open Looker Studio and validate metrics & filters.

Canonical schema (recommended columns)

Use these canonical column names in your staging models so the mart can union them easily:

event_date (DATE or TIMESTAMP normalized to UTC)

platform (e.g. Facebook, TikTok)

campaign_id, campaign_name

adgroup_id, adgroup_name

ad_id, ad_name

impressions (INT)

clicks (INT)

engagements (INT) — optional, platform-specific

conversions (INT)

spend (NUMERIC / FLOAT)

revenue (NUMERIC) — if available

currency (STRING) — optional

Key metric definitions (use these formulas in SQL)

Cost per engagement (CPE) = SUM(spend) / NULLIF(SUM(engagements), 0)

Conversion cost (Cost per conversion / CPA) = SUM(spend) / NULLIF(SUM(conversions), 0)

CPC (cost per click) = SUM(spend) / NULLIF(SUM(clicks), 0)

Impressions by channel = SUM(impressions) FILTER (WHERE platform = 'Facebook') (or grouped by platform)

Use NULLIF(..., 0) to avoid division-by-zero errors.

Example SQL snippet (in ads_basic_performance.sql)

select
  platform,
  event_date,
  sum(spend) as spend,
  sum(clicks) as clicks,
  sum(engagements) as engagements,
  sum(conversions) as conversions,
  sum(spend) / nullif(sum(clicks),0) as cpc,
  sum(spend) / nullif(sum(engagements),0) as cpe,
  sum(spend) / nullif(sum(conversions),0) as cost_per_conversion
from {{ ref('stg_src_unified') }}
group by 1,2

Staging template (example)

Create models/staging/stg_src_example.sql using the following pattern:

with raw as (
  select * from {{ ref('src_example') }} -- seed name (dbt seed creates a relation named like the seed)
)

select
  cast(event_date as date) as event_date,
  'Example' as platform,
  cast(campaign_id as string) as campaign_id,
  campaign_name,
  cast(adgroup_id as string) as adgroup_id,
  cast(ad_id as string) as ad_id,
  cast(impressions as int64) as impressions,
  cast(clicks as int64) as clicks,
  cast(engagements as int64) as engagements,
  cast(conversions as int64) as conversions,
  cast(spend as numeric) as spend
from raw

Note: if your seed file name is src_example.csv, dbt will create a relation named src_example when you run dbt seed.

Common errors & troubleshooting

Compilation Error: depends on a node named 'stg_src_facebook' — means a model references a staging model that doesn't exist or the seed name is different. Fix by:

Confirm seeds/ contains src_facebook.csv.

Confirm models/staging/stg_src_facebook.sql exists and is named correctly.

Run dbt seed to ensure the seed relation exists.

Division by zero in metrics — use NULLIF(sum(clicks), 0).

Mismatched field types — cast explicitly in staging models.

Best practices

Keep seeds as raw exports and do transformations in staging models.

Use canonical column names across platforms for easy unioning.

Add dbt tests for critical fields (not null, unique keys where applicable, accepted ranges).

Document each staging model with description in the model file header so dbt docs is useful.

Prefer incremental models for very large historical seeds and set a clear partition_by on event_date.

Keep timezone normalization consistent (store event_date in UTC or clearly document timezone).

Looker Studio / BigQuery

After dbt run, confirm the ads_basic_performance table exists in BigQuery (dataset and table names depend on dbt_project.yml profiles).

Point Looker Studio to that BigQuery table and validate the fields and calculated metrics.

Add Looker Studio calculated fields only for ad hoc reporting — prefer to keep canonical metrics in BigQuery.

Example commands
# load CSVs
dbt seed --select src_facebook src_tiktok

# build the mart
dbt run --models marts.ads_basic_performance

# run tests
dbt test --models marts.ads_basic_performance

# generate docs
dbt docs generate && dbt docs serve
