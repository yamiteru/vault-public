Maybe I could first create an open-source self-hosted version with limited features and then build SaaS on top of that.

This way I could create a fan-base first and ask for money for a more advanced version later.

# Features

- Track page visits
- Track redirect urls
- Track website/service status
- Track website/service performance
- Track (custom) events
- Create custom dashboards
- Create custom notifications
- Daily/weekly/monthly reports
- GDPR/etc compliant
- Hosted in Europe
- No cookies or JS
- Self-hosted or SaaS or both
- Usage based pricing
- Public API and webhooks
- Public website/service status page

# Stack

## Shared

Repo: Github
CI/CD: Github
Payments: Paddle

## Backend

Events: ClickHouse
Database: PostgreSQL
Server: Oracle Arm Compute
Mailing: Plunk
Language: Go

## Frontend

Framework: SvelteKit

# Event types

## Visit

Uses tracking pixel.

## Custom

Each event can be connected to `user_id`.

Several pre-defined types of events:

- 1D (`number`) 
- 2D (`[number, number]`)
- ...

Events can be created but it has to create a new database.

## Link

# Pricing

- Pay as you go
- Unlimited projects
- Unlimited users
- (Count * Type) * Retention
    - Count = how many events have been logged
    - Type = X for pre-defined, Y for custom event
    - Retention = how long to keep the data for
- Tiny free plan
    - 1000 events
    - 1 month retention
    - We want to be careful
- Pipe data to a different platform
    - Sends a webhook every hour
    - Retention is fixed to one hour
    - Pay only for events
