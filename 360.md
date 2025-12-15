# D2C E-Commerce Customer 360° - dbt Models

## Übersicht

Diese dbt-Modelle erstellen einen **unified Customer 360° View** für D2C E-Commerce, 
der via **Hightouch** nach **Klaviyo** gesynct wird.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DATA SOURCES                                 │
├─────────────────┬─────────────────┬─────────────────────────────────┤
│    Shopify      │    Klaviyo      │           Zendesk               │
│  (Orders, Kunden)│ (Email Events)  │      (Support Tickets)          │
└────────┬────────┴────────┬────────┴──────────────┬──────────────────┘
         │                 │                       │
         ▼                 ▼                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      STAGING LAYER                                   │
│  stg_shopify__customers    stg_klaviyo__profiles   stg_zendesk__*   │
│  stg_shopify__orders       stg_klaviyo__events                      │
└─────────────────────────────────────────────────────────────────────┘
         │                 │                       │
         ▼                 ▼                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    INTERMEDIATE LAYER                                │
│  int_customer__order_metrics     (RFM, CLV, Purchase Patterns)      │
│  int_customer__email_engagement  (Open/Click Rates, Engagement)     │
│  int_customer__support_metrics   (Tickets, CSAT, Resolution)        │
└─────────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       MARTS LAYER                                    │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    dim_customers                             │   │
│   │            (Unified Customer 360° View)                      │   │
│   │                                                              │   │
│   │  • Profile Info        • Email Engagement                    │   │
│   │  • Purchase Metrics    • Support Metrics                     │   │
│   │  • RFM Scores          • Lifecycle Stage                     │   │
│   │  • Value Tier          • Engagement Score                    │   │
│   │  • Marketing Flags     • Segment Assignments                 │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│              ┌───────────────┼───────────────┐                      │
│              ▼               ▼               ▼                      │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐               │
│   │seg_winback   │ │seg_vip       │ │seg_at_risk   │  ...          │
│   │_candidates   │ │_customers    │ │_high_value   │               │
│   └──────────────┘ └──────────────┘ └──────────────┘               │
└─────────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      HIGHTOUCH                                       │
│                   (Reverse ETL)                                      │
│                                                                      │
│   Syncs:                                                            │
│   • dim_customers → Klaviyo Profiles (alle Properties)              │
│   • seg_* → Klaviyo Lists (für Flows & Campaigns)                   │
└─────────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       KLAVIYO                                        │
│                                                                      │
│   Flows:                                                            │
│   • Winback Flow (triggered by seg_winback_candidates)              │
│   • VIP Welcome Flow (triggered by seg_vip_customers)               │
│   • At-Risk Intervention (triggered by seg_high_value_at_risk)      │
│   • Post-Purchase Nurture (triggered by seg_repeat_purchase_*)      │
└─────────────────────────────────────────────────────────────────────┘
```

## Model Struktur

```
models/
├── staging/
│   ├── stg_shopify__customers.sql
│   ├── stg_shopify__orders.sql
│   ├── stg_klaviyo__profiles.sql
│   ├── stg_klaviyo__events.sql
│   ├── stg_zendesk__tickets.sql
│   └── stg_zendesk__users.sql
├── intermediate/
│   ├── int_customer__order_metrics.sql
│   ├── int_customer__email_engagement.sql
│   └── int_customer__support_metrics.sql
├── marts/
│   ├── dim_customers.sql              # ← Haupt-Model für Klaviyo
│   └── segments/
│       ├── seg_winback_candidates.sql
│       ├── seg_vip_customers.sql
│       ├── seg_repeat_purchase_candidates.sql
│       └── seg_high_value_at_risk.sql
└── schema.yml
```

## Hightouch Setup

### 1. Source: BigQuery verbinden

```
Project ID: your-gcp-project
Dataset: dbt_prod (oder euer Output-Schema)
Service Account: hightouch-sa@your-project.iam.gserviceaccount.com
```

### 2. Models auswählen

Hightouch kann direkt dbt Models referenzieren:
- Aktiviere "dbt Cloud" oder "dbt Core" Integration
- Verbinde euer Git Repo
- Models mit Tag `klaviyo_sync` werden automatisch erkannt

### 3. Syncs konfigurieren

#### Sync 1: Customer Profiles (dim_customers → Klaviyo Profiles)

| BigQuery Column | Klaviyo Field | Typ |
|-----------------|---------------|-----|
| email | Email | Identifier |
| first_name | First Name | Property |
| last_name | Last Name | Property |
| lifetime_revenue | LTV | Property |
| total_orders | Total Orders | Property |
| lifecycle_stage | Lifecycle Stage | Property |
| customer_value_tier | Value Tier | Property |
| engagement_score | Engagement Score | Property |
| rfm_segment | RFM Segment | Property |
| is_vip | Is VIP | Property |
| days_since_last_order | Days Since Last Order | Property |
| ... | ... | ... |

**Schedule:** Alle 6 Stunden oder bei dbt run completion

#### Sync 2-5: Segments (seg_* → Klaviyo Lists)

Für jeden Segment-Table:
1. Neuen Sync erstellen
2. Model: z.B. `seg_winback_candidates`
3. Destination: Klaviyo List
4. Mode: **Mirror** (Profiles werden automatisch hinzugefügt/entfernt)
5. Match on: Email

**Schedule:** Täglich oder alle 12 Stunden

## Klaviyo Flows Setup

### Winback Flow
- **Trigger:** Added to List "Winback Candidates"
- **Conditional Split:** By `winback_stage` property
  - early_winback → Soft reminder, no discount
  - mid_winback → 10% discount offer
  - late_winback → 15-20% discount + urgency

### VIP Flow
- **Trigger:** Added to List "VIP Customers"
- **Conditional Split:** By `vip_tier` property
  - platinum → Personal outreach, exclusive access
  - gold → Early access to sales
  - silver → Loyalty rewards reminder

### At-Risk Intervention
- **Trigger:** Added to List "High Value At Risk"
- **Conditional Split:** By `recommended_action` property
  - resolve_support_first → Delay, check if ticket resolved
  - service_recovery → Apology + compensation offer
  - incentive_offer → Win-back discount

## Key Metrics im dim_customers

| Metric | Beschreibung | Verwendung |
|--------|--------------|------------|
| `lifecycle_stage` | prospect → new → active → at_risk → lapsing → churned | Segmentierung |
| `customer_value_tier` | vip / high / medium / low / no_purchase | Priorisierung |
| `engagement_score` | 0-100 Score aus Purchase + Email + Support | Health Metric |
| `rfm_segment` | 3-stelliger RFM Code (z.B. "555" = Best) | Targeting |
| `is_winback_candidate` | Boolean Flag | Trigger für Flows |
| `is_high_value_at_risk` | Boolean Flag | Priority Alert |

## Voraussetzungen

### Data Sources in BigQuery
- Shopify-Daten (via Fivetran, Airbyte, oder Stitch)
- Klaviyo-Daten (via Fivetran oder Klaviyo's native BigQuery export)
- Zendesk-Daten (via Fivetran oder Airbyte)

### Packages (dbt_packages.yml)
```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.0.0", "<2.0.0"]
```

## Kosten-Vergleich

| Lösung | Jährliche Kosten |
|--------|------------------|
| **Diese Lösung** | |
| BigQuery (Scanning + Storage) | ~€500-1.000 |
| dbt Cloud (Team) | ~€1.200 |
| Hightouch (Starter) | ~€4.000 |
| Klaviyo (besteht bereits) | €0 zusätzlich |
| **Gesamt** | **~€5.500-6.200/Jahr** |
| | |
| **Salesforce Alternative** | |
| Sales Cloud | ~€20.000+ |
| Data Cloud | ~€60.000+ |
| Marketing Cloud | ~€15.000+ |
| Implementation | ~€50.000+ |
| **Gesamt** | **€145.000+/Jahr** |

## Next Steps

1. [ ] Sources in BigQuery verifizieren (Schema-Namen anpassen)
2. [ ] dbt Models deployen & testen
3. [ ] Hightouch Free Tier aktivieren
4. [ ] Ersten Sync (dim_customers) konfigurieren
5. [ ] In Klaviyo: Flow für ersten Segment bauen
6. [ ] PoC dem Chef präsentieren 🎯
