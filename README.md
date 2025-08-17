Marketing Common Data Model (MCDM) Challenge
This dbt project is a solution to the Marketing Common Data Model Challenge. The primary goal is to ingest raw advertising data from multiple platforms (Facebook, Bing, TikTok, Twitter), transform and unify it into a single, standardized model, and use this model to power a marketing performance dashboard.

🚀 Live Dashboard
You can view the recreated marketing performance dashboard here:

[Link to your Looker Studio Dashboard] (<- Replace with your dashboard link)

✨ Features
Data Unification: Combines data from disparate ad platforms into a single source of truth.

Standardized Metrics: Calculates key performance indicators (KPIs) like CPC, Conversion Cost, and Cost Per Engagement consistently across all channels.

Scalability: Designed to be easily extended with new data sources.

Data-Driven Insights: Powers a dashboard to answer questions like, "Which channel has a better click-through rate?" or "What is the overall conversion cost?"

🛠️ Tech Stack
Data Transformation: dbt (Data Build Tool)

Data Warehouse: Google BigQuery

Data Visualization: Google Looker Studio

📂 Repository Structure
.
├── dbt_project.yml       # dbt project configuration
├── models                # Contains the dbt models
│   └── ads_basic_performance.sql  # The final unified data model
├── seeds                 # Raw data files (CSVs)
│   ├── src_ads_bing_all_data.csv
│   ├── src_ads_creative_facebook_all_data.csv
│   ├── src_ads_tiktok_ads_all_data.csv
│   └── src_promoted_tweets_twitter_all_data.csv
└── README.md             # This file

⚙️ Setup and Usage
To get this project running in your own dbt Cloud environment:

Clone the Repository: Clone this repository and connect it to a new dbt Cloud project.

Configure BigQuery Connection: Set up a connection to your Google BigQuery instance in dbt Cloud.

Load Raw Data: Run the dbt seed command to load the raw CSV data from the /seeds directory into your BigQuery project.

dbt seed

Run the Model: Execute the dbt run command to transform the raw data and create the final ads_basic_performance table.

dbt run

Connect to Looker Studio: Connect Google Looker Studio to your BigQuery project and use the ads_basic_performance table as the data source for your dashboard.

🔌 How to Add a New Ad Platform
This model is designed for easy extension. Follow these steps to add a new data source:

Add Raw Data: Place the new platform's raw data as a .csv file in the /seeds directory. For example, src_ads_linkedin_all_data.csv.

Update the Model: Open the models/ads_basic_performance.sql file and add a new Common Table Expression (CTE) to read, cast, and rename the columns from your new seed file to match the MCDM structure.

-- Example CTE for a new platform
, linkedin as (
    select
        cast(ad_id as string) as ad_id,
        cast(ad_group_id as string) as adset_id,
        cast(campaign_id as string) as campaign_id,
        channel,
        cast(clicks as int64) as clicks,
        cast(date as date) as date,
        cast(impressions as int64) as impressions,
        cast(cost as float64) as spend,
        cast(conversions as int64) as total_conversions,
        -- Use `null` for fields that don't exist in the source
        null as video_views
    from {{ ref('src_ads_linkedin_all_data') }}
)

Union the Data: Add your new CTE to the final unioned_data CTE using UNION ALL.

-- In the unioned_data CTE
...
union all
select * from twitter
union all
select * from linkedin -- Add the new CTE here

Re-run dbt: Run dbt seed and dbt run to incorporate the new data into your final table. Your dashboard will update automatically.
