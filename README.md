Instructions for Adding New Ad Platforms to MCDM
This document provides step-by-step instructions for integrating data from new advertising platforms into the existing Marketing Common Data Modeling (MCDM) structure.
Overview
The current MCDM implementation supports four advertising platforms:

Facebook (Meta Ads)
TikTok Ads
Twitter (Promoted Tweets)
Bing Ads

All platforms are unified into a single ads_basic_performance table with standardized fields and calculated metrics.
Step-by-Step Process for Adding New Platforms
1. Create the Raw Data Seed
First, add your new platform's raw data as a dbt seed:
sql-- seeds/src_ads_[platform_name]_all_data.csv
-- Upload your raw CSV data to the seeds folder
2. Create Platform-Specific Transformation Model
Create a new model file: models/[platform_name]_data.sql
sql-- models/[platform_name]_data.sql
WITH rename_[platform_name] AS (
    SELECT
        -- Map raw fields to standardized MCDM fields
        raw_field_1 AS ad_id,
        raw_field_2 AS adset_id,
        -- ... continue mapping
    FROM {{ ref('src_ads_[platform_name]_all_data') }}
),
[platform_name]_data AS (
    SELECT
        -- Required MCDM fields (see field mapping section below)
        insert_date,
        ad_id,
        adset_id,
        campaign_id,
        channel,
        creative_title,
        clicks,
        date,
        impressions,
        revenue,
        spend,
        conversions,
        engagement,
        -- Calculated metrics
        ROUND(spend / NULLIF(engagement, 0), 2) AS engagement_cost,
        ROUND(spend / NULLIF(conversions, 0), 2) AS conversion_cost,
        ROUND(spend / NULLIF(clicks, 0), 2) AS cpc,
        ROUND((clicks / NULLIF(impressions, 0)) * 100, 2) AS ctr,
        ROUND(spend / NULLIF(impressions, 0), 2) AS cpi,
        ROUND((engagement / NULLIF(impressions, 0)) * 100, 2) AS engagement_rate,
        video_views
    FROM rename_[platform_name]
)
SELECT * FROM [platform_name]_data
3. Update the Main MCDM Model
Add your new platform to the ads_basic_performance model:
sql-- models/ads_basic_performance.sql
-- Add this UNION ALL block to the existing model:

UNION ALL
SELECT
    CAST(ad_id AS STRING) AS ad_id,
    CAST(adset_id AS STRING) AS adset_id,
    CAST(campaign_id AS STRING) AS campaign_id,
    CAST(channel AS STRING) AS channel,
    CAST(creative_title AS STRING) AS creative_title,
    CAST(clicks AS INT64) AS clicks,
    date,
    CAST(impressions AS INT64) AS impressions,
    CAST(revenue AS INT64) AS revenue,
    CAST(spend AS INT64) AS spend,
    CAST(conversions AS INT64) AS conversions,
    CAST(engagement AS INT64) AS engagement,
    CAST(video_views AS INT64) AS video_views
FROM {{ ref('[platform_name]_data') }}
Field Mapping Guidelines
Required MCDM Fields
MCDM FieldDescriptionData TypeNotesad_idUnique ad identifierSTRINGUse platform's ad IDadset_idAd set/group identifierSTRINGMay be NULL for some platformscampaign_idCampaign identifierSTRINGRequiredchannelPlatform nameSTRINGe.g., 'Facebook', 'TikTok'creative_titleAd creative title/textSTRINGUse main ad textclicksNumber of clicksINT64RequireddateDate of performanceDATERequiredimpressionsNumber of impressionsINT64RequiredrevenueRevenue generatedINT64May be NULLspendAmount spentINT64RequiredconversionsNumber of conversionsINT64Platform-specific definitionengagementTotal engagementsINT64Platform-specific calculationvideo_viewsVideo view countINT64May be NULL
Platform-Specific Mapping Examples
Engagement Field Logic

Facebook: likes + comments + inline_link_clicks + shares + views
TikTok: rt_installs + registrations (or define based on available engagement metrics)
Twitter: Use existing engagements field
Bing: Set to 0 (no engagement metrics available)

Conversions Field Logic

Facebook: complete_registration + mobile_app_install + purchase + purchase_value + inline_link_clicks
TikTok: conversions + skan_conversion
Twitter: Set to NULL (not available)
Bing: Use existing conv field

Handling Missing Fields
When a platform doesn't provide certain MCDM fields:

Set to NULL: For optional fields like revenue, video_views
Set to 0: For numeric fields where zero makes sense (like engagement in Bing)
Calculate if possible: Derive from available fields when logical
Document assumptions: Always document your mapping decisions

Calculated Metrics
The following metrics are automatically calculated in each platform model:

Cost per Engagement: spend / engagement
Conversion Cost: spend / conversions
Cost per Click (CPC): spend / clicks
Click-through Rate (CTR): (clicks / impressions) * 100
Cost per Impression (CPI): spend / impressions
Engagement Rate: (engagement / impressions) * 100

Use NULLIF() to avoid division by zero errors.
Testing Your Implementation

Run seeds: dbt seed to load your raw data
Test individual model: dbt run --select [platform_name]_data
Test full MCDM: dbt run --select ads_basic_performance
Validate data: Compare key metrics with your platform's native reporting
Check dashboard: Verify new platform appears correctly in Looker Studio

Data Quality Considerations

Date formats: Ensure consistent DATE format across platforms
Currency: Standardize currency (all platforms should use same currency)
Null handling: Be consistent with NULL vs 0 for missing values
Data types: Follow the casting patterns in the main model
Field validation: Verify field mappings match your platform's data structure

Common Troubleshooting

Missing data in dashboard: Check if channel name matches exactly
Incorrect metrics: Verify engagement and conversion field calculations
Type errors: Ensure proper data type casting in final UNION
Date parsing: Confirm date field is properly formatted as DATE type

Example Implementation Checklist

 Raw data added as seed file
 Platform-specific model created with proper field mapping
 Engagement logic defined based on available platform metrics
 Conversion logic defined based on platform capabilities
 All required MCDM fields mapped or set to appropriate defaults
 Calculated metrics implemented with NULL handling
 Platform added to main ads_basic_performance UNION
 Models run successfully without errors
 Data appears correctly in dashboard
 Metrics validated against platform reporting

By following these instructions, you can successfully integrate any new advertising platform into the existing MCDM structure while maintaining data consistency and dashboard functionality.
