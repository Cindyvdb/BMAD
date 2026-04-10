# Session Notes: Traffic_Stats Analysis & Redesign
**Date:** 2026-04-10  
**Agents involved:** Winston (Architect), Mary (Analyst)  
**Database:** PRODSQL → SRC_SiteTracker  
**Table:** dbo.Traffic_Stats  

---

## Context

This session analysed the existing `dbo.Traffic_Stats` table in the **SRC_SiteTracker** database on **PRODSQL**, assessed architectural weaknesses, identified dependencies, and outlined Power BI visualisation recommendations.

---

## Current Table: dbo.Traffic_Stats

- **121 columns**, **199,474 rows**
- **No primary key, no indexes** — pure heap, every query is a full table scan
- All columns nullable, no defaults
- Key identifiers: `Road_ID` (bigint), `Suburb_ID` (int), `Suburb` (varchar 50), `Prov_ID` (int), `Province` (varchar 40), `Period` (varchar 20), `YearMonth` (int)

### Column Groups
| Group | Columns | Type |
|---|---|---|
| Vehicle segments A–H | 10 cols (e.g. `A: Small - Medium`, `H: Trucks & 3501Kg or Heavier`) | int |
| Age brackets | Age<30, Age30to45, Age45to60, Age>60, Age_UN | int |
| Gender | Customer_M, Customer_F, Customer_U | int |
| Time of day | 5to9, 9to15, 15to20, 20to5 | int |
| Income bands | Income_A through Income_M (13 cols) | int |
| LSM | LSM_1 through LSM_10, LSM_Unknown (12 cols) | decimal |
| Aggregates (redundant) | SUM_TRUCKS, SUM_PASSENGER, SUM_BUS, SUM_MINIBUS, SUM_WORKING, SUM_UNKNOWN_SEGMENT | int |
| Totals | TOTAL_AGE, TOTAL_GENDER, TOTAL_TIME, TOTAL_HH_INCOME, TOTAL_LSM | int |
| Percentages (derived, redundant) | PERC_ prefix — one per demographic count column | numeric |
| Weighted / frequency | Uniqecars_Wgt, Trips_Wgt, Frequency, Once, Twice, Three_Eight, Nine_Twenty, More_Than_20, Factor | decimal/float |
| OOH KPIs | Impact, Likely_Impact, Reach, Likely_Reach, GRP, Likely_GRP | int |

---

## Architectural Issues Identified (Winston)

1. **Wide pivoted schema** — every new vehicle/demographic category requires an `ALTER TABLE ADD COLUMN`
2. **PERC_ columns are derivable** — storing them creates update anomaly risk
3. **SUM_ columns are redundant** — already computable from segment columns
4. **Denormalized geography** — Suburb/Province varchar stored on 200K rows
5. **No PK, no indexes** — heap table; all queries are full scans
6. **Dual time fields** — both `Period` (varchar) and `YearMonth` (int) represent the same dimension

---

## Dependency Analysis

**Only one object references `dbo.Traffic_Stats`:**

### `dbo.usp_publish_media_track` (Stored Procedure)
Monthly ETL loader that:
1. `DELETE` rows for current month (`WHERE YearMonth = @YearMonth`)
2. `INSERT` all 121 columns from staging table `dbo.Temp5`
3. Also refreshes `dbo.Client_Roads` from linked server `SARevealed..Tracker_Road_Client_Sites`
4. Drops 6 temp tables: `Temp1` through `Temp6` (broader upstream pipeline builds these)

**Blast radius: LOW** — only one proc, no views.  
Note: Dynamic SQL references are not captured by `sys.sql_expression_dependencies` — worth a manual text search of other procs if doing a full migration.

---

## Recommended Immediate Actions (no breaking changes)

```sql
-- Step 1: Add surrogate PK (eliminates heap scan)
ALTER TABLE dbo.Traffic_Stats ADD StatID bigint IDENTITY(1,1) NOT NULL;
ALTER TABLE dbo.Traffic_Stats ADD CONSTRAINT PK_Traffic_Stats PRIMARY KEY CLUSTERED (StatID);

-- Step 2: Add two covering indexes for most common access patterns
CREATE INDEX IX_Traffic_Road_YearMonth   ON dbo.Traffic_Stats (Road_ID, YearMonth);
CREATE INDEX IX_Traffic_Suburb_YearMonth ON dbo.Traffic_Stats (Suburb_ID, YearMonth);
```

These are safe — the proc only does `DELETE WHERE YearMonth = @YearMonth` and bulk `INSERT`.

---

## Recommended Long-Term Redesign (Winston)

Normalize to a **star schema**:

```sql
-- Dimension: all demographic/segment categories
CREATE TABLE dbo.Traffic_Dimension (
    DimID    int IDENTITY PRIMARY KEY,
    DimType  varchar(30) NOT NULL,  -- 'VehicleSegment','AgeGroup','Gender','Income','LSM','TimeOfDay'
    DimCode  varchar(10) NOT NULL,
    DimLabel varchar(60) NOT NULL
);

-- Dimension: geography
CREATE TABLE dbo.Geography (
    SuburbID   int PRIMARY KEY,
    SuburbName varchar(50) NOT NULL,
    ProvID     int NOT NULL,
    Province   varchar(40) NOT NULL
);

-- Narrow fact table
CREATE TABLE dbo.Traffic_Stats_v2 (
    StatID      bigint IDENTITY PRIMARY KEY,
    Road_ID     bigint        NOT NULL,
    SuburbID    int           NOT NULL REFERENCES dbo.Geography(SuburbID),
    YearMonth   int           NOT NULL,
    OneWay      tinyint       NOT NULL,
    DimID       int           NOT NULL REFERENCES dbo.Traffic_Dimension(DimID),
    TripCount   int           NOT NULL DEFAULT 0,
    WeightedVal decimal(18,4) NULL,
    INDEX IX_Road_YearMonth (Road_ID, YearMonth),
    INDEX IX_YearMonth_Dim  (YearMonth, DimID)
);
```

**Migration note:** Data arrives pre-pivoted in `Temp5` — an `UNPIVOT` step would be needed in `usp_publish_media_track` when switching to the new schema.

---

## Power BI Visual Recommendations (Mary)

**Domain context:** This is an OOH (Out-of-Home) advertising measurement dataset. Primary audience = media planners and clients.

### Recommended Report Pages

| Page | Key Visuals |
|---|---|
| **1. Executive Summary** | KPI Cards (TotalTrips, Reach, GRP, Impact), Line chart of Impact over YearMonth |
| **2. Audience Profile** | Donut (gender), Column chart (age brackets), Column chart (LSM 1–10 with national benchmark line), Horizontal bar (income bands A–M) — use PERC_ columns for labels |
| **3. Vehicle Segments** | Treemap or 100% Stacked Bar (A–H segments), Scatter Plot (Road_ID × TotalTrips, bubble = SUM_TRUCKS) |
| **4. Geographic Performance** | Filled Map choropleth by Province (color = GRP), drill-through to Suburb level; if lat/long available on Roads, point map of actual road positions |
| **5. Time-of-Day** | Heatmap matrix (Roads × time bands: 5to9 / 9to15 / 15to20 / 20to5) |
| **6. Site Ranking** | Ranked bar charts (Top N roads by TotalTrips, Impact, GRP), Matrix with conditional formatting heatmap (Road × Period → GRP value) |
| **7. Frequency Distribution** | Stacked column (Once / Twice / Three_Eight / Nine_Twenty / More_Than_20) — audience loyalty signal |

### Normalized Schema → Power BI Star Schema Benefit
- One slicer (`DimType`) drives all demographic visuals — no hardcoded column charts per dimension
- PERC_ columns become DAX calculated measures (no storage overhead)
- SUM_ columns become DAX aggregates
- New vehicle/demographic categories appear automatically from the dimension table

---

## Suggested Next Steps

- [ ] Run the `ALTER TABLE` + `CREATE INDEX` scripts immediately (no risk, big gain)
- [ ] Investigate the upstream pipeline (`Temp1`–`Temp6`) to understand where the pivot happens
- [ ] Decide: migrate to normalized schema, or keep wide format and wrap with a view layer for Power BI
- [ ] Search other stored procedures for dynamic SQL referencing `Traffic_Stats` (manual text search)
- [ ] Define DAX measures in Power BI for GRP, Reach, Frequency calculations
- [ ] Explore `dbo.Client_Roads` table — referenced in the same proc, may be relevant to the road dimension

---

## To Resume This Work

Load this file and share with the agent. Then invoke:
- **Winston** (architect) for the schema migration: `bmad-agent-architect`
- **Mary** (analyst) for Power BI requirements: `bmad-agent-analyst`
- `[CA]` Create Architecture to formally document the redesign decision
