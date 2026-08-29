# 🚂 UK Railways Power BI Report

> **Report file:** `Report.pbix`
> **Model name:** `Model`
> **Default mode:** Import
> **Culture:** `en-US`
> **Last structure modified:** 28 Aug 2026

A comprehensive Power BI analytics solution for UK rail ticket sales, covering passenger behaviour, sales performance, railway operations, and route analysis.

---

## 📋 Table of Contents

1. [Semantic Model Overview](#-semantic-model-overview)
2. [Tables & Columns](#-tables--columns)
   - [FACT\_TICKET\_SALES](#fact_ticket_sales)
   - [DIM\_DATE](#dim_date)
   - [DIM\_TIME](#dim_time)
   - [DIM\_STATION](#dim_station)
   - [DIM\_PURCHASE\_CHANNEL](#dim_purchase_channel)
   - [DIM\_TICKET\_TYPE](#dim_ticket_type)
   - [DIM\_JOURNEY\_STATUS](#dim_journey_status)
   - [\_Measures](#_measures)
   - [\_RefreshInfo](#_refreshinfo)
3. [Relationships](#-relationships)
4. [Measures Reference](#-measures-reference)
   - [00 Common](#folder-00-common)
   - [01 Passenger Analysis](#folder-01-passenger-analysis)
   - [02 Sales Performance](#folder-02-sales-performance)
   - [03 Railway Performance](#folder-03-railway-performance)
   - [04 Route Analysis](#folder-04-route-analysis)
5. [Report Pages](#-report-pages)
   - [Page 1 — Passenger Analysis](#page-1--passenger-analysis)
   - [Page 2 — Sales Performance](#page-2--sales-performance)
   - [Page 3 — Railway Performance](#page-3--railway-performance)
   - [Page 4 — Route Analysis](#page-4--route-analysis)
6. [Data Source & Power Query](#-data-source--power-query)

---

## 🗂 Semantic Model Overview

The model follows a classic **star schema** with one central fact table surrounded by six dimension tables. All tables are loaded in **Import** mode from a single Excel source (`railway.csv.xlsx`).

```
DIM_DATE ──────────────────────────────┐
DIM_TIME ──────────────────────────────┤
DIM_STATION (departure, active) ───────┤
DIM_STATION (arrival,   inactive) ─────┼──► FACT_TICKET_SALES
DIM_PURCHASE_CHANNEL ──────────────────┤
DIM_TICKET_TYPE ───────────────────────┤
DIM_JOURNEY_STATUS ────────────────────┘
```

> **Design note:** Multiple **inactive** relationships from `FACT_TICKET_SALES` to `DIM_TIME` and `DIM_STATION` enable role-playing dimensions (e.g. departure vs. arrival time, departure vs. arrival station) via `USERELATIONSHIP()` in DAX measures.

---

## 📊 Tables & Columns

### FACT\_TICKET\_SALES

The central grain of the model. **Each row = one ticket sold.**

| Column | Data Type | Hidden | Description |
|---|---|---|---|
| `Transaction ID` | Text | No | Unique ticket identifier |
| `Price` | Integer | No | Ticket price in GBP |
| `Refund Request` | Boolean | No | Whether the passenger requested a refund (`TRUE` / `FALSE`) |
| `JourneyDateKey` | Integer | Yes | FK → `DIM_DATE[DateKey]` (**active**) |
| `PurchaseDateKey` | Integer | Yes | FK → `DIM_DATE[DateKey]` (inactive) |
| `DepartureTimeKey` | Integer | Yes | FK → `DIM_TIME[TimeKey]` (**active**) |
| `ArrivalTimeKey` | Integer | Yes | FK → `DIM_TIME[TimeKey]` (inactive) |
| `ActualArrivalTimeKey` | Integer | Yes | FK → `DIM_TIME[TimeKey]` (inactive) |
| `PurchaseTimeKey` | Integer | Yes | FK → `DIM_TIME[TimeKey]` (inactive) |
| `DepartureStationKey` | Integer | Yes | FK → `DIM_STATION[StationKey]` (**active**) |
| `ArrivalStationKey` | Integer | Yes | FK → `DIM_STATION[StationKey]` (inactive) |
| `ChannelKey` | Integer | Yes | FK → `DIM_PURCHASE_CHANNEL[ChannelKey]` (**active**) |
| `TicketTypeKey` | Integer | Yes | FK → `DIM_TICKET_TYPE[TicketTypeKey]` (**active**) |
| `StatusKey` | Integer | Yes | FK → `DIM_JOURNEY_STATUS[StatusKey]` (**active**) |

**Power Query source:** The raw `railway` table, with all surrogate keys computed via lookup merges and date/time integer encoding (`yyyyMMdd`, `HHmm`).

---

### DIM\_DATE

Date dimension covering all purchase and journey dates. Marked as a **Date Table** (`dataCategory: Time`).

| Column | Data Type | Hidden | Notes |
|---|---|---|---|
| `DateKey` | Integer | Yes | `yyyyMMdd` surrogate key |
| `Date` | Date | No | Full calendar date (unique) |
| `Year` | Integer | No | Calendar year |
| `Quarter` | Text | No | e.g. `Q1`, `Q2` |
| `MonthNo` | Integer | No | Month number 1–12 |
| `MonthName` | Text | No | e.g. `January`; sorted by `MonthNo` |
| `Day` | Integer | No | Day of month |
| `DayOfWeek` | Text | No | e.g. `Monday` |
| `IsWeekend` | Boolean | No | `TRUE` for Saturday / Sunday |
| `Day Appr` | Text | No | 3-letter abbreviation (e.g. `Mon`); sorted by `DayOfWeekNumber` |
| `DayOfWeekNumber` *(calculated)* | Integer | No | `WEEKDAY([Date], 1)` — used for sort order |

**Power Query source:** Generated from the union of all distinct purchase dates and journey dates in the `railway` table, then expanded into a full date spine.

---

### DIM\_TIME

Time dimension at **minute** granularity. Acts as a role-playing dimension for four time-type relationships.

| Column | Data Type | Hidden | Notes |
|---|---|---|---|
| `Time` | Time | No | Minute-truncated time value (format: `hh:mm`) |
| `TimeKey` | Integer | Yes | `HH * 100 + MM` integer key |
| `TimeBand` | Text | No | Banded time-of-day category (see below) |
| `Time - Copy` | Time | No | Hour-truncated copy of `Time` |
| `AM/PM` | Text | No | `AM` or `PM` |

**Time band definitions:**

| Band | Hours (24 h) |
|---|---|
| Night | 00:00 – 05:59 |
| Morning Peak | 06:00 – 09:59 |
| Midday | 10:00 – 15:59 |
| Evening Peak | 16:00 – 18:59 |
| Evening | 19:00 – 23:59 |

**Power Query source:** Union of all four time columns from `railway` (purchase, departure, arrival, actual arrival), truncated to the minute, deduplicated, sorted, then `TimeKey`, `TimeBand`, and `AM/PM` columns are added.

---

### DIM\_STATION

Shared station dimension used for both departure **and** arrival via role-playing relationships.

| Column | Data Type | Hidden |
|---|---|---|
| `StationName` | Text | No |
| `StationKey` | Integer | Yes |

**Power Query source:** Union of `Departure Station` and `Arrival Destination` from `railway`, deduplicated and assigned an auto-incremented integer key.

---

### DIM\_PURCHASE\_CHANNEL

Describes how the ticket was purchased and paid for, including railcard status.

| Column | Data Type | Hidden | Notes |
|---|---|---|---|
| `Purchase Type` | Text | No | `Online` or `Station` |
| `Payment Method` | Text | No | e.g. `Contactless`, `Credit Card`, `Debit Card` |
| `Railcard` | Text | No | Railcard type, or `None` |
| `ChannelKey` | Integer | Yes | Surrogate key |
| `Railcard Status` *(calculated column)* | Text | No | `Railcard Holder` if Railcard ≠ `None`, else `Non Railcard Holder` |

**Calculated column DAX:**

```dax
IF(
    NOT ISBLANK(DIM_PURCHASE_CHANNEL[Railcard])
        && UPPER(TRIM('DIM_PURCHASE_CHANNEL'[Railcard])) <> "NONE",
    "Railcard Holder",
    "Non Railcard Holder"
)
```

**Power Query source:** Distinct combinations of `Purchase Type`, `Payment Method`, and `Railcard` from `railway`.

---

### DIM\_TICKET\_TYPE

Ticket classification by class and fare type.

| Column | Data Type | Hidden |
|---|---|---|
| `Ticket Class` | Text | No |
| `Ticket Type` | Text | No |
| `TicketTypeKey` | Integer | Yes |

**Values:** `Ticket Class` → `Standard Class`, `First Class` · `Ticket Type` → `Advance`, `Anytime`, `Off-Peak`

---

### DIM\_JOURNEY\_STATUS

Journey outcome and disruption reason.

| Column | Data Type | Hidden |
|---|---|---|
| `Journey Status` | Text | No |
| `Reason for Delay` | Text | No |
| `StatusKey` | Integer | Yes |

**Values for `Journey Status`:** `On Time`, `Delayed`, `Cancelled`
**Values for `Reason for Delay`:** `Weather Conditions`, `Signal Failure`, `Technical Issue`, `Traffic`, `Staff Shortage`, and others (blank for on-time / non-delayed journeys).

---

### \_Measures

A dedicated **disconnected table** that hosts all **69 measures**. Contains a single hidden placeholder column `Measures` (Integer). Measures are organised into five display folders.

---

### \_RefreshInfo

Single-column utility table that records the last model refresh timestamp.

| Column | Data Type |
|---|---|
| `LastRefresh` | DateTime |

Used by the `Last Refreshed Text`, `Last Refreshed (Raw)`, `Last Refresh Badge HTML`, and `Last Refresh Clock HTML` measures.

---

## 🔗 Relationships

All relationships are **Many-to-One** (`FACT_TICKET_SALES` → Dimension) with **single-direction** cross-filtering.

| From Table | From Column | To Table | To Column | Active | Notes |
|---|---|---|---|---|---|
| `FACT_TICKET_SALES` | `JourneyDateKey` | `DIM_DATE` | `DateKey` | ✅ | Default date context |
| `FACT_TICKET_SALES` | `PurchaseDateKey` | `DIM_DATE` | `DateKey` | ❌ | Activated via `USERELATIONSHIP` |
| `FACT_TICKET_SALES` | `DepartureTimeKey` | `DIM_TIME` | `TimeKey` | ✅ | Default time context |
| `FACT_TICKET_SALES` | `ArrivalTimeKey` | `DIM_TIME` | `TimeKey` | ❌ | Activated in arrival-time measures |
| `FACT_TICKET_SALES` | `ActualArrivalTimeKey` | `DIM_TIME` | `TimeKey` | ❌ | Activated in delay calculation measures |
| `FACT_TICKET_SALES` | `PurchaseTimeKey` | `DIM_TIME` | `TimeKey` | ❌ | Available for purchase-time analysis |
| `FACT_TICKET_SALES` | `DepartureStationKey` | `DIM_STATION` | `StationKey` | ✅ | Default station context |
| `FACT_TICKET_SALES` | `ArrivalStationKey` | `DIM_STATION` | `StationKey` | ❌ | Activated via `USERELATIONSHIP` |
| `FACT_TICKET_SALES` | `ChannelKey` | `DIM_PURCHASE_CHANNEL` | `ChannelKey` | ✅ | |
| `FACT_TICKET_SALES` | `TicketTypeKey` | `DIM_TICKET_TYPE` | `TicketTypeKey` | ✅ | |
| `FACT_TICKET_SALES` | `StatusKey` | `DIM_JOURNEY_STATUS` | `StatusKey` | ✅ | |
| `DIM_DATE` | `Date` | `LocalDateTable_*` | `Date` | ✅ | Auto-generated by Power BI for date hierarchy drilldown |

> **7 of 12 relationships are inactive** — this is intentional, enabling role-playing dimension patterns without causing DAX ambiguity errors.

---

## 📐 Measures Reference

All 69 measures live in the `_Measures` table and are grouped into five display folders.

---

### Folder: 00 Common

Foundation measures used across all pages and as building blocks for derived measures.

---

#### `Total Revenue`
> **Format:** `£#,##0`

```dax
SUM('FACT_TICKET_SALES'[Price])
```

Total ticket revenue in GBP.

---

#### `Ticket Count`
> **Format:** `#,##0`

```dax
COUNTROWS('FACT_TICKET_SALES')
```

Total number of tickets sold.

---

#### `Total Revenue (by Purchase Date)`
> **Format:** `£#,##0`

```dax
CALCULATE(
    [Total Revenue],
    USERELATIONSHIP('FACT_TICKET_SALES'[PurchaseDateKey], DIM_DATE[DateKey])
)
```

Revenue filtered by **purchase date** instead of journey date. Activates the inactive `PurchaseDateKey → DIM_DATE` relationship.

---

#### `Average Ticket Price`
> **Format:** `£#,##0.00` · *Average revenue per ticket sold.*

```dax
DIVIDE([Total Revenue], [Ticket Count])
```

---

#### `Revenue Previous Month`
> **Format:** `£#,##0` · *Revenue for the previous calendar month, by journey date.*

```dax
CALCULATE([Total Revenue], PREVIOUSMONTH(DIM_DATE[Date]))
```

---

#### `Revenue MoM %`
> **Format:** `0.0%;-0.0%;0.0%` · *Month-over-month change in revenue.*

```dax
DIVIDE([Total Revenue] - [Revenue Previous Month], [Revenue Previous Month])
```

---

#### `Ticket Count MoM %`
> **Format:** `0.0%;-0.0%;0.0%` · *Month-over-month change in tickets sold.*

```dax
VAR Prev = CALCULATE([Ticket Count], PREVIOUSMONTH(DIM_DATE[Date]))
RETURN DIVIDE([Ticket Count] - Prev, Prev)
```

---

#### `Revenue % of Total`
> **Format:** `0.0%` · *Share of overall revenue represented by the current filter context.*

```dax
DIVIDE([Total Revenue], CALCULATE([Total Revenue], REMOVEFILTERS()))
```

---

#### `Journey Days`
> **Format:** `#,##0` · *Number of distinct days on which journeys took place.*

```dax
DISTINCTCOUNT('FACT_TICKET_SALES'[JourneyDateKey])
```

---

#### `Actual Journey Days`
> **Format:** `0`

```dax
CALCULATE(
    COUNTROWS(FACT_TICKET_SALES),
    DIM_JOURNEY_STATUS[Journey Status] <> "Cancelled"
)
```

Counts rows excluding cancelled journeys — i.e., journeys that actually operated.

---

#### `ZOBRY`
> **Format:** `0.00` *(Internal / scratch measure)*

```dax
CALCULATE(
    COUNTROWS(FACT_TICKET_SALES),
    DIM_PURCHASE_CHANNEL[Purchase Type] = "Station"
)
```

Count of station-purchased tickets. Functionally equivalent to `Station Purchase Journeys`.

---

#### `Last Refreshed Text`
> **Format:** *(text)* · *Formatted 'last refreshed' label for a card or text visual.*

```dax
"Data last refreshed: "
    & FORMAT(MAX('_RefreshInfo'[LastRefresh]), "dd mmm yyyy \a\t h:mm AM/PM")
```

Returns a string like `Data last refreshed: 28 Aug 2026 at 12:30 PM`. Updates only on model refresh.

---

#### `Last Refreshed (Raw)`
> **Format:** `dd mmm yyyy hh:mm:ss AM/PM` · *Raw last-refresh timestamp for conditional formatting.*

```dax
MAX('_RefreshInfo'[LastRefresh])
```

---

#### `Station Carousel HTML`
> **Format:** *(HTML)* · *Infinite auto-scrolling station carousel. Bind to an HTML Content visual.*

```dax
VAR StationPills =
    CONCATENATEX(
        VALUES(DIM_STATION[StationName]),
        "<span class='pill'>" & DIM_STATION[StationName] & "</span>",
        "",
        DIM_STATION[StationName], ASC
    )
RETURN
    "<style>
        @import url('https://fonts.googleapis.com/css2?family=Barlow+Condensed:wght@700&display=swap');
        .carousel-wrap { width:100%; overflow:hidden; background:#152A5C; border-radius:6px;
                         border-top:3px solid #C8102E; display:flex; align-items:center; }
        .carousel-track { display:flex; white-space:nowrap;
                          animation:scrollTrack 45s linear infinite; }
        .pill { display:inline-flex; align-items:center; gap:9px; color:#FFFFFF;
                font-family:'Barlow Condensed'; font-weight:700; font-size:16px;
                letter-spacing:1px; text-transform:uppercase; padding:8px 20px;
                border:1.5px solid #FFFFFF; border-radius:3px; }
        .pill::before { content:''; width:7px; height:7px;
                        background:#C8102E; border-radius:50%; }
        @keyframes scrollTrack { 0%{transform:translateX(0);} 100%{transform:translateX(-50%);} }
    </style>
    <div class='carousel-wrap'>
        <div class='carousel-track'>" & StationPills & StationPills & "</div>
    </div>"
```

Dynamically regenerates whenever stations are added or removed from the model.

---

#### `Last Refresh Badge HTML`
> **Format:** *(HTML)* · *'DATA REFRESHED HH:MM - DD MON YYYY' muted-grey badge.*

```dax
"<style>@import url('...Barlow+Condensed...');</style>"
& "<div style='...font-size:15px; letter-spacing:0.8px; color:#746F80;...'>"
& "DATA REFRESHED "
& FORMAT(MAX('_RefreshInfo'[LastRefresh]), "hh:mm")
& " - "
& UPPER(FORMAT(MAX('_RefreshInfo'[LastRefresh]), "dd mmm yyyy"))
& "</div>"
```

Bind to an HTML Content visual.

---

#### `Last Refresh Clock HTML`
> **Format:** *(HTML)* · *Large clock-style refresh time in JetBrains Mono, colour `#312782`.*

```dax
"<style>@import url('...JetBrains+Mono...');</style>"
& "<div style='...font-size:32px; color:#312782;...'>"
& FORMAT(MAX('_RefreshInfo'[LastRefresh]), "h:mm AM/PM")
& "</div>"
```

Typically paired above or beside `Last Refresh Badge HTML`.

---

### Folder: 01 Passenger Analysis

Measures for understanding passenger demographics, railcard usage, ticket preferences, and travel patterns.

---

#### `Railcard Tickets`
> **Format:** `#,##0` · *Tickets purchased using any railcard discount.*

```dax
CALCULATE([Ticket Count], DIM_PURCHASE_CHANNEL[Railcard] <> "None")
```

---

#### `Railcard Penetration %`
> **Format:** `0.0%` · *Share of tickets sold with a railcard discount applied.*

```dax
DIVIDE([Railcard Tickets], [Ticket Count])
```

---

#### `Avg Price (Railcard Holders)`
> **Format:** `£#,##0.00` · *Average price paid by railcard holders.*

```dax
CALCULATE([Average Ticket Price], DIM_PURCHASE_CHANNEL[Railcard] <> "None")
```

---

#### `Avg Price (No Railcard)`
> **Format:** `£#,##0.00` · *Average price paid by passengers without a railcard.*

```dax
CALCULATE([Average Ticket Price], DIM_PURCHASE_CHANNEL[Railcard] = "None")
```

---

#### `First Class Mix %`
> **Format:** `0.0%` · *Share of tickets sold in First Class.*

```dax
DIVIDE(
    CALCULATE([Ticket Count], DIM_TICKET_TYPE[Ticket Class] = "First Class"),
    [Ticket Count]
)
```

---

#### `Advance Booking %`
> **Format:** `0.0%` · *Share of tickets booked on an Advance fare.*

```dax
DIVIDE(
    CALCULATE([Ticket Count], DIM_TICKET_TYPE[Ticket Type] = "Advance"),
    [Ticket Count]
)
```

---

#### `Online Purchase %`
> **Format:** `0.0%` · *Share of tickets purchased online rather than at a station.*

```dax
DIVIDE(
    CALCULATE([Ticket Count], DIM_PURCHASE_CHANNEL[Purchase Type] = "Online"),
    [Ticket Count]
)
```

---

#### `Contactless Payment %`
> **Format:** `0.0%` · *Share of tickets paid for by contactless.*

```dax
DIVIDE(
    CALCULATE([Ticket Count], DIM_PURCHASE_CHANNEL[Payment Method] = "Contactless"),
    [Ticket Count]
)
```

---

#### `Peak Travel Band`
> **Format:** *(text)* · *The time-of-day band with the most departures in context.*

```dax
VAR Bands =
    ADDCOLUMNS(
        VALUES(DIM_TIME[TimeBand]),
        "@Tix", CALCULATE([Ticket Count])
    )
RETURN
    CONCATENATEX(TOPN(1, Bands, [@Tix], DESC), DIM_TIME[TimeBand], ", ")
```

Returns a single string such as `"Morning Peak"` or `"Evening Peak"`.

---

#### `Label`
> **Format:** *(text)*

```dax
FORMAT([Actual Journey Days], "#,##0") & " | " & [Most Common Departure Time]
```

Combined label used in bar charts: actual journey count + most common departure time for the station in context.

---

#### `Railcard Status Icon`
> **Data Category:** `ImageUrl` · *Dynamic railcard SVG icon.*

```dax
VAR _RailcardStatus = SELECTEDVALUE(DIM_PURCHASE_CHANNEL[Railcard Status])
RETURN
    IF(
        _RailcardStatus = "Non Railcard Holder",
        "data:image/svg+xml;base64,<SVG with red X overlay>",
        "data:image/svg+xml;base64,<standard railcard SVG>"
    )
```

Returns a base64-encoded SVG data URI. Shows the standard railcard icon for `Railcard Holder`; adds a red ✕ overlay for `Non Railcard Holder`. Bind to an **Image** visual.

---

#### `Railcard Status Icon HTML`
> **Format:** *(HTML)* · *HTML wrapper for the Railcard Status Icon. Bind to an HTML Content visual.*

```dax
"<div style='display:flex; align-items:center; justify-content:center;...'>"
& "<img src='" & [Railcard Status Icon] & "' style='max-width:100%;...' />"
& "</div>"
```

---

#### `KPI Card Ticket Count HTML`
> **Format:** *(HTML)* · *KPI card for Ticket Count in Barlow Condensed, navy-on-white.*

```dax
"<style>@import url('...Barlow+Condensed...');</style>"
& "<div style='...font-family:Barlow Condensed;...'>"
    & "<div style='font-size:12px; color:#8A8F98; text-transform:uppercase;'>Ticket Count</div>"
    & "<div style='font-size:40px; color:#152A5C;'>" & FORMAT([Ticket Count], "#,##0") & "</div>"
& "</div>"
```

---

#### `KPI Card Railcard Penetration HTML`
> **Format:** *(HTML)* · *KPI card for Railcard Penetration %.*

```dax
-- Same card structure as above
-- Displays: FORMAT([Railcard Penetration %], "0.0%")
```

---

#### `KPI Card First Class Mix HTML`
> **Format:** *(HTML)* · *KPI card for First Class Mix %.*

```dax
-- Same card structure
-- Displays: FORMAT([First Class Mix %], "0.0%")
```

---

#### `KPI Card Peak Travel Band HTML`
> **Format:** *(HTML)* · *KPI card for Peak Travel Band.*

```dax
-- Same card structure
-- Displays: [Peak Travel Band]
```

---

### Folder: 02 Sales Performance

Measures focused on revenue performance, channel mix, and purchasing behaviour.

---

#### `Avg Daily Revenue`
> **Format:** `£#,##0` · *Average revenue earned per day with journeys.*

```dax
DIVIDE([Total Revenue], [Journey Days])
```

---

#### `Best Day Revenue`
> **Format:** `£#,##0` · *Highest single-day revenue in context.*

```dax
MAXX(VALUES(DIM_DATE[Date]), CALCULATE([Total Revenue]))
```

---

#### `Online Revenue`
> **Format:** `£#,##0` · *Revenue from tickets bought online.*

```dax
CALCULATE([Total Revenue], DIM_PURCHASE_CHANNEL[Purchase Type] = "Online")
```

---

#### `Station Revenue`
> **Format:** `£#,##0` · *Revenue from tickets bought at a station.*

```dax
CALCULATE([Total Revenue], DIM_PURCHASE_CHANNEL[Purchase Type] = "Station")
```

---

#### `Online Revenue Mix %`
> **Format:** `0.0%` · *Share of revenue coming through the online channel.*

```dax
DIVIDE([Online Revenue], [Total Revenue])
```

---

#### `Revenue MTD`
> **Format:** `£#,##0` · *Revenue month to date, by journey date.*

```dax
TOTALMTD([Total Revenue], DIM_DATE[Date])
```

---

#### `Avg Purchase Lead Time (Days)`
> **Format:** `0.0` · *Average number of days between buying the ticket and travelling.*

```dax
AVERAGEX(
    'FACT_TICKET_SALES',
    VAR PKey  = 'FACT_TICKET_SALES'[PurchaseDateKey]
    VAR JKey  = 'FACT_TICKET_SALES'[JourneyDateKey]
    VAR PDate = DATE(INT(PKey / 10000), MOD(INT(PKey / 100), 100), MOD(PKey, 100))
    VAR JDate = DATE(INT(JKey / 10000), MOD(INT(JKey / 100), 100), MOD(JKey, 100))
    RETURN DATEDIFF(PDate, JDate, DAY)
)
```

Reconstructs full `DATE` values from the `yyyyMMdd` integer keys before computing the difference.

---

#### `Same Day Purchase %`
> **Format:** `0.0%` · *Share of tickets bought on the day of travel.*

```dax
VAR SameDay =
    COUNTROWS(
        FILTER(
            'FACT_TICKET_SALES',
            'FACT_TICKET_SALES'[PurchaseDateKey] = 'FACT_TICKET_SALES'[JourneyDateKey]
        )
    )
RETURN DIVIDE(SameDay, [Ticket Count])
```

---

#### `Station Purchase Journeys`
> **Format:** `#,##0` · *Tickets/journeys purchased at a station rather than online.*

```dax
CALCULATE([Ticket Count], DIM_PURCHASE_CHANNEL[Purchase Type] = "Station")
```

---

#### `Avg Passengers per Day`
> **Format:** `#,##0` · *Average number of passengers (tickets) per day with journeys.*

```dax
DIVIDE([Ticket Count], [Journey Days])
```

---

#### `KPI Card Average Ticket Price HTML`
> **Format:** *(HTML)* · *KPI card for Average Ticket Price.*

```dax
-- Displays: "£" & FORMAT([Average Ticket Price], "#,##0.00")
```

---

#### `KPI Card Avg Passengers per Day HTML`
> **Format:** *(HTML)* · *KPI card for Avg Passengers per Day.*

```dax
-- Displays: FORMAT([Avg Passengers per Day], "#,##0")
```

---

#### `Passenger Trend Area Chart SVG`
> **Format:** *(SVG / HTML)* · *Self-contained SVG area chart of Ticket Count by month.*

```dax
VAR MonthData =
    SUMMARIZECOLUMNS(
        DIM_DATE[MonthNo], DIM_DATE[MonthName],
        "@Tickets", CALCULATE([Ticket Count]),
        "@MoM",     CALCULATE([Ticket Count MoM %])
    )
-- Computes normalised X/Y pixel coordinates for each month point
-- Renders:
--   <path>  — shaded area fill (navy, 10 % opacity)
--   <path>  — stroke line (navy #152A5C)
--   <circle> markers at each data point
--   Value labels above each point
--   MoM % trend arrows: ▲ green (#1E8E3E) for growth, ▼ red (#D93025) for decline
--   Month abbreviation labels on the X-axis
-- Dynamically adapts to however many months exist in the model
```

Bind to an **HTML Content** visual.

---

### Folder: 03 Railway Performance

Measures for operational KPIs: punctuality, delays, cancellations, and refund risk.

---

#### `On Time Journeys`
> **Format:** `#,##0` · *Journeys that arrived on time.*

```dax
CALCULATE([Ticket Count], DIM_JOURNEY_STATUS[Journey Status] = "On Time")
```

---

#### `On Time %`
> **Format:** `0.0%` · *Share of journeys arriving on time.*

```dax
DIVIDE([On Time Journeys], [Ticket Count])
```

---

#### `Delayed Journeys`
> **Format:** `#,##0` · *Journeys that ran but arrived late.*

```dax
CALCULATE([Ticket Count], DIM_JOURNEY_STATUS[Journey Status] = "Delayed")
```

---

#### `Delayed %`
> **Format:** `0.0%` · *Share of journeys arriving late.*

```dax
DIVIDE([Delayed Journeys], [Ticket Count])
```

---

#### `Cancelled Journeys`
> **Format:** `#,##0` · *Journeys cancelled before completion.*

```dax
CALCULATE([Ticket Count], DIM_JOURNEY_STATUS[Journey Status] = "Cancelled")
```

---

#### `Cancellation %`
> **Format:** `0.0%` · *Share of journeys cancelled.*

```dax
DIVIDE([Cancelled Journeys], [Ticket Count])
```

---

#### `Avg Delay Minutes`
> **Format:** `0.0` · *Average minutes late across delayed journeys (scheduled vs. actual arrival).*

```dax
AVERAGEX(
    FILTER(
        'FACT_TICKET_SALES',
        NOT ISBLANK('FACT_TICKET_SALES'[ActualArrivalTimeKey])
    ),
    VAR SchedKey  = 'FACT_TICKET_SALES'[ArrivalTimeKey]
    VAR ActKey    = 'FACT_TICKET_SALES'[ActualArrivalTimeKey]
    VAR SchedMins = INT(SchedKey / 100) * 60 + MOD(SchedKey, 100)
    VAR ActMins   = INT(ActKey   / 100) * 60 + MOD(ActKey,   100)
    VAR RawDiff   = ActMins - SchedMins
    VAR AdjDiff   = IF(RawDiff < -720, RawDiff + 1440, RawDiff)   -- midnight crossing
    RETURN IF(AdjDiff > 0, AdjDiff)
)
```

Reads `ArrivalTimeKey` and `ActualArrivalTimeKey` directly from the fact table without requiring `USERELATIONSHIP`. Handles midnight crossings via a `+1440 min` adjustment.

---

#### `Total Delay Minutes`
> **Format:** `#,##0` · *Total passenger-minutes lost to delays.*

```dax
SUMX(
    FILTER('FACT_TICKET_SALES', NOT ISBLANK('FACT_TICKET_SALES'[ActualArrivalTimeKey])),
    VAR SchedKey  = 'FACT_TICKET_SALES'[ArrivalTimeKey]
    VAR ActKey    = 'FACT_TICKET_SALES'[ActualArrivalTimeKey]
    -- same HHmm → minutes conversion as Avg Delay Minutes
    RETURN IF(AdjDiff > 0, AdjDiff, 0)
)
```

---

#### `Refund Requests`
> **Format:** `#,##0` · *Tickets where the passenger requested a refund.*

```dax
CALCULATE([Ticket Count], 'FACT_TICKET_SALES'[Refund Request] = "Yes")
```

---

#### `Refund Request Rate %`
> **Format:** `0.0%` · *Share of tickets with a refund request.*

```dax
DIVIDE([Refund Requests], [Ticket Count])
```

---

#### `Revenue at Risk (Refunds)`
> **Format:** `£#,##0` · *Ticket revenue attached to refund requests.*

```dax
CALCULATE([Total Revenue], 'FACT_TICKET_SALES'[Refund Request] = "Yes")
```

---

#### `Top Delay Reason`
> **Format:** *(text)* · *Most frequent cause of disruption in context.*

```dax
VAR Reasons =
    FILTER(
        ADDCOLUMNS(
            VALUES(DIM_JOURNEY_STATUS[Reason for Delay]),
            "@Cnt", CALCULATE([Ticket Count])
        ),
        NOT ISBLANK(DIM_JOURNEY_STATUS[Reason for Delay])
    )
RETURN
    CONCATENATEX(TOPN(1, Reasons, [@Cnt], DESC), DIM_JOURNEY_STATUS[Reason for Delay], ", ")
```

Returns a single string such as `"Signal Failure"` or `"Weather Conditions"`.

---

### Folder: 04 Route Analysis

Measures for station and route-level analysis, including departure and arrival KPIs.

---

#### `Distinct Routes`
> **Format:** `#,##0` · *Number of unique departure-to-arrival station pairs travelled.*

```dax
COUNTROWS(
    SUMMARIZE(
        'FACT_TICKET_SALES',
        'FACT_TICKET_SALES'[DepartureStationKey],
        'FACT_TICKET_SALES'[ArrivalStationKey]
    )
)
```

---

#### `Avg Tickets per Route`
> **Format:** `#,##0.0` · *Average tickets sold per route.*

```dax
DIVIDE([Ticket Count], [Distinct Routes])
```

---

#### `Avg Revenue per Route`
> **Format:** `£#,##0` · *Average revenue generated per route.*

```dax
DIVIDE([Total Revenue], [Distinct Routes])
```

---

#### `Revenue by Arrival Station`
> **Format:** `£#,##0` · *Revenue grouped by arrival station instead of departure station.*

```dax
CALCULATE(
    [Total Revenue],
    USERELATIONSHIP('FACT_TICKET_SALES'[ArrivalStationKey], DIM_STATION[StationKey])
)
```

---

#### `Tickets by Arrival Station`
> **Format:** `#,##0` · *Tickets grouped by arrival station instead of departure station.*

```dax
CALCULATE(
    [Actual Journey Days],
    USERELATIONSHIP('FACT_TICKET_SALES'[ArrivalStationKey], DIM_STATION[StationKey])
)
```

---

#### `Top Departure Station`
> **Format:** *(text)* · *Departure station with the most tickets in context.*

```dax
VAR Stations =
    ADDCOLUMNS(
        VALUES(DIM_STATION[StationName]),
        "@Tix", CALCULATE([Ticket Count])
    )
RETURN
    CONCATENATEX(TOPN(1, Stations, [@Tix], DESC), DIM_STATION[StationName], ", ")
```

---

#### `Busiest Route`
> **Format:** *(text)* · *The departure–arrival pair carrying the most passengers in context.*

```dax
VAR Routes =
    ADDCOLUMNS(
        SUMMARIZE(
            'FACT_TICKET_SALES',
            'FACT_TICKET_SALES'[DepartureStationKey],
            'FACT_TICKET_SALES'[ArrivalStationKey]
        ),
        "@Tix", CALCULATE([Ticket Count])
    )
VAR TopRoute = TOPN(1, Routes, [@Tix], DESC)
RETURN
    CONCATENATEX(
        TopRoute,
        LOOKUPVALUE(DIM_STATION[StationName], DIM_STATION[StationKey],
                    'FACT_TICKET_SALES'[DepartureStationKey])
            & " to "
            & LOOKUPVALUE(DIM_STATION[StationName], DIM_STATION[StationKey],
                          'FACT_TICKET_SALES'[ArrivalStationKey]),
        ", "
    )
```

Returns a string such as `"Manchester Piccadilly to London Euston"`.

---

#### `Route Yield per Ticket`
> **Format:** `£#,##0.00` · *Revenue per ticket on the routes in context; useful for spotting premium corridors.*

```dax
DIVIDE([Total Revenue], [Ticket Count])
```

---

#### `Most Common Departure Time`
> **Format:** *(text)* · *Single most frequent departure time for the departure station(s) in context.*

```dax
VAR Times =
    ADDCOLUMNS(
        VALUES(DIM_TIME[Time]),
        "@Cnt", CALCULATE([Ticket Count])
    )
VAR TopTime = TOPN(1, Times, [@Cnt], DESC, DIM_TIME[Time], ASC)
RETURN
    CONCATENATEX(TopTime, FORMAT(DIM_TIME[Time], "h:mm AM/PM"), ", ")
```

Returns a string such as `"5:45 PM"`.

---

#### `Station Bar Label`
> **Format:** *(text)* · *Combined 'count | time' label for the Top Departure Stations bar chart.*

```dax
FORMAT([Ticket Count], "#,##0") & " | " & [Most Common Departure Time]
```

e.g. `"1,484 | 5:45 PM"`.

---

#### `Most Common Arrival Time`
> **Format:** *(text)* · *Single most frequent arrival time for the arrival station(s) in context.*

```dax
VAR Times =
    ADDCOLUMNS(
        VALUES(DIM_TIME[Time]),
        "@Cnt", CALCULATE(
            [Ticket Count],
            USERELATIONSHIP('FACT_TICKET_SALES'[ArrivalStationKey], DIM_STATION[StationKey]),
            USERELATIONSHIP('FACT_TICKET_SALES'[ArrivalTimeKey],    DIM_TIME[TimeKey])
        )
    )
VAR TopTime = TOPN(1, Times, [@Cnt], DESC, DIM_TIME[Time], ASC)
RETURN
    CONCATENATEX(TopTime, FORMAT(DIM_TIME[Time], "h:mm AM/PM"), ", ")
```

Activates both the arrival-station **and** arrival-time inactive relationships simultaneously.

---

#### `Arrival Station Bar Label`
> **Format:** *(text)* · *Combined 'count | time' label for the Top Arrival Stations bar chart.*

```dax
FORMAT([Tickets by Arrival Station], "#,##0") & " | " & [Most Common Arrival Time]
```

e.g. `"1,484 | 6:10 PM"`.

---

## 📄 Report Pages

The report is structured across **four analytical pages**, each aligned to its corresponding measure display folder.

---

### Page 1 — Passenger Analysis

> **Measure folder:** `01 Passenger Analysis`

An in-depth look at passenger demographics, ticket preferences, railcard usage, and travel patterns.

#### Measures Used on This Page

| Measure | Type | Purpose |
|---|---|---|
| `Ticket Count` | KPI | Total passengers in the current selection |
| `Railcard Tickets` | KPI | Raw count of railcard-holder tickets |
| `Railcard Penetration %` | KPI | Share of passengers using a railcard |
| `Avg Price (Railcard Holders)` | KPI | Price comparison — railcard holders |
| `Avg Price (No Railcard)` | KPI | Price comparison — non-railcard holders |
| `First Class Mix %` | KPI | Share of First Class tickets |
| `Advance Booking %` | KPI | Share of Advance-fare tickets |
| `Online Purchase %` | KPI | Share of online purchases |
| `Contactless Payment %` | KPI | Share of contactless payments |
| `Peak Travel Band` | KPI (text) | Most common departure time band |
| `KPI Card Ticket Count HTML` | HTML visual | Styled KPI card — ticket count |
| `KPI Card Railcard Penetration HTML` | HTML visual | Styled KPI card — railcard % |
| `KPI Card First Class Mix HTML` | HTML visual | Styled KPI card — first class % |
| `KPI Card Peak Travel Band HTML` | HTML visual | Styled KPI card — peak band |
| `Railcard Status Icon` | Image visual | Dynamic railcard icon (switches on slicer) |
| `Railcard Status Icon HTML` | HTML visual | HTML wrapper for the railcard icon |
| `Label` | Bar chart label | `actual journeys | departure time` per station |
| `Station Carousel HTML` | HTML visual | Scrolling station name carousel |
| `Last Refresh Badge HTML` | HTML visual | Data freshness badge |
| `Last Refresh Clock HTML` | HTML visual | Large refresh time clock |

#### Key Slicers / Filters

| Field | Table | Purpose |
|---|---|---|
| `MonthName` | `DIM_DATE` | Filter by month |
| `Day Appr` / `DayOfWeek` | `DIM_DATE` | Filter by day of week |
| `Railcard Status` | `DIM_PURCHASE_CHANNEL` | Railcard Holder vs. Non Railcard Holder |
| `Purchase Type` | `DIM_PURCHASE_CHANNEL` | Online vs. Station |
| `Payment Method` | `DIM_PURCHASE_CHANNEL` | e.g. Contactless, Credit Card |
| `Ticket Class` | `DIM_TICKET_TYPE` | Standard Class vs. First Class |
| `Ticket Type` | `DIM_TICKET_TYPE` | Advance, Anytime, Off-Peak |
| `TimeBand` | `DIM_TIME` | Time-of-day band |

---

### Page 2 — Sales Performance

> **Measure folder:** `02 Sales Performance`

Revenue trends, channel performance, and booking lead-time analysis.

#### Key Measures on This Page

| Measure | Format | Purpose |
|---|---|---|
| `Total Revenue` | `£#,##0` | Total revenue in context |
| `Average Ticket Price` | `£#,##0.00` | Avg revenue per ticket |
| `Avg Daily Revenue` | `£#,##0` | Revenue per operating day |
| `Best Day Revenue` | `£#,##0` | Single highest-revenue day |
| `Online Revenue` | `£#,##0` | Revenue via online channel |
| `Station Revenue` | `£#,##0` | Revenue via station channel |
| `Online Revenue Mix %` | `0.0%` | Online share of revenue |
| `Revenue MTD` | `£#,##0` | Month-to-date revenue |
| `Revenue MoM %` | `0.0%` | Month-over-month revenue change |
| `Avg Purchase Lead Time (Days)` | `0.0` | Avg days between purchase and travel |
| `Same Day Purchase %` | `0.0%` | Share of last-minute ticket purchases |
| `Avg Passengers per Day` | `#,##0` | Avg daily passenger volume |
| `Station Purchase Journeys` | `#,##0` | Tickets bought at the station |
| `KPI Card Average Ticket Price HTML` | HTML | Styled KPI card |
| `KPI Card Avg Passengers per Day HTML` | HTML | Styled KPI card |
| `Passenger Trend Area Chart SVG` | HTML | Monthly ticket trend with MoM % arrows |

---

### Page 3 — Railway Performance

> **Measure folder:** `03 Railway Performance`

Operational KPIs covering punctuality, delay analysis, cancellations, and refund exposure.

#### Key Measures on This Page

| Measure | Format | Purpose |
|---|---|---|
| `On Time Journeys` | `#,##0` | Count of on-time journeys |
| `On Time %` | `0.0%` | Punctuality rate |
| `Delayed Journeys` | `#,##0` | Count of delayed journeys |
| `Delayed %` | `0.0%` | Delay rate |
| `Cancelled Journeys` | `#,##0` | Count of cancellations |
| `Cancellation %` | `0.0%` | Cancellation rate |
| `Avg Delay Minutes` | `0.0` | Average lateness in minutes |
| `Total Delay Minutes` | `#,##0` | Total passenger-minutes lost |
| `Top Delay Reason` | text | Most frequent disruption cause |
| `Refund Requests` | `#,##0` | Count of refund requests |
| `Refund Request Rate %` | `0.0%` | Refund request rate |
| `Revenue at Risk (Refunds)` | `£#,##0` | Revenue exposed to refunds |

---

### Page 4 — Route Analysis

> **Measure folder:** `04 Route Analysis`

Route-level performance, station popularity, and journey timing patterns.

#### Key Measures on This Page

| Measure | Format | Purpose |
|---|---|---|
| `Distinct Routes` | `#,##0` | Number of unique origin–destination pairs |
| `Busiest Route` | text | Top route by passenger count |
| `Top Departure Station` | text | Most popular departure station |
| `Avg Tickets per Route` | `#,##0.0` | Avg tickets sold per route |
| `Avg Revenue per Route` | `£#,##0` | Avg revenue per route |
| `Route Yield per Ticket` | `£#,##0.00` | Revenue per ticket (premium-route identifier) |
| `Revenue by Arrival Station` | `£#,##0` | Revenue attributed to arrival station |
| `Tickets by Arrival Station` | `#,##0` | Tickets attributed to arrival station |
| `Most Common Departure Time` | text | Peak departure time for selected station |
| `Station Bar Label` | text | `count \| time` label for departure bar chart |
| `Most Common Arrival Time` | text | Peak arrival time for selected station |
| `Arrival Station Bar Label` | text | `count \| time` label for arrival bar chart |

---

## 🛠 Data Source & Power Query

**Source file:** `railway.csv.xlsx` (single worksheet: `railway`)

The `railway` worksheet contains the original flat-file columns. All dimension tables are derived from it in Power Query, and the fact table is built by merging all six surrogate keys back in. **No data duplication occurs** — dimension data is extracted, deduplicated, and keyed before being referenced by the fact table.

### Source Columns (original `railway` table)

| Column | Used By |
|---|---|
| `Transaction ID` | `FACT_TICKET_SALES` |
| `Date of Purchase` | `DIM_DATE`, `FACT_TICKET_SALES` (PurchaseDateKey) |
| `Date of Journey` | `DIM_DATE`, `FACT_TICKET_SALES` (JourneyDateKey) |
| `Time of Purchase` | `DIM_TIME`, `FACT_TICKET_SALES` (PurchaseTimeKey) |
| `Departure Time` | `DIM_TIME`, `FACT_TICKET_SALES` (DepartureTimeKey) |
| `Arrival Time` | `DIM_TIME`, `FACT_TICKET_SALES` (ArrivalTimeKey) |
| `Actual Arrival Time` | `DIM_TIME`, `FACT_TICKET_SALES` (ActualArrivalTimeKey) |
| `Departure Station` | `DIM_STATION`, `FACT_TICKET_SALES` (DepartureStationKey) |
| `Arrival Destination` | `DIM_STATION`, `FACT_TICKET_SALES` (ArrivalStationKey) |
| `Purchase Type` | `DIM_PURCHASE_CHANNEL`, `FACT_TICKET_SALES` (ChannelKey) |
| `Payment Method` | `DIM_PURCHASE_CHANNEL` |
| `Railcard` | `DIM_PURCHASE_CHANNEL` |
| `Ticket Class` | `DIM_TICKET_TYPE`, `FACT_TICKET_SALES` (TicketTypeKey) |
| `Ticket Type` | `DIM_TICKET_TYPE` |
| `Journey Status` | `DIM_JOURNEY_STATUS`, `FACT_TICKET_SALES` (StatusKey) |
| `Reason for Delay` | `DIM_JOURNEY_STATUS` |
| `Refund Request` | `FACT_TICKET_SALES` |
| `Price` | `FACT_TICKET_SALES` |

### Transformation Summary

| Table | Key Transformations |
|---|---|
| `DIM_DATE` | Union of purchase/journey dates → full date spine → calendar attributes (`Year`, `Quarter`, `MonthNo`, `MonthName`, `Day`, `DayOfWeek`, `IsWeekend`, `Day Appr`) |
| `DIM_TIME` | Union of 4 time columns → truncate to minute → distinct → sort → add `TimeKey` (`HH*100+MM`), `TimeBand`, `AM/PM` |
| `DIM_STATION` | Union of departure / arrival station names → distinct → sort → auto-increment `StationKey` |
| `DIM_PURCHASE_CHANNEL` | Distinct `(Purchase Type, Payment Method, Railcard)` triples → sort → auto-increment `ChannelKey` |
| `DIM_TICKET_TYPE` | Distinct `(Ticket Class, Ticket Type)` pairs → sort → auto-increment `TicketTypeKey` |
| `DIM_JOURNEY_STATUS` | Distinct `(Journey Status, Reason for Delay)` pairs → `Text.Proper` normalisation → deduplicate → auto-increment `StatusKey` |
| `FACT_TICKET_SALES` | Date/time key encoding (`yyyyMMdd`, `HHmm`) → 6 left-outer lookup merges for all surrogate keys → column selection → `Refund Request` boolean conversion (`"Yes"` → `TRUE`, `"No"` → `FALSE`) |

---

*Documentation generated from live Power BI Desktop semantic model via MCP · UK Railways Project · 29 Aug 2026*

