# Agents.md

This file documents the travel analytics data model and business logic to make future analysis consistent and fast.

## Business context
- Business structure: multiple companies/brands called series (d1, d2, m1, etc.).
- Each series has multiple verticals: flight, car, hotel, etc.
- Monetization is affiliate based with three service types:
  - api: apps using Kayak/Skyscanner APIs to show results
  - deeplink: direct deeplink to Kayak/Skyscanner sites
  - ads: Kayak ads
- Session definition: a user session is created when a user performs an exit click on any platform.
- Platforms: web, android, ios.

## Database scope
All data in this repo refers to ClickHouse database `travel`.
Data sources include:
- First-party backend (sessions, users, conversions, reports)
- GA4 (analytics reports)
- Google Ads (campaign/ad performance + click view)
- Meta Ads (campaign/ad/ad set/creative + insights)
- App Store (commerce, engagement, usage)
- Google Play (installs, ratings, crashes, store performance)

## Table naming conventions
- `travel___*`: first-party backend tables
- `ga4_*` and `google_analytics___*`: GA4 extracts and reports
- `google___*`, `google_*`: Google Ads extracts
- `fb___*`: Meta Ads extracts
- `app_store___*`: Apple App Store analytics
- `google_play___*`: Google Play Console analytics
- `*_staging___*`: staging tables from ingestion
- `_dlt_*` / `*___dlt_*`: ingestion metadata (DLT)
- `*_mv`: materialized views

## First-party core tables (travel___*)
These are the primary sources for session, attribution, and revenue logic.

### travel___users_usersession
Core session table. Each row represents an exit click session.
Key columns:
- id (Int64)
- created_at (DateTime64)
- user_id (Int64)
- vertical (String) - flight/car/hotel/etc
- route (String) - often origin-destination or route token
- country_code (String)
- service_type (String) - api/deeplink/ads
- gclid, fbclid, msclkid, ttclid, twclid (String) - ad click ids
- params__* (search parameters such as dates, locations, passengers)

### travel___users_appuser
Application user entity.
Key columns:
- id (Int64)
- user_id (Int64)
- app_id (Int64)
- vendor_id, pseudo_id (String)
- created_at, updated_at (DateTime64)
- email, name, email_consent
- acquired_route, referrer_url

### travel___users_deviceuser
Device identity map.
Key columns:
- id (Int64)
- device_id, device_id_type (String)
- created_at (DateTime64)

### travel___users_attribution
Session-level ad attribution info.
Key columns:
- id (Int64)
- session_id (Int64)
- ad_id, adgroup_id, campaign_id, campaign_group_id, account_id
- ad_objective_name
- created_at (DateTime64)

### travel___users_conversions
Conversion events mapped to sessions and reports.
Key columns:
- id (Int64)
- user_session_id (Int64)
- report_id (Int64)
- revenue (Decimal)
- created_at, updated_at (DateTime64)
- in_ga4_at, in_meta_at, in_g_ads_at, in_bing_at (DateTime64)

### travel___reports_report
Affiliate revenue reports.
Key columns:
- id (Int64)
- date (Date)
- click_time (DateTime64)
- revenue (Decimal)
- currency (String)
- provider_service_id (Int64)
- tracking_param (String)
- is_sale (Bool)
- country_code (String)

### travel___reports_domain
Domain metadata.
Key columns:
- id (Int64)
- domain (String)
- created_at, updated_at (DateTime64)
- expiry_date (Date)

### travel___users_provider / travel___users_providerservice
Affiliate provider catalog and service mapping.
Key columns:
- provider: id, name
- providerservice: id, provider_id, service, web_conversion_name, app_conversion_name

### travel___users_application
Application metadata (brand/app/series mapping).
Key columns:
- id (Int64)
- managing_system (String) - use for series (d1/d2/m1)
- platform (String) - web/android/ios
- vertical (String)
- package_name_or_url (String)
- fb_account_id, fb_decryption_key

### travel___inline_ad_logs
Inline ad logs (Kayak ads).
Key columns:
- id (String)
- created_at (DateTime64)
- product_type, site, headline, description
- cpc_price, cpc_currency
- origin, destination (and *_v_text)
- travelers, cabin_class, start_date, end_date
- user_id, os

### Enriched views
- travel___user_sessions_enriched: session + app/brand/series/route/context
- travel___conversions_enriched: conversion + revenue + provider + channel exposure
- *_mv: materialized views (read-only in analysis)

## External sources

### GA4 (ga4_* and google_analytics___*)
GA4 extracts are split into report tables with consistent metric columns.
Common fields: property_id, date, platform, startdate/enddate, metrics.

Key tables and columns:
- ga4_events_report: eventname, eventcount, totalusers, totalrevenue
- ga4_conversions_report: eventname, conversions, totalusers, totalrevenue
- ga4_pages / ga4_pages_path_report: pagepath, screenpageviews, totalusers, conversions
- ga4_traffic_sources: sessionsource, sessionmedium, sessions, totalusers
- ga4_locations: country, region, city, sessions, screenpageviews

Raw GA4 exports:
- google_analytics___ga4_events: event_name, event_count, total_users
- google_analytics___ga4_session_traffic_sources: session_source, session_medium, session_campaign
- google_analytics___ga4_traffic_sources: source, medium, campaign
- google_analytics___ga4_device_category: device_category, sessions, conversions
- google_analytics___ga4_user_engagement: sessions, engagement_rate, average_session_duration
- google_analytics___ga4_users: active_users, new_vs_returning

### Google Ads (google___*, google_*)
Performance and entity metadata tables.
Key performance tables:
- google___google_ads: date, account_id, campaign_id, ad_group_id, ad_id, cost_micros, impressions, clicks, conversions
- google___google_ads_conversion_actions: conversion_action, conversions, conversions_value
- google_click_view: click_view_gclid, campaign_id, ad_group_id, segments_date
Entity tables:
- google___google_ads_campaigns: campaign_id, campaign_name, bidding_strategy_type, daily_budget
- google_ad_group / google_ad_group_ad: ad group and ad metadata (wide tables)
- google_customer: customer account info

### Meta Ads (fb___*)
Campaign, ad set, ad, creative, and insights tables.
Key tables:
- fb___ads: id, adset_id, campaign_id, creative__id, status, created_time
- fb___ad_sets: id, campaign_id, optimization_goal, billing_event, budgets
- fb___ad_creatives: id, name, call_to_action_type, object_story_spec__*
- fb___insights and fb___insights__*: performance metrics and breakdowns

### App Store (app_store___*)
App Store analytics (all measures stored as strings, cast as needed).
Key tables:
- app_store___app_store_commerce: date, download_type, source_type, page_type, territory, counts
- app_store___app_store_engagement: event, engagement_type, counts, unique_counts
- app_store___app_usage: event, installs, sessions, total_session_duration, crashes, unique_devices

### Google Play (google_play___*)
Google Play Console analytics.
Key tables:
- google_play___google_play_installs: daily_user_installs, daily_user_uninstalls, active_device_installs
- google_play___google_play_ratings: daily_average_rating, total_average_rating
- google_play___google_play_crashes: daily_crashes, daily_anrs
- google_play___google_play_store_performance: store_listing_acquisitions, conversion_rate, utm_source

## Key business logic mapping
- Series/brand: use `travel___users_application.managing_system` to map to d1/d2/m1, etc.
- Platform: use `travel___users_application.platform` and GA4 platform fields.
- Vertical: use `travel___users_usersession.vertical` and `travel___users_application.vertical`.
- Service type (api/deeplink/ads): use `travel___users_usersession.service_type`.
- Session definition: each row in `travel___users_usersession` is an exit click session.
- Revenue: use `travel___reports_report.revenue` (affiliate reports) and `travel___users_conversions.revenue`.

## Recommended joins
- Session to attribution:
  - travel___users_usersession.id = travel___users_attribution.session_id
- Session to conversion:
  - travel___users_usersession.id = travel___users_conversions.user_session_id
- Conversion to report:
  - travel___users_conversions.report_id = travel___reports_report.id
- User to app metadata:
  - travel___users_appuser.app_id = travel___users_application.id
  - travel___users_usersession.user_id = travel___users_appuser.user_id
- Series/brand mapping:
  - travel___users_application.managing_system -> series key (d1/d2/m1)

## Metrics glossary
- Session: one row in `travel___users_usersession` (exit click event).
- User: `travel___users_appuser.user_id` (can map to app/series via `app_id`).
- Conversion: one row in `travel___users_conversions` mapped to a session.
- Revenue: affiliate revenue from `travel___reports_report.revenue` (via `report_id`).
- Conversion rate: conversions / sessions (join sessions to conversions).
- Revenue per session: revenue / sessions (join sessions to conversions and reports).
- Series: `travel___users_application.managing_system`.
- Platform: `travel___users_application.platform` (web/android/ios).
- Vertical: `travel___users_usersession.vertical` or `travel___users_application.vertical`.
- Service type: `travel___users_usersession.service_type` (api/deeplink/ads).
- Paid click ids: gclid (Google), fbclid (Meta), msclkid (Microsoft), ttclid (TikTok), twclid (Twitter/X).

## Sample queries

### 1) Cohort revenue by series and vertical (user cohort month)
```sql
WITH users AS (
  SELECT
    user_id,
    toStartOfMonth(created_at) AS cohort_month,
    app_id
  FROM travel.travel___users_appuser
),
sessions AS (
  SELECT
    id AS session_id,
    user_id,
    vertical
  FROM travel.travel___users_usersession
)
SELECT
  u.cohort_month,
  a.managing_system AS series,
  s.vertical,
  countDistinct(s.session_id) AS sessions,
  countDistinct(c.id) AS conversions,
  sum(r.revenue) AS revenue
FROM users u
JOIN sessions s ON s.user_id = u.user_id
LEFT JOIN travel.travel___users_conversions c ON c.user_session_id = s.session_id
LEFT JOIN travel.travel___reports_report r ON r.id = c.report_id
LEFT JOIN travel.travel___users_application a ON a.id = u.app_id
GROUP BY u.cohort_month, series, s.vertical
ORDER BY u.cohort_month DESC, revenue DESC;
```

### 2) Revenue per session by service type and platform
```sql
SELECT
  a.managing_system AS series,
  a.platform,
  s.service_type,
  countDistinct(s.id) AS sessions,
  countDistinct(c.id) AS conversions,
  sum(r.revenue) AS revenue,
  if(sessions = 0, 0, revenue / sessions) AS revenue_per_session
FROM travel.travel___users_usersession s
LEFT JOIN travel.travel___users_conversions c ON c.user_session_id = s.id
LEFT JOIN travel.travel___reports_report r ON r.id = c.report_id
LEFT JOIN travel.travel___users_appuser u ON u.user_id = s.user_id
LEFT JOIN travel.travel___users_application a ON a.id = u.app_id
GROUP BY series, a.platform, s.service_type
ORDER BY revenue DESC;
```

### 3) Paid vs organic session mix by platform
```sql
SELECT
  a.platform,
  multiIf(
    s.gclid IS NOT NULL, 'google_ads',
    s.fbclid IS NOT NULL, 'meta_ads',
    s.msclkid IS NOT NULL, 'microsoft_ads',
    s.ttclid IS NOT NULL, 'tiktok_ads',
    s.twclid IS NOT NULL, 'twitter_ads',
    'organic_or_unknown'
  ) AS channel,
  countDistinct(s.id) AS sessions
FROM travel.travel___users_usersession s
LEFT JOIN travel.travel___users_appuser u ON u.user_id = s.user_id
LEFT JOIN travel.travel___users_application a ON a.id = u.app_id
GROUP BY a.platform, channel
ORDER BY sessions DESC;
```

## Data quality notes
- App Store metrics are stored as strings and need casting (toInt64OrZero / toFloat64OrZero).
- GA4 report tables store date as String; GA4 raw exports store DateTime64.
- Some datasets include staging copies; prefer non-staging tables unless validating ingestion.

## Quick table inventory (largest first)
- travel___users_usersession
- travel___users_appuser
- travel___inline_ad_logs
- travel___users_deviceuser
- google_play___google_play_installs
- travel___reports_report
- travel___user_sessions_enriched
- travel___users_conversions
- travel___users_attribution
- app_store___app_store_engagement

If you add new sources or fields, update this file with the business meaning, primary keys, and recommended joins.
