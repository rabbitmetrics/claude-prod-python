# Production Python Prompts

Three one-line use cases for building production pipelines with Claude Code.

## 1. Data Pipeline

Calculate customer lifetime metrics from Shopify orders: total orders, total spent, average order value, and days since first purchase. Read from data/orders.csv.

## 2. ML Pipeline

Predict which users will generate revenue based on behavioral events. Read users from data/users.csv and events from data/events.csv.

## 3. AI Pipeline

Standardize messy campaign names into structured fields using Claude as an AI pipeline. Use a generate.py stage that calls the Anthropic API to parse each campaign name into channel, campaign_type, season, and version. Read from data/campaigns.csv.
