# Scope Script — Beginner to Production Guide
### Simple Language · Real Examples · Everything You Need

---

## Before You Start — What is Scope Script?

Imagine you have **billions of rows of advertising data** — every click, every impression, every conversion from Microsoft Ads — stored in files on a giant cluster of computers.

You cannot open that in Excel. You cannot run it on your laptop. You need a special language that runs on **thousands of computers at the same time** to process it fast.

That language is **Scope Script**.

> **Simple analogy:** SQL runs on one database. Scope runs on thousands of computers at once. That is the only big difference in concept. If you know SQL, you already know 70% of Scope.

Scope is used by Microsoft Ads teams every day to:
- Pull campaign performance data (Impressions, Clicks, Revenue, Conversions)
- Calculate metrics like CTR, CPC, ROAS
- Build Power BI dashboards
- Report on PMax, Search, Shopping, Display campaigns

---

## How to Read This Guide

Each section has:
1. **What it is** — simple explanation
2. **Why you need it** — real reason
3. **Syntax** — the actual code pattern
4. **Example** — based on real Microsoft Ads data work

Work through it top to bottom. Do not skip sections.

---

---

# CHAPTER 1 — The Basics

---

## 1.1 Variables — `#DECLARE`

### What is it?
A variable in Scope is a named box that holds a value. You declare it once at the top of the script and use it everywhere below.

### Why do you need it?
Imagine your script runs for the date **2024-03-01**. If you hardcode that date in 20 places and need to change it, you must change it 20 times. With a variable, you change it once.

### Syntax
```sql
#DECLARE  VariableName  datatype  =  value;
```

### Example
```sql
-- A date stored as text
#DECLARE START_DATE  string  = "2024-03-01";

-- A number
#DECLARE Days  int  = 7;

-- A true/false flag
#DECLARE RunPerformance  bool  = true;

-- A file path stored as text
#DECLARE OutputFolder  string  = "/output/ads/reports/";
```

### How to use the variable — put @ before the name
```sql
-- Use @VariableName anywhere in the script
#DECLARE MyDate  string  = "2024-03-01";

-- Use it like this:
OUTPUT @results
TO @OutputFolder;        -- uses the folder declared above
```

---

## 1.2 Runtime Parameters — `@@PARAM@@`

### What is it?
Sometimes the value of a variable is not hardcoded — it is **sent in by the job scheduler** when the script runs automatically every day.

### Why do you need it?
In production, a script runs every morning automatically. The scheduler tells the script "today is 2024-03-15" — the script does not need to know the date in advance.

### Syntax
```sql
-- Double @@ means "the scheduler will fill this in at runtime"
#DECLARE START_DATE  string  =  @@RUN_DATE@@;
#DECLARE Days        int     =  @@Days@@;
```

### When testing manually
Replace `@@RUN_DATE@@` with a real date for testing:
```sql
-- For testing only (comment out in production):
#DECLARE START_DATE  string  = "2024-03-01";

-- For production (uncomment this):
-- #DECLARE START_DATE  string  = @@RUN_DATE@@;
```

---

## 1.3 Calculating New Variables From Old Ones

You can build one variable using another already declared variable.

```sql
-- Declare start date
#DECLARE START_DATE  string  = "2024-03-01";

-- Declare how many days to cover
#DECLARE Days  int  = 7;

-- Calculate end date automatically: start + (days - 1)
-- If start = March 1 and Days = 7, end = March 7
#DECLARE END_DATE  string  =  DateTime.Parse(@START_DATE).AddDays(@Days - 1).ToString("yyyy-MM-dd");

-- Build full DateTime objects (needed for VIEWs)
#DECLARE StartDateTime  DateTime  =  DateTime.Parse(@START_DATE + " 00:00");
#DECLARE EndDateTime    DateTime  =  DateTime.Parse(@END_DATE   + " 23:59");
```

> **Read it like English:** "Parse the start date, add Days-1 to it, format as yyyy-MM-dd."

---

## 1.4 Feature Flags — Turn Script Sections On/Off

### What is it?
A boolean (`true/false`) variable that controls whether a whole section of the script runs.

### Why do you need it?
One script can do many things — pull demand data, pull performance data, build dimension tables. You do not always need all three. Flags let you run only what you need.

```sql
#DECLARE Demand      bool  = true;   -- Run the demand section
#DECLARE Performance bool  = true;   -- Run the performance section
#DECLARE DimTable    bool  = true;   -- Run the dimension table section
```

These flags are used with `#IF` which is explained next.

---

## 1.5 `#IF` / `#ENDIF` — Conditional Sections

### What is it?
Like an if-statement but for the **script structure** itself. The code inside only runs if the condition is true.

### Why do you need it?
If `Performance = false`, skip the entire performance section. No wasted compute, no errors.

```sql
-- Only run this block if the Performance flag is true
#IF(@Performance)

    -- All your performance data code goes here
    PerformanceData =
        SELECT CampaignId, SUM(Clicks) AS Clicks
        FROM @source
        GROUP BY CampaignId;

#ENDIF
```

### `#ERROR` — Crash the Script on Purpose
```sql
-- Guard: if BOTH flags are false, there is nothing to do — crash with a clear message
#IF(!@Demand AND !@Performance)
    #ERROR "At least one of Demand or Performance must be true";
#ENDIF
```

> The `!` means NOT. `!@Demand` means "Demand is false".

### `#IF` Inside a SELECT — Different Columns for Different Runs

This is a powerful trick. When `Days == 1` (single day run), use the declared end date. Otherwise use the Date column from the data.

```sql
SELECT
    #IF(@Days == 1)
        @EndDateTime.ToString("MM/dd/yyyy") AS Date,   -- single day: use declared date
    #ELSE
        Date,                                           -- multi day: use column from data
    #ENDIF
    CustomerId,
    SUM(Revenue) AS Revenue
FROM @data
GROUP BY Date, CustomerId;
```

---

---

# CHAPTER 2 — Reading Data

---

## 2.1 `EXTRACT` — Read a Text File

### What is it?
Reads a plain text file (like CSV or TSV) into a variable. You must tell Scope what columns are in the file and what type each column is.

### Why do you need it?
Mapping files (customer segments, country lists, account manager lists) are stored as plain text files. You read them with EXTRACT.

### Syntax
```sql
@variable =
    EXTRACT  ColumnName1  datatype,
             ColumnName2  datatype,
             ColumnName3  datatype
    FROM  "/path/to/file.txt"
    USING DefaultTextExtractor();
```

### Real Example — Reading a Customer Segment Mapping File
```sql
@CustomerSegments =
    EXTRACT  CustomerId    : int,
             SegmentName   : string
    FROM "/local/Team/BI/Mapping/CustomerSegment.txt"
    USING DefaultTextExtractor();
```

### Data Types You Will Use

| Type | What it stores | Example |
|---|---|---|
| `string` | Any text | `"Search Campaign"` |
| `int` | Whole number (up to ~2 billion) | `12345` |
| `long` | Very large whole number | `9876543210` |
| `double` | Decimal number | `3.14` |
| `decimal` | High-precision decimal (for money) | `99.9999` |
| `bool` | True or False | `true` |
| `DateTime` | A date and time | `2024-03-01` |
| `int?` | Whole number OR null (missing) | `null` or `42` |
| `byte?` | Small number (0-255) OR null | `null` or `5` |
| `sbyte?` | Small signed number OR null | `null` or `-1` |

> The `?` at the end means **nullable** — the value can be missing. Always use nullable types when a column might have empty/missing values, otherwise the script crashes.

---

## 2.2 `SSTREAM` — Read a Structured Stream (Binary File)

### What is it?
A structured stream (`.ss` file) is Microsoft's fast binary file format — like a highly compressed database file. Much faster to read than a text file.

### Why do you need it?
Most big data in Microsoft Ads is stored as `.ss` files because they are much faster to read and write than text. You cannot open them in a text editor.

### Syntax
```sql
-- Read a single .ss file (no schema needed — it is stored inside the file)
@mydata = SSTREAM "/path/to/file.ss";
```

---

## 2.3 `SSTREAM SPARSE STREAMSET` — Read Many Daily Files at Once

### What is it?
This is the most important read pattern in production scripts. It reads **many files across a date range** in one command — all the daily files from March 1 to March 7 for example.

### Why do you need it?
Data is stored in daily folders: one folder per day. To analyse a week of data you would normally need 7 separate reads. STREAMSET does all 7 in one line.

### The Folder Structure on the Cluster
```
/shares/data/Folder/2024/03/01/DataFile.ss
/shares/data/Folder/2024/03/02/DataFile.ss
/shares/data/Folder/2024/03/03/DataFile.ss
... and so on
```

### Syntax
```sql
@mydata =
    SSTREAM SPARSE STREAMSET  @BasePath
        PATTERN  @FilePattern
        RANGE    __date = [@START_DATE, @END_DATE];
```

### Real Example
```sql
#DECLARE BasePath     string = "/shares/bingAds.BI.OI/AdsOI/DemandMetrics/Data/";
#DECLARE FilePattern  string = "UnifiedLayerDemand/%Y/%m/%d/Campaigns.ss";

-- This reads ALL daily Campaigns.ss files from START_DATE to END_DATE
@Campaigns =
    SSTREAM SPARSE STREAMSET @BasePath
        PATTERN @FilePattern
        RANGE __date = [@START_DATE, @END_DATE];
```

### Understanding the Pattern
- `%Y` = 4-digit year (2024)
- `%m` = 2-digit month (03)
- `%d` = 2-digit day (01)

If your range is March 1 to March 3, Scope automatically reads:
```
.../2024/03/01/Campaigns.ss
.../2024/03/02/Campaigns.ss
.../2024/03/03/Campaigns.ss
```

All rows from all three files are combined into one table — `@Campaigns`.

### What does `SPARSE` mean?
If a file is missing for a particular date (weekend, holiday, no data), `SPARSE` skips it instead of crashing. **Always use SPARSE.**

---

## 2.4 `VIEW with PARAMS` — Read a Logical Data Layer

### What is it?
A VIEW is a pre-built data source maintained by a central team. You do not need to know where the physical files are — you just call the VIEW with parameters.

### Why do you need it?
The Monetization View is the core Microsoft Ads data source. It handles impressions, clicks, and conversions. The physical file paths change over time — the VIEW always points to the right place.

### Syntax
```sql
@result =
    VIEW "/path/to/SomeView.view"
    PARAMS
    (
        ParameterName1 = value1,
        ParameterName2 = value2
    );
```

### Real Pattern — Reading Monetization Data (3 Separate Reads)

In Microsoft Ads analysis, you read the same Monetization VIEW three times — once for impressions, once for clicks, once for conversions. Each read has different flags.

```sql
-- READ 1: Impressions only
MVImp =
    VIEW @MonetizationViewPath
    PARAMS
    (
        StartDateTimeUtc   = @StartDateTime,
        EndDateTimeUtc     = @EndDateTime,
        ReadAdImpressions  = true,       -- Read impressions
        ReadClicks         = false,      -- Skip clicks
        ReadConversions    = false,      -- Skip conversions
        FraudType          = "NonFraud"  -- Always filter fraud
    );

-- READ 2: Clicks only
MVClick =
    VIEW @MonetizationViewPath
    PARAMS
    (
        StartDateTimeUtc   = @StartDateTime,
        EndDateTimeUtc     = @EndDateTime,
        ReadAdImpressions  = false,
        ReadClicks         = true,       -- Read clicks
        ReadConversions    = false,
        FraudType          = "NonFraud"
    );

-- READ 3: Conversions only
MVConv =
    VIEW @MonetizationViewPath
    PARAMS
    (
        StartDateTimeUtc    = @StartDateTime,
        EndDateTimeUtc      = @EndDateTime,
        ReadClicks          = false,
        ReadConversions     = true,      -- Read conversions
        ReadUETConversions  = true,      -- Include UET (Universal Event Tracking)
        FraudType           = "NonFraud"
    );
```

> You read it three times because impressions, clicks, and conversions are stored differently internally. Later you merge all three into one fact table.

---

---

# CHAPTER 3 — Transforming Data

---

## 3.1 `SELECT` — The Core of Every Step

### What is it?
Exactly like SQL SELECT. Pick columns, rename them, calculate new ones, filter rows, and group them.

```sql
@result =
    SELECT  ColumnA,
            ColumnB,
            ColumnA + ColumnB  AS Combined,
            SUM(ColumnC)       AS Total
    FROM   @source
    WHERE  ColumnA > 0
    GROUP BY ColumnA, ColumnB;
```

---

## 3.2 Re-assigning the Same Variable — Step-by-Step Refinement

### What is it?
In Scope, you can assign to the same variable name multiple times. Each assignment **refines** the data step by step. The execution engine figures out the right order.

### Why this is important
Production scripts do this constantly. `Campaigns` might be assigned 4 or 5 times — each time it gets cleaner and more useful.

```sql
-- Step 1: Read raw data
Campaigns =
    SSTREAM SPARSE STREAMSET @BasePath PATTERN @Pattern RANGE __date = [@START_DATE, @END_DATE];

-- Step 2: Pick only the columns you need and clean types
Campaigns =
    SELECT DISTINCT
        __date                      AS Date,
        AccountId,
        Convert.ToInt64(CampaignId) AS CampaignId,    -- convert int to long
        AdvertisingChannelTypeId,
        BiddingSchemeId,
        IsEligible,
        IsServable
    FROM Campaigns;

-- Step 3: Join with account info to get customer details
Campaigns =
    SELECT  B.Date,
            A.CustomerId,
            A.AccountId,
            B.CampaignId,
            B.AdvertisingChannelTypeId,
            B.BiddingSchemeId,
            B.IsEligible,
            B.IsServable
    FROM Accounts AS A
    INNER JOIN Campaigns AS B
    ON A.AccountId == B.AccountId;
```

> Read each `Campaigns =` as "now Campaigns is this". It builds up like layers.

---

## 3.3 Aggregation Functions

### `SUM()` — Add up values
```sql
SUM(Impressions)  AS TotalImpressions
SUM(Revenue)      AS TotalRevenue
```

### `COUNT()` — Count all rows in a group (no argument in Scope)
```sql
COUNT()  AS TotalRows
```

### `COUNT(DISTINCT column)` — Count unique values
```sql
COUNT(DISTINCT CampaignId)  AS UniqueCampaigns
```

### `COUNTIF(condition)` — Count rows where a condition is true ⭐ Scope-specific
This does NOT exist in standard SQL. Scope's most useful special function.

```sql
-- Count how many campaigns are PMax (ChannelTypeId 9) per customer
SELECT
    CustomerId,
    COUNTIF(AdvertisingChannelTypeId == 9)  AS PMaxCampaigns,
    COUNTIF(AdvertisingChannelTypeId == 1)  AS SearchCampaigns,
    COUNT()                                  AS TotalCampaigns
FROM @Campaigns
GROUP BY CustomerId;
```

### `FIRST()` — Take the first value in a group ⭐ Scope-specific
Use this when you `GROUP BY` an ID but need to keep a name column that is the same for all rows in the group.

```sql
-- Group by CustomerId, keep the CustomerName (same for all rows of that customer)
SELECT
    CustomerId,
    FIRST(CustomerName)  AS CustomerName,
    FIRST(Segment)       AS Segment
FROM @raw
GROUP BY CustomerId;
```

### `MAX()`, `MIN()`, `AVG()` — Standard aggregates
```sql
MAX(Revenue)      AS HighestRevenue
MIN(AdSpend)      AS LowestSpend
AVG(ClickCount)   AS AverageClicks
```

---

## 3.4 `IF()` — Conditional Value Inside SELECT ⭐

### What is it?
`IF(condition, value_if_true, value_if_false)` — picks one of two values based on a condition. Used constantly in production for labelling and normalising IDs.

```sql
-- If campaign is PMax (type 9), keep the AdDisplayTypeId, otherwise use -9999
SELECT
    IF(CampaignTypeId == 9, AdDisplayTypeId, -9999)  AS AdDisplayTypeId

-- If campaign is Smart Shopping (type 3, subtype 12), use 312 as a combined ID
    IF(CampaignTypeId == 3 AND CampaignSubTypeId == 12, 312, CampaignTypeId)  AS NormalizedCampaignTypeId

-- Boolean flag: is this a PMax campaign?
    IF(AdvertisingChannelTypeId == 9, true, false)  AS IsPMax

FROM @data;
```

### Nested IF — Multiple Conditions
When you have more than two options, nest IF inside IF:

```sql
-- If Smart Shopping → 312, If PMax → keep original ID, else → -9999
IF(IsSS,   312,
IF(IsPMax, AdvertisingChannelTypeId,
           -9999))
AS NormalizedChannelId
```

> Read it from the outside in: "If SS use 312, otherwise check if PMax..."

---

## 3.5 `??` — Null Safety Operator ⭐ Very Important

### What is it?
`column ?? default_value` means: "if this column is null, use the default value instead."

### Why do you need it?
When you do a LEFT OUTER JOIN (explained in Chapter 4), some rows from the right table will not have a match. Those columns will be **null**. You must handle this or your output will have null values.

```sql
-- After a LEFT JOIN, some customers might not have a segment mapped
SELECT
    CustomerId,
    CustomerName     ?? "Unknown"       AS CustomerName,
    Segment          ?? "Unsegmented"   AS Segment,
    ServiceCountry   ?? "Unknown"       AS ServiceCountry,
    AccountExecutive ?? "Unassigned"    AS AE,
    AccountManager   ?? "Unassigned"    AS AM
FROM @joined_data;
```

---

## 3.6 `Convert.*` — Change a Column's Data Type

### Why do you need it?
One table stores `CampaignId` as `int`, another stores it as `long`. If you try to JOIN them, Scope throws a type mismatch error. You must convert one to match the other.

```sql
-- Convert int to long (needed for large CampaignIds)
Convert.ToInt64(CampaignId)    AS CampaignId

-- Convert long to int
Convert.ToInt32(CustomerId)    AS CustomerId

-- Convert string date to DateTime for date comparisons
Convert.ToDateTime(DateString) AS Date

-- Convert DateTime to formatted string for output
Convert.ToDateTime(MBDate).ToString("MM/dd/yyyy")  AS Date
```

### Literal Type Casting
When you need a zero in a specific type (for UNION ALL — explained below):

```sql
(long)0        AS Impressions    -- zero as a long integer
(decimal)0.0   AS Conversions    -- zero as a decimal
```

---

## 3.7 `WHERE` — Filter Rows

```sql
-- Basic filter
WHERE Revenue > 0

-- Multiple conditions (AND)
WHERE AdSpend >= 50 AND Clicks > 0

-- OR condition
WHERE AdvertisingChannelTypeId == 1 OR AdvertisingChannelTypeId == 9

-- Filter a list of values — IN()
WHERE AdvertisingChannelTypeId IN (1, 3, 6, 9)

-- Include nulls in IN list (null can mean "same as Search" in some data)
WHERE AdvertisingChannelTypeId IN (null, 1, 3, 6, 9) AND IsServable

-- Fraud filter — ALWAYS include this when reading ad data
WHERE FraudQualityBand >= 2
```

> **FraudQualityBand >= 2** is mandatory. Band 0 and 1 are fraudulent clicks/impressions. Never remove this filter.

---

## 3.8 `SELECT DISTINCT` — Remove Duplicate Rows

```sql
-- Get unique customer-account-campaign combinations
@unique =
    SELECT DISTINCT
        CustomerId,
        AccountId,
        CampaignId
    FROM @raw;
```

---

## 3.9 `UNION ALL` — Combine Two Tables into One ⭐ Critical Pattern

### What is it?
Takes two tables with the same columns and stacks them vertically.

### Why do you need it?
Impressions and Clicks come from separate VIEW reads. To build one fact table with both, you union them and then aggregate (SUM) to collapse them.

### The Standard Impression + Click Merge Pattern

**Step 1 — Give impressions zero-value click columns:**
```sql
@imp_rows =
    SELECT  Date, CustomerId, CampaignId,
            Impressions,
            (long)0     AS Clicks,        -- no clicks in this read
            Revenue,
            (decimal)0  AS Conversions    -- no conversions in this read
    FROM @MVImp;
```

**Step 2 — Give clicks zero-value impression columns:**
```sql
@clk_rows =
    SELECT  Date, CustomerId, CampaignId,
            (long)0     AS Impressions,   -- no impressions in this read
            Clicks,
            Revenue,
            Conversions
    FROM @MVClick;
```

**Step 3 — Stack them with UNION ALL:**
```sql
@combined =
    SELECT * FROM @imp_rows
    UNION ALL
    SELECT * FROM @clk_rows;
```

**Step 4 — SUM everything to get one row per campaign:**
```sql
@fact =
    SELECT  Date, CustomerId, CampaignId,
            SUM(Impressions) AS Impressions,    -- zeros cancel out
            SUM(Clicks)      AS Clicks,         -- zeros cancel out
            SUM(Revenue)     AS Revenue,
            SUM(Conversions) AS Conversions
    FROM @combined
    GROUP BY Date, CustomerId, CampaignId;
```

> The zeros cancel out perfectly. Each campaign ends up with one row containing real impressions AND real clicks.

### `UNION` vs `UNION ALL`
- `UNION ALL` = keep everything including duplicates (faster, use this)
- `UNION` = remove duplicate rows (slower, use only when deduplication is needed)

---

---

# CHAPTER 4 — Joining Tables

---

## 4.1 `INNER JOIN` — Only Rows That Match in Both Tables

```sql
-- Only campaigns that exist in BOTH the campaign table AND the accounts table
@result =
    SELECT  A.CustomerId,
            A.CustomerName,
            B.CampaignId,
            B.Revenue
    FROM @Accounts   AS A
    INNER JOIN @Campaigns AS B
    ON A.AccountId == B.AccountId;
```

> Rows that exist in one table but not the other are **dropped**. Use INNER JOIN when you only want confirmed matches.

---

## 4.2 `LEFT OUTER JOIN` — Keep All Rows from Left Table ⭐

```sql
-- Keep all campaigns. If no segment mapping exists, Segment will be null
@result =
    SELECT  A.CampaignId,
            A.Revenue,
            B.Segment ?? "Unknown"  AS Segment
    FROM @Campaigns      AS A
    LEFT OUTER JOIN @Segments AS B
    ON A.CustomerId == B.CustomerId;
```

> Always pair LEFT OUTER JOIN with `??` to handle the nulls on unmatched rows.

---

## 4.3 Multi-Table LEFT JOIN — Building a Dimension Table

This pattern joins many mapping files onto one customer list. Each mapping gives extra attributes.

```sql
@DIM_Customer =
    SELECT  A.CustomerId,
            B.CustomerName   ?? "Unknown"     AS CustomerName,
            C.Segment        ?? "Unsegmented" AS Segment,
            D.L1VerticalName ?? "Unknown"     AS Vertical,
            E.ServiceCountry ?? "Unknown"     AS ServiceCountry,
            F.AccountExec    ?? "Unassigned"  AS AE,
            G.AccountManager ?? "Unassigned"  AS AM
    FROM @CustomerList        AS A
    LEFT OUTER JOIN @NameView     AS B ON A.CustomerId == B.CustomerId
    LEFT OUTER JOIN @SegmentMap   AS C ON A.CustomerId == C.CustomerId
    LEFT OUTER JOIN @VerticalMap  AS D ON A.CustomerId == D.CustomerId
    LEFT OUTER JOIN @CountryMap   AS E ON A.CustomerId == E.CustomerId
    LEFT OUTER JOIN @AEMap        AS F ON A.CustomerId == F.CustomerId
    LEFT OUTER JOIN @AMMap        AS G ON A.CustomerId == G.CustomerId;
```

After this join, always deduplicate with FIRST() in case any mapping file had duplicate rows:

```sql
@DIM_Customer =
    SELECT  CustomerId,
            FIRST(CustomerName)    AS CustomerName,
            FIRST(Segment)         AS Segment,
            FIRST(Vertical)        AS Vertical,
            FIRST(ServiceCountry)  AS ServiceCountry,
            FIRST(AE)              AS AE,
            FIRST(AM)              AS AM
    FROM @DIM_Customer
    GROUP BY CustomerId;
```

---

## 4.4 `[SKEWJOIN]` — Fix Slow Joins with Uneven Data ⭐

### What is it?
A performance hint that tells Scope how to handle joins where one key appears millions of times (like a click ID from a massive advertiser).

### Why do you need it?
The Click-to-Conversion join (joining by RGUID + ClickId) is skewed because top advertisers have millions of clicks under the same campaign. Without this hint, the join runs on one overloaded machine and takes forever or crashes.

```sql
[SKEWJOIN=(SKEW=FROMLEFT, REPARTITION=FULLJOIN)]
@ClickConv =
    SELECT  A.Date,
            A.CustomerId,
            A.CampaignId,
            A.Clicks,
            A.Revenue,
            B.Conversions,
            B.GrossProfit
    FROM @MVClick AS A
    LEFT OUTER JOIN @MVConv AS B
    ON A.RGUID == B.RGUID AND A.ClickId == B.ClickId;
```

> `SKEW=FROMLEFT` means the left table (clicks) has the skewed data. `REPARTITION=FULLJOIN` means both sides get redistributed across machines to balance it.

---

## 4.5 The Click-to-Conversion Join Pattern ⭐

### How conversions are connected to clicks
Every click gets a `RGUID` (request ID) and a `ClickId`. When a user later converts (buys something), that conversion record is tagged with the same `RGUID` and `ClickId`. You join on BOTH to connect them.

**Step 1 — Cap conversions at 1 per click (prevent double counting):**
```sql
@ConvDeduped =
    SELECT  RGUID,
            ClickId,
            IF(SUM(ConversionCredit) > 1, 1, SUM(ConversionCredit))  AS Conversions,
            MAX(AdvertiserReportedRevenue)                             AS GrossProfit
    FROM @MVConv
    WHERE FraudQualityBand >= 2
    GROUP BY RGUID, ClickId;
```

> `IF(SUM > 1, 1, SUM)` caps it at 1 — one click should only produce at most 1 conversion credit.

**Step 2 — LEFT JOIN clicks to capped conversions:**
```sql
[SKEWJOIN=(SKEW=FROMLEFT, REPARTITION=FULLJOIN)]
@ClickConv =
    SELECT  A.Date,
            A.CustomerId,
            A.CampaignId,
            A.Clicks,
            A.Revenue,
            B.Conversions ?? (decimal)0.0  AS Conversions,
            B.GrossProfit ?? (decimal)0.0  AS GrossProfit
    FROM @MVClick  AS A
    LEFT OUTER JOIN @ConvDeduped AS B
    ON A.RGUID == B.RGUID AND A.ClickId == B.ClickId;
```

---

---

# CHAPTER 5 — Writing Results

---

## 5.1 `OUTPUT` to Text File — For Power BI and Reporting

```sql
-- Privacy annotation is required (explained in Chapter 6)
[Privacy.Asset.NonPersonal]
OUTPUT @MyData
TO "/output/PowerBi/FACT_Performance.txt"
WITH STREAMEXPIRY "30"
USING DefaultTextOutputter(outputHeader: true);
```

- `outputHeader: true` = write column names in the first row
- `STREAMEXPIRY "30"` = delete this file after 30 days automatically

---

## 5.2 `OUTPUT` to SSTREAM — For Other Scripts to Read

```sql
[Privacy.Asset.NonPersonal]
OUTPUT @MyData
TO SSTREAM "/output/Performance/data.ss"
    CLUSTERED BY CustomerId, CampaignId
    SORTED BY CustomerId, CampaignId
    WITH STREAMEXPIRY "30";
```

- `CLUSTERED BY` = spread data across cluster machines by these columns (parallel read performance)
- `SORTED BY` = within each cluster, sort by these columns (efficient range queries)

### When to use which output

| Use Case | Output Type |
|---|---|
| Power BI report | Text file with `DefaultTextOutputter` |
| Excel download | Text file with `DefaultTextOutputter` |
| Another Scope script reads it | SSTREAM |
| Intermediate data (not final) | SSTREAM with 7-day expiry |
| Final output | Text or SSTREAM with 30-day expiry |

---

## 5.3 `STREAMEXPIRY` — How Long to Keep Files

Files stored on the cluster are not free. They cost compute resources. Always set an expiry.

```sql
#DECLARE FinalExpiry        string = "30";  -- Final outputs: 30 days
#DECLARE IntermediateExpiry string = "7";   -- Intermediate: 7 days

-- Use the variable
OUTPUT @final_data
TO SSTREAM @FinalPath
WITH STREAMEXPIRY @FinalExpiry;
```

---

---

# CHAPTER 6 — Advanced Production Concepts

---

## 6.1 Privacy Annotations — Required on Every OUTPUT ⭐

### What is it?
Every `OUTPUT` must have a privacy tag above it. This is enforced by Microsoft's compliance system. A script without privacy annotations will fail review or fail to run.

```sql
-- Aggregated data with no individual-level rows → NonPersonal
[Privacy.Asset.NonPersonal]
OUTPUT @AggregatedData
TO SSTREAM @OutputPath
WITH STREAMEXPIRY "30";
```

| Annotation | Use When |
|---|---|
| `[Privacy.Asset.NonPersonal]` | Data is aggregated — no individual user rows |
| `[Privacy.Asset.Personal]` | Data contains user-identifiable information |
| `[Privacy.Asset.Pseudonymous]` | Data has hashed/pseudonymous IDs |

> In most advertising analytics, you aggregate to CustomerId/CampaignId level, so outputs are `NonPersonal`.

---

## 6.2 `[LOWDISTINCTNESS(...)]` — Column Privacy Annotation

### What is it?
When a column has very few unique values (like `CampaignTypeId` which has only ~15 possible values), you annotate it so the privacy scanner knows it is not a unique identifier.

```sql
-- Annotate before the SELECT that uses low-distinctness columns
[LOWDISTINCTNESS(CampaignTypeId, MediumId, MarketplaceClassificationId, Date, CustomerId)]
@MVImp =
    SELECT  Date,
            CustomerId,
            CampaignTypeId,       -- only ~15 unique values
            MediumId,             -- only a few unique values
            MarketplaceClassificationId,
            SUM(Impressions)  AS Impressions,
            SUM(Revenue)      AS Revenue
    FROM @MVImp
    WHERE FraudQualityBand >= 2;
```

---

## 6.3 `EXISTS()` — Check if a File Exists Before Reading ⭐

### Why do you need it?
Scripts that run daily **append** to a history file. On the very first run, that history file does not exist yet. Without `EXISTS()`, the script would crash trying to read a non-existent file.

```sql
#IF(@AddHistory AND EXISTS(@HistoryFile))

    @FinalOutput =
        -- Today's fresh data
        SELECT * FROM @TodayData

        UNION ALL

        -- Old history MINUS today's date range (to avoid duplicates)
        SELECT *
        FROM
        (
            EXTRACT  Date         : string,
                     CustomerId   : int?,
                     CampaignTypeId : uint,
                     Impressions  : long,
                     Clicks       : long,
                     Revenue      : decimal,
                     Conversions  : decimal
            FROM @HistoryFile
            USING DefaultTextExtractor()
        )
        WHERE !(   Convert.ToDateTime(Date) >= Convert.ToDateTime(@START_DATE)
               AND Convert.ToDateTime(Date) <= Convert.ToDateTime(@END_DATE));

#ENDIF
```

### The Incremental Append Pattern — Explained Simply

Think of a history file as a growing archive:

```
Day 1 run: History = [Day1 data]
Day 2 run: History = [Day1 data] + [Day2 data]
Day 3 run: History = [Day1 data] + [Day2 data] + [Day3 data]

If Day 2 reruns: 
  History = [Day1 data] + [Fresh Day2 data] + [Day3 data]
  (old Day2 is excluded by the WHERE filter, fresh Day2 replaces it)
```

---

## 6.4 `#CS` / `#ENDCS` — Write C# Helper Functions ⭐

### What is it?
When Scope's built-in functions cannot do what you need, you write a C# function directly in the script. It becomes callable like any built-in function.

### Why do you need it?
Converting numeric IDs to readable labels (BiddingSchemeId 6 → "Target ROAS"), or processing asset type IDs into categories — this logic needs a lookup table that is easiest to write in C#.

### Syntax — Put `#CS` at the Bottom of Your Script
```sql
#CS
// Must be static
// Handle nullable types (byte? not byte)
static string GetBiddingScheme(byte? schemeId)
{
    switch (schemeId)
    {
        case 1:  return "Manual";
        case 2:  return "Max Clicks";
        case 3:  return "Max Conversions";
        case 4:  return "Target CPA";
        case 5:  return "Enhanced CPC";
        case 6:  return "Target ROAS";
        case 7:  return "Max ROAS";
        case 8:  return "Max Conversion Value";
        default: return "Unknown";
    }
}

static string GetChannelName(int channelTypeId)
{
    switch (channelTypeId)
    {
        case 1:   return "Search";
        case 3:   return "Shopping";
        case 6:   return "Smart";
        case 9:   return "PMax";
        case 312: return "Smart Shopping";
        default:  return "Unknown";
    }
}

static string GetAssetType(string assetTypeId)
{
    if (assetTypeId == null) return "";

    switch (assetTypeId)
    {
        case "32": return "Headline";
        case "33": return "Long Headline";
        case "34": return "Description";
        case "36": return "CTA";
        default:   return "Image";
    }
}
#ENDCS
```

### Call C# Functions in SELECT — Just Like Built-in Functions
```sql
@labelled =
    SELECT  CustomerId,
            CampaignId,
            GetChannelName(CampaignTypeId)      AS ChannelName,
            GetBiddingScheme(BiddingSchemeId)   AS BiddingStrategy
    FROM @data;
```

### Rules for `#CS` blocks
- All functions must be `static`
- Place the `#CS` block at the very **bottom** of the script (after all OUTPUTs)
- Use `byte?` / `int?` (nullable) for any column that might be null
- Handle `null` explicitly — `if (value == null) return "";`
- Use C# syntax inside — not Scope syntax

---

## 6.5 The Adoption / Penetration Metric Pattern

### What is it?
A business metric that answers: "What percentage of customers who run any ad campaign also run a PMax campaign?"

### The Two-Step Pattern

```sql
-- STEP 1: Per customer per account — flag PMax and get revenue
@AccountLevel =
    SELECT  Date,
            CustomerId,
            AccountId,
            COUNTIF(AdvertisingChannelTypeId == 9)  AS PMaxCount,
            SUM(Revenue)                             AS Revenue
    FROM @AllCampaigns
    GROUP BY Date, CustomerId, AccountId;

-- STEP 2: Roll up to customer level
-- Numerator   = accounts running at least 1 PMax campaign
-- Denominator = all accounts (regardless of channel)
@Adoption =
    SELECT  Date,
            CustomerId,
            COUNTIF(PMaxCount > 0)               AS Numerator,
            SUM(PMaxCount > 0 ? Revenue : 0)     AS NumeratorRevenue,
            COUNT()                               AS Denominator,
            SUM(Revenue)                          AS DenominatorRevenue
    FROM @AccountLevel
    GROUP BY Date, CustomerId;
```

**Adoption Rate** = `Numerator / Denominator`
**SUI (Spend Under Influence)** = `NumeratorRevenue / DenominatorRevenue`

---

---

# CHAPTER 7 — Working Fast in Production

---

## 7.1 How to Read Any Production Script

Follow these 6 steps when you open any script:

**Step 1:** Read the header comment block — understand what the script does in plain English.

**Step 2:** Find all `#DECLARE` at the top — these are all the inputs, dates, flags, and file paths.

**Step 3:** Find `#IF(@SomeBoolFlag)` blocks — understand which sections are active.

**Step 4:** Find `SSTREAM`, `VIEW`, and `EXTRACT` — these are where data enters the script.

**Step 5:** Follow the variable name through its re-assignments step by step — each `Campaigns =` refines it.

**Step 6:** Find all `OUTPUT` statements at the bottom — these are what the script produces.

---

## 7.2 Standard Script Structure

Always structure your script in this order:

```
1.  MODULE imports (privacy module)
2.  #DECLARE runtime parameters (@@RUN_DATE@@, @@Days@@)
3.  #DECLARE boolean flags (Demand, Performance, DimTable)
4.  #IF guard / #ERROR (crash early if flags are invalid)
5.  #DECLARE computed dates (END_DATE, StartDateTime, EndDateTime)
6.  #DECLARE project paths (BasePath, FinalFileFolder)
7.  #DECLARE all input paths
8.  #DECLARE all output paths

9.  #IF(@Demand)
        Read demand data (SSTREAM STREAMSET)
        Filter and clean
        Join with account/customer data
        Calculate adoption metrics
        OUTPUT demand results
    #ENDIF

10. #IF(@Performance)
        VIEW read for impressions
        VIEW read for clicks
        VIEW read for conversions
        Deduplicate conversions
        SKEWJOIN clicks to conversions
        UNION ALL impressions and clicks
        GROUP BY to get final fact table
        OUTPUT performance results
    #ENDIF

11. #IF(@DimTable)
        EXTRACT mapping files (segment, country, AE, AM)
        FIRST() deduplicate each mapping
        Multi-LEFT JOIN dimension build
        FIRST() deduplicate final dimension
        OUTPUT dimension table
    #ENDIF

12. #CS
        C# helper functions
    #ENDCS
```

---

## 7.3 Debugging Tips

```sql
-- TIP 1: Print a calculated value during compile (to verify dates)
#DECLARE END_DATE string = DateTime.Parse(@START_DATE).AddDays(6).ToString("yyyy-MM-dd");
//#ERROR @END_DATE;   -- Uncomment to see the calculated value, comment back when done

-- TIP 2: Check row counts at any step
@check = SELECT COUNT() AS RowCount FROM @Campaigns;
OUTPUT @check TO "/debug/campaign_count.txt" USING DefaultTextOutputter(outputHeader: true);

-- TIP 3: Take only a sample while testing (remove in production)
@sample = SELECT TOP 1000 * FROM @HugeTable;

-- TIP 4: Check if file exists before reading
#IF(EXISTS(@MyHistoryFile))
    // read it
#ELSE
    #ERROR "History file not found - is this the first run?";
#ENDIF
```

---

## 7.4 Common Errors and How to Fix Them

| Error | Cause | Fix |
|---|---|---|
| Type mismatch on JOIN | `CampaignId` is `int` in one table, `long` in another | Add `Convert.ToInt64(CampaignId)` before the JOIN |
| Null reference crash | Column is null and you did math on it | Use `?? 0` default or `int?` nullable type |
| Divide by zero | Calculating CTR or ROAS when denominator is 0 | `(denom == 0 ? 0.0 : numerator / denom)` |
| Script too slow / skew error | JOIN has uneven data on one key | Add `[SKEWJOIN]` hint |
| File not found crash | Daily file missing for a date | Use `SPARSE` in STREAMSET |
| Privacy annotation error | OUTPUT has no `[Privacy.Asset.*]` above it | Add `[Privacy.Asset.NonPersonal]` |
| Duplicate rows in output | Multiple LEFT JOINs created multiplied rows | Add `FIRST()` + `GROUP BY` at the end |
| `#ERROR` fires unexpectedly | Boolean flag condition is wrong | Check your `#DECLARE` flag values |

---

## 7.5 Complete Quick Reference

### Scope-Only Functions (Not in Standard SQL)

| Function | What it does | Example |
|---|---|---|
| `COUNTIF(cond)` | Count rows where condition is true | `COUNTIF(ChannelType == 9)` |
| `COUNT()` | Count all rows (no argument) | `COUNT()` |
| `FIRST(col)` | Take first value in group | `FIRST(CustomerName)` |
| `IF(c, a, b)` | Return a if c is true, else b | `IF(IsPMax, 9, -1)` |
| `EXISTS(path)` | True if file exists | `#IF(EXISTS(@file))` |
| `a ?? b` | Use b if a is null | `Name ?? "Unknown"` |
| `c ? a : b` | C# ternary — same as IF() | `IsPMax ? Revenue : 0` |

### Preprocessor Reference

| Directive | What it does |
|---|---|
| `#DECLARE name type = value` | Declare a script-level variable |
| `@@PARAM@@` | Value injected by scheduler at runtime |
| `#IF(cond)` / `#ENDIF` | Compile code only if condition is true |
| `#ERROR "msg"` | Stop script and show message |
| `#CS` / `#ENDCS` | Write inline C# functions |

### Data Source Reference

| Source | When to use |
|---|---|
| `EXTRACT ... FROM "path" USING DefaultTextExtractor()` | Plain text / mapping files |
| `SSTREAM "path.ss"` | Single binary structured stream |
| `SSTREAM SPARSE STREAMSET @base PATTERN @pat RANGE __date = [x,y]` | Multiple daily files |
| `VIEW "path.view" PARAMS(...)` | Pre-built logical data layer |

### Output Reference

| Output | When to use |
|---|---|
| `TO "path.txt" USING DefaultTextOutputter(outputHeader:true)` | Power BI, reports, human-readable |
| `TO SSTREAM "path.ss" CLUSTERED BY ... SORTED BY ...` | Another script reads it, fast storage |

---

*This guide covers every concept from the real production Microsoft Ads PMax Performance script:*
*`#DECLARE` · `@@runtime params@@` · `#IF/#ENDIF/#ERROR` · `SSTREAM SPARSE STREAMSET` · `VIEW with PARAMS` · `EXTRACT` · `COUNTIF` · `FIRST` · `IF()` · `??` · `IN()` · `Convert.*` · `(long)0 cast` · `UNION ALL merge` · `INNER JOIN` · `LEFT OUTER JOIN` · `SKEWJOIN` · `Privacy.Asset` · `LOWDISTINCTNESS` · `EXISTS()` · `#CS/#ENDCS` · `STREAMEXPIRY` · Incremental append · Click-Conversion join · Impression-Click merge · Adoption metric · Dimension table build*
