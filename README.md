# 🚂 UK Railways Power BI Dashboard

A Power BI analytics project exploring UK rail ticket sales — passenger behaviour, sales performance, railway operations, and route analysis — built on a star-schema semantic model with 69 custom DAX measures.

![Power BI](https://img.shields.io/badge/Power%20BI-Desktop-F2C811?logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-69%20Measures-blue)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)

---

## 📌 Project Overview

This project analyzes ticket sales data for a UK railway operator to answer key business questions:

- How are passengers travelling — by railcard, ticket class, fare type, and payment method?
- What drives revenue, and how does it trend month over month?
- How punctual is the railway, and where are delays and cancellations concentrated?
- Which stations and routes carry the most traffic, and which are the most profitable?

The result is a 4-page interactive Power BI report backed by a clean star-schema data model.

---

## 🛠 Tools & Skills Used

- **Power BI Desktop** — data modeling, report design
- **Power Query** — data cleaning and transformation (star schema build from a single flat file)
- **DAX** — 69 measures covering KPIs, time intelligence, and dynamic HTML/SVG visuals
- **Data Modeling** — star schema with role-playing dimensions via `USERELATIONSHIP()`

---

## 🗂 Dataset

**Source:** `railway.csv.xlsx` — a single flat-file export of ~ticket-level transactions, transformed in Power Query into a star schema:

| Table | Type | Description |
|---|---|---|
| `FACT_TICKET_SALES` | Fact | One row per ticket sold (price, refund flag, journey/purchase keys) |
| `DIM_DATE` | Dimension | Calendar attributes for journey & purchase dates |
| `DIM_TIME` | Dimension | Minute-level time with peak/off-peak bands |
| `DIM_STATION` | Dimension | Shared departure/arrival station list |
| `DIM_PURCHASE_CHANNEL` | Dimension | Purchase type, payment method, railcard |
| `DIM_TICKET_TYPE` | Dimension | Ticket class & fare type |
| `DIM_JOURNEY_STATUS` | Dimension | On time / delayed / cancelled + delay reason |

Seven role-playing relationships (e.g. departure vs. arrival station/time) are kept inactive and activated in DAX via `USERELATIONSHIP()`.

---

## 📊 Dashboard Pages

| Page | Focus |
|---|---|
| **1. Passenger Analysis** | Railcard penetration, ticket class mix, booking type, peak travel bands |
| **2. Sales Performance** | Revenue trends, MoM growth, channel mix, purchase lead time |
| **3. Railway Performance** | On-time %, delays, cancellations, refund exposure |
| **4. Route Analysis** | Busiest routes, top stations, revenue yield per route |

*(Add dashboard screenshots here, e.g. `/screenshots/page1.png`)*

---

## 💡 Key Insights

- Identified the busiest routes and stations and their peak travel times.
- Quantified the revenue share and pricing gap between railcard holders and non-holders.
- Measured on-time performance and pinpointed the leading causes of delay.
- Tracked month-over-month revenue and ticket volume trends.

---

## 📐 Highlights of the DAX Layer

69 measures organized into 5 display folders (Common, Passenger Analysis, Sales Performance, Railway Performance, Route Analysis), including:

- Time-intelligence measures (`Revenue MoM %`, `Revenue MTD`, `Revenue Previous Month`)
- Role-playing dimension measures using `USERELATIONSHIP()` for arrival vs. departure analysis
- Dynamic HTML/SVG visuals (KPI cards, station carousel, trend charts) rendered natively in DAX

> Full measure-by-measure documentation with DAX code is available in [`/docs/measures.md`](./docs/measures.md).

---

## 📁 Repository Structure

```
uk-railways-powerbi-report/
├── README.md              # This file
├── Report.pbix             # Power BI report (if included)
├── docs/
│   └── measures.md         # Full DAX measure reference
└── screenshots/            # Dashboard page images
```

---

*Built with Power BI Desktop · UK Railways Project*
