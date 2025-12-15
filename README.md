# MLC-direct Customer 360° CDP (Customer Data Platform) Konzept

## Übersicht

Konzept für einen **einheitlichen Customer 360° View** für die MLC-direct D2C E-Commerce-Plattform, inklusive **Reverse ETL** Synchronisation nach **Klaviyo**.

### Was ist Reverse ETL?

**Reverse ETL** ist eine Technologie, die als **Brücke zwischen Data Warehouse (BigQuery) und operativen Tools (z.B. Klaviyo, Salesforce, HubSpot)** fungiert:

- **Ohne Reverse ETL:** Kundendaten liegen isoliert im Data Warehouse und können nicht für Marketing genutzt werden
- **Mit Reverse ETL:** Kundensegmente und -eigenschaften werden automatisch in Marketing-Tools übertragen

Reverse ETL ermöglicht es, datengetriebene Kundenprofile und Segmente, die wir in BigQuery erstellen, direkt für personalisierte E-Mail-Kampagnen zu nutzen – ohne manuelle Exporte oder IT-Aufwand.

**Beispiel-Tools:** Hightouch, Census, Polytomic, Rudderstack

➡️ *Im weiteren Dokument wird Hightouch als Reverse ETL Tool verwendet.*

```
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                                      DATENQUELLEN                                              │
├───────────────┬───────────────┬───────────────┬───────────────┬───────────────────────────────┤
│   Shopify     │    Amazon     │   Shipments   │    Klaviyo    │           Zendesk             │
│ (Bestellungen,│ (Bestellungen,│ (DHL, DPD,    │ (Email Events)│      (Support Tickets)        │
│    Kunden)    │    Kunden)    │  Tracking)    │               │                               │
└───────┬───────┴───────┬───────┴───────┬───────┴───────┬───────┴───────────────┬───────────────┘
        │               │               │               │                       │
        ▼               ▼               ▼               ▼                       ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    STAGING LAYER                                               │
│  stg_shopify__customers    stg_amazon__customers    stg_shipments           stg_klaviyo__*    │
│  stg_shopify__orders       stg_amazon__orders                               stg_zendesk__*    │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
        │               │               │               │                       │
        ▼               ▼               ▼               ▼                       ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                                  INTERMEDIATE LAYER                                            │
│  int_customer__order_metrics      (RFM, CLV, Kaufmuster)                                      │
│  int_customer__shipment_metrics   (Lieferstatus, Tracking, Zustellquote)                      │
│  int_customer__email_engagement   (Öffnungs-/Klickraten, Engagement)                          │
│  int_customer__support_metrics    (Tickets, CSAT, Lösungszeit)                                │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     MARTS LAYER                                                │
│                                                                                                │
│   ┌───────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                              dim_customers                                             │   │
│   │                      (Einheitlicher Customer 360° View)                               │   │
│   │                                                                                        │   │
│   │  • Profil-Informationen    • Email Engagement      • Shipment-Metriken                │   │
│   │  • Kauf-Metriken           • Support-Metriken      • Lieferzufriedenheit              │   │
│   │  • RFM Scores              • Lifecycle Stage                                          │   │
│   │  • Value Tier              • Engagement Score                                         │   │
│   │  • Marketing Flags         • Segment-Zuweisungen                                      │   │
│   └───────────────────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                                       │
│        ┌───────────────────────────────┼───────────────────────────────┐                      │
│        ▼                               ▼                               ▼                      │
│   ┌──────────────┐            ┌──────────────┐                ┌──────────────┐               │
│   │  seg_*       │            │events_       │                │seg_at_risk   │  ...          │
│   │  (Segmente)  │            │shipments     │                │_high_value   │               │
│   └──────────────┘            └──────────────┘                └──────────────┘               │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     HIGHTOUCH                                                  │
│                                   (Reverse ETL)                                                │
│                                                                                                │
│   Syncs:                                                                                      │
│   • dim_customers → Klaviyo Profiles (alle Properties)                                        │
│   • seg_* → Klaviyo Lists (für Flows & Campaigns)                                             │
│   • events_shipments → Klaviyo Events (Package Shipped, Package Delivered)                    │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                                      KLAVIYO                                                   │
│                                                                                                │
│   Flows:                                                                                      │
│   • Winback Flow (ausgelöst durch seg_winback_candidates)                                     │
│   • VIP Welcome Flow (ausgelöst durch seg_vip_customers)                                      │
│   • Shipment Flows (ausgelöst durch Package Shipped / Delivered Events)                       │
│   • At-Risk Intervention (ausgelöst durch seg_high_value_at_risk)                             │
│   • Post-Purchase Nurture (ausgelöst durch seg_repeat_purchase_*)                             │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Erweiterbarkeit

Die Plattform ist **leicht erweiterbar** für neue Vertriebskanäle. Beim Hinzufügen neuer Shops (z.B. Temu, eBay, Otto, Kaufland) müssen lediglich:
1. Die Datenquelle an BigQuery angeschlossen werden (via DLT / Cloud Functions)
2. Entsprechende Staging-Modelle (`stg_[shop]__customers`, `stg_[shop]__orders`) erstellt werden

Die bestehende Logik in den Intermediate- und Marts-Layern übernimmt automatisch die Integration in den Customer 360° View.

## Klaviyo Flows Setup Beispiele

### Winback Flow (Idee: Kunden reaktivieren, die länger nicht gekauft haben)
- **Trigger:** Hinzugefügt zur Liste "Winback Candidates"
- **Conditional Split:** Nach `winback_stage` Property
  - early_winback → Sanfte Erinnerung, kein Rabatt
  - mid_winback → 10% Rabatt-Angebot
  - late_winback → 15-20% Rabatt + Dringlichkeit

### VIP Flow (Idee: Top-Kunden mit hohem CLV besonders wertschätzen)
- **Trigger:** Hinzugefügt zur Liste "VIP Customers"
- **Conditional Split:** Nach `vip_tier` Property
  - platinum → Persönliche Ansprache, exklusiver Zugang
  - gold → Früher Zugang zu Sales
  - silver → Erinnerung an Treueprogramm

### Cross-Channel Flow (Idee: Kunden belohnen, die über mehrere Kanäle kaufen - z.B. Shopify + Amazon)
- **Trigger:** Hinzugefügt zur Liste "Cross-Channel Customers"
- **Conditional Split:** Nach `channels_count` oder `purchase_channels` Property
  - 2 Kanäle → Danke-Mail + kleiner Bonus
  - 3+ Kanäle → Exklusiver Loyalty-Status + besondere Angebote

### Shipment Flows (Idee: Kunden proaktiv über Versandstatus informieren)

**Package Shipped Flow:**
- **Trigger:** Event "Package Shipped"
- Tracking-Link + "Dein Paket ist unterwegs"
- Cross-Sell: "Während du wartest - passende Produkte entdecken"
- Vorfreude aufbauen: Anwendungstipps für bestellte Produkte

**Package Delivered Flow:**
- **Trigger:** Event "Package Delivered"
- **Nach 3-5 Tagen:** Review Request ("Wie gefällt dir dein Produkt?")
- **Nach 7 Tagen:** Anwendungstipps ("So holst du das Beste aus deinem Produkt")
- **Nachkauf-Reminder starten:** Timer für Replenishment (z.B. nach 25 Tagen bei 30-Tage-Packung)

**Delivery Problem Flow:**
- **Trigger:** Event "Delivery Failed" oder "Delivery Delayed"
- Proaktive Entschuldigung bei Verspätung
- Support-Kontakt anbieten bei fehlgeschlagener Zustellung

### At-Risk Intervention (Idee: High Value Customers mit Zendesk Tickets die ungelöst sind)
- **Trigger:** Hinzugefügt zur Liste "High Value At Risk"
- **Conditional Split:** Nach `recommended_action` Property
  - resolve_support_first → Verzögerung, prüfen ob Ticket gelöst
  - service_recovery → Entschuldigung + Kompensationsangebot
  - incentive_offer → Rückgewinnungs-Rabatt

### Product Recommendation Flow (Idee: Cross-Sell/Upsell basierend auf Kaufhistorie)
- **Trigger:** Hinzugefügt zur Liste "Upsell Candidates"
- **Conditional Split:** Nach `last_purchased_category` oder `preferred_categories` Property

**Beispiele für Nahrungsergänzungsmittel (tetesept, goodvita, rund.um):**
  - Energie-Produkte gekauft → Empfehlung für Konzentrations-Produkte (Focus Plus Sticks)
  - Schlaf-Produkte gekauft → Cross-Sell zu Ruhe/Calm-Produkten (Reishi Kapseln)
  - Immun-Produkte gekauft → Upsell zu Komplett-Bundles
  - Einzelprodukte gekauft → Bundle-Builder Empfehlung mit Rabatt
  - Kapseln gekauft → Cross-Sell zu passenden Plus Sticks (Booster)
  - tetesept Käufer → Empfehlung für Premium-Linie (rund.um, goodvita)

## Wichtige Metriken im dim_customers

| Metrik | Beschreibung | Verwendung |
|--------|--------------|------------|
| `lifecycle_stage` | prospect → new → active → at_risk → lapsing → churned | Segmentierung |
| `customer_value_tier` | vip / high / medium / low / no_purchase | Priorisierung |
| `engagement_score` | 0-100 Score aus Kauf + Email + Support | Health Metric |
| `rfm_segment` | 3-stelliger RFM Code (z.B. "555" = Beste) | Targeting |
| `is_winback_candidate` | Boolean Flag | Trigger für Flows |
| `is_high_value_at_risk` | Boolean Flag | Prioritäts-Alarm |
| `preferred_categories` | Meist gekaufte Produktkategorien | Cross-Sell Targeting |
| `avg_order_value` | Durchschnittlicher Bestellwert | Upsell-Potenzial |
| `cross_sell_score` | 0-100 Score für Cross-Sell Wahrscheinlichkeit | Priorisierung |
| `avg_delivery_time` | Durchschnittliche Lieferzeit | Service-Qualität |
| `delivery_success_rate` | Zustellquote | Risiko-Indikator |

## Voraussetzungen

### Datenquellen in BigQuery
- ✅ Shopify-Daten (bereits über bestehende Pipeline synchronisiert)
- ✅ Amazon-Daten (bereits über bestehende Pipeline synchronisiert)
- ✅ Shipment-Daten (bereits in BigQuery - DHL, DPD, etc.)
- ⏳ Klaviyo-Daten (ausstehend - über Klaviyos nativem BigQuery Export)
- ⏳ Zendesk-Daten (ausstehend - über DLT <1 Tag Setup)

## Kostenvergleich

| Lösung | Monatliche Kosten | Jährliche Kosten |
|--------|-------------------|------------------|
| **Diese Lösung** | | |
| dbt (Open Source, Self-Hosted) | ~€20 | ~€240 |
| BigQuery (Scanning + Storage) | ~€20 | ~€240 |
| Hightouch (Free Tier: 2 Syncs) | €0 | €0 |
| Klaviyo (besteht bereits) | €0 | €0 |
| **Gesamt** | **~€40/Monat** | **~€480/Jahr** |
| | | |
| **Salesforce Alternative** | | |
| Sales Cloud | | ~€20.000+ |
| Data Cloud | | ~€60.000+ |
| Marketing Cloud | | ~€15.000+ |
| Implementierung | | ~€50.000+ |
| **Gesamt** | | **€145.000+/Jahr** |

### Hightouch Free Tier Details
- **2 aktive Syncs** pro Monat (reicht für: 1x dim_customers → Klaviyo Profiles + 1x Segment-Sync)
- Stündliche Sync-Frequenz (maximal)
- Unbegrenzte Destinations und User Seats
- 100 Mio. Operations/Monat
- Für mehr Syncs: Self-Serve Plan (Preis auf Anfrage) oder Segmente direkt in Klaviyo aus den synchronisierten Properties erstellen

### Implementierungsvergleich

| | Diese Lösung | Salesforce |
|--|--------------|------------|
| **Externe Hilfe nötig** | Nein | Ja (Implementierungspartner) |
| **Vorhandene Expertise** | Ja (BigQuery, dbt, Klaviyo bereits im Einsatz) | Nein |
| **Beratungsaufwand** | Minimal | Hoch (Discovery, Workshops, Change Management) |
| **Time-to-Value** | Wochen | Monate |
| **Risiko** | Gering (inkrementell erweiterbar) | Hoch (Big-Bang Migration) |

**Fazit:** Beide Lösungen erfordern Implementierung, aber diese Lösung nutzt bereits vorhandene Tools und internes Know-how. Salesforce würde externe Berater, längere Projektlaufzeiten und erheblichen Abstimmungsaufwand erfordern.

**Hauptaufwand:** Der größte Implementierungsaufwand liegt im Erstellen der dbt-Modelle. Dies ist jedoch ein perfekter Use Case für **Claude Code** (KI-gestützte Entwicklung), wodurch wir deutlich schneller sind als bei klassischer Entwicklung.

## Nächste Schritte

1. [ ] Quellen in BigQuery verifizieren (Schema-Namen anpassen)
2. [ ] dbt-Modelle deployen & testen
3. [ ] Hightouch Free Tier aktivieren
4. [ ] Ersten Sync (dim_customers) konfigurieren
5. [ ] In Klaviyo: Flow für erstes Segment erstellen

---

# Anhang: Technische Details

## Modell-Struktur

```
models/
├── staging/
│   ├── stg_shopify__customers.sql
│   ├── stg_shopify__orders.sql
│   ├── stg_amazon__customers.sql
│   ├── stg_amazon__orders.sql
│   ├── stg_shipments.sql
│   ├── stg_klaviyo__profiles.sql
│   ├── stg_klaviyo__events.sql
│   ├── stg_zendesk__tickets.sql
│   └── stg_zendesk__users.sql
├── intermediate/
│   ├── int_customer__order_metrics.sql
│   ├── int_customer__shipment_metrics.sql
│   ├── int_customer__email_engagement.sql
│   └── int_customer__support_metrics.sql
├── marts/
│   ├── dim_customers.sql              # ← Haupt-Modell für Klaviyo Profiles
│   ├── events_shipments.sql           # ← Für Klaviyo Events (Shipped/Delivered)
│   └── segments/
│       ├── seg_winback_candidates.sql
│       ├── seg_vip_customers.sql
│       ├── seg_repeat_purchase_candidates.sql
│       └── seg_high_value_at_risk.sql
└── schema.yml
```

## Hightouch Setup

### 1. Quelle: BigQuery verbinden

```
Project ID: merz-logistic-merge-poc
Dataset: dbt_prod_klavyio (dedizierter Mart für Klaviyo Marts)
Service Account: bigquery-mlcflow@merz-logistic-merge-poc.iam.gserviceaccount.com
```

### 2. Modelle auswählen

Hightouch kann direkt dbt-Modelle referenzieren:
- Aktiviere "dbt Core" Integration
- Verbinde das [Git Repo](https://github.com/MLC-Digital-Transformation/dbt)
- Modelle mit Tag `klaviyo_sync` werden automatisch erkannt

### 3. Syncs konfigurieren

#### Sync 1: Customer Profiles (dim_customers → Klaviyo Profiles)

| BigQuery-Spalte | Klaviyo-Feld | Typ |
|-----------------|--------------|-----|
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

**Zeitplan:** z.B. Alle 6 Stunden oder wenn die Marts refreshed werden.

#### Sync 2: Shipment Events (events_shipments → Klaviyo Events)

| BigQuery-Spalte | Klaviyo-Feld | Typ |
|-----------------|--------------|-----|
| email | Email | Identifier |
| event_name | Event Name | Event (Package Shipped / Package Delivered) |
| tracking_url | Tracking URL | Property |
| carrier | Carrier | Property |
| order_id | Order ID | Property |
| event_timestamp | Timestamp | Property |

**Zeitplan:** Stündlich (oder häufiger bei Bedarf)

#### Sync 3-6: Segmente (seg_* → Klaviyo Lists)

Für jede Segment-Tabelle:
1. Neuen Sync erstellen
2. Modell: z.B. `seg_winback_candidates`
3. Ziel: Klaviyo List
4. Modus: **Mirror** (Profile werden automatisch hinzugefügt/entfernt)
5. Abgleich über: Email

**Zeitplan:** Täglich oder alle 12 Stunden
