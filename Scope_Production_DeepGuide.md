# Scope Script — Real Production Guide
### Based on Real Microsoft Ads Big Data Script Patterns
### Covers Every Concept You Need to Work Fast and Correctly

---

> **Note:** This guide is written purely from the code patterns — no actual data, paths, client names, or business values are stored or referenced. All examples are generic educational reconstructions.

---

## What This Guide Covers

When you open a real production Scope script, you will see things that are **NOT in any beginner tutorial** — `#DECLARE`, `#IF`, `SSTREAM`, `SPARSE STREAMSET`, `VIEW`, `SKEWJOIN`, `Privacy.Asset`, `#CS`, `FIRST()`, `COUNTIF()`, `??` operator, `EXISTS()` and more.

This guide explains **every single one of them** — what it is, why it exists, and how to use it correctly — so you can read, write, and debug production scripts fast.

---

## PART 1 — Script Header and Parameters

---

### 1.1 Script Header Block (Comments)

Every production script starts with a standard comment block. This is just documentation but teams enforce it strictly. Never skip it.

```sql
/**********************************************************
1. Script Name    : ProjectName_Version_Date
2. Client Name    : Client
3. Project Name   : Project
4. Author         : Your Name (alias)
5. Reviewer       : Reviewer Name (alias)
6. Description    : What this script does
7. Stream Expiry  : 30 days
8. GIT Location   : /Team/Project/Scripts/
9. Script Name    : ScriptName.script
***********************************************************/
```

---

### 1.2 `#DECLARE` — Script-Level Variables

`#DECLARE` creates a variable for the whole script. It is a **preprocessor declaration** — it is resolved before the script runs. Think of it as a find-and-replace that happens at compile time.

```sql
#DECLARE VariableName  datatype = value;
```

**Real usage patterns:**

```sql
// Simple string
#DECLARE START_DATE string = "2024-03-01";

// Integer
#DECLARE Days int = 7;

// Boolean flag
#DECLARE Performance bool = true;

// Build a path dynamically using string.Format
#DECLARE OutputFolder string = "/output/project/";
#DECLARE OutputFile   string = string.Format(@"{0}/results_{1}.txt", @OutputFolder, @START_DATE);

// Compute a date from another declared variable
#DECLARE END_DATE string = DateTime.Parse(@START_DATE).AddDays(@Days - 1).ToString("yyyy-MM-dd");

// Build a full DateTime object
#DECLARE StartDateTimeUtc DateTime = DateTime.Parse(@START_DATE + " 00:00");
#DECLARE EndDateTimeUtc   DateTime = DateTime.Parse(@END_DATE   + " 23:59");
```

**Runtime Parameters** — passed in by the job scheduler at runtime:

```sql
// @@ means "this value is injected at runtime by the job runner"
#DECLARE START_DATE string = @@RUN_DATE@@;
#DECLARE Days       int    = @@Days@@;
```

> When you run the script manually during testing, replace `@@RUN_DATE@@` with a hardcoded date like `"2024-03-01"`. When it runs in production via the scheduler, the runner injects the value automatically.

---

### 1.3 `#IF` / `#ELSE` / `#ENDIF` / `#ERROR` — Preprocessor Conditionals

These control **which parts of the script compile and run**. They are evaluated before execution — not at query time. Use them to turn sections on/off with boolean flags.

```sql
#DECLARE RunDemand     bool = true;
#DECLARE RunPerformance bool = false;

// Guard — crash the script immediately if both flags are false
#IF(!@RunDemand AND !@RunPerformance)
    #ERROR "You must enable at least one section";
#ENDIF

// Conditional code block — only compiles if RunDemand is true
#IF(@RunDemand)
    // ... demand data processing code ...
#ENDIF

// Inline conditional SELECT — different column based on Days flag
SELECT
    #IF(@Days == 1)
        @EndDateTimeUtc.ToString("MM/dd/yyyy") AS Date,
    #ELSE
        Date,
    #ENDIF
    CustomerId
FROM @MyData;
```

**Why this matters:**
- One script can handle multiple scenarios (daily run vs range run, demand only vs performance only)
- Unused sections are completely skipped — no wasted compute cost
- `#ERROR` makes the script fail fast with a clear message instead of silently producing wrong output

---

## PART 2 — Reading Data

---

### 2.1 `EXTRACT` — Read a Flat File

Used when data is in a plain text file (CSV, TSV, custom delimiter). You must define every column name and its type.

```sql
@mydata =
    EXTRACT ColumnA  string,
            ColumnB  int,
            ColumnC  double
    FROM    "/path/to/file.txt"
    USING   DefaultTextExtractor();
```

**Type reference:**

| Scope Type | Meaning | Example value |
|---|---|---|
| `string` | Text | `"Search"` |
| `int` | 32-bit integer | `12345` |
| `long` | 64-bit integer | `9876543210` |
| `double` | Decimal | `3.14` |
| `decimal` | High-precision decimal | `99.9999` |
| `bool` | True/False | `true` |
| `DateTime` | Date and time | `2024-03-01` |
| `byte?` | Nullable byte | `null` or `5` |
| `int?` | Nullable int | `null` or `42` |
| `sbyte?` | Nullable signed byte | `-1` or `null` |

**Nullable types** — the `?` suffix means the value can be null (missing). Always use nullable when a column might have missing values, otherwise your script crashes on null rows.

---

### 2.2 `SSTREAM` — Read a Structured Stream File

Structured Streams (`.ss` files) are Microsoft's internal compressed columnar binary format — much faster than text for big data. You do not define schema because it is already stored inside the file.

```sql
// Read a single .ss file
@mydata = SSTREAM "/path/to/file.ss";
```

---

### 2.3 `SSTREAM SPARSE STREAMSET` — Read Many Files at Once (Date Range)

This is how you read **multiple daily files across a date range** in one command. It is one of the most important patterns in production scripts.

```sql
// Pattern: the path has date placeholders %Y/%m/%d
#DECLARE BasePath string = "/shares/team/data/";
#DECLARE FilePattern string = "Folder/%Y/%m/%d/DataFile.ss";

@mydata =
    SSTREAM SPARSE STREAMSET @BasePath
        PATTERN @FilePattern
        RANGE __date = [@START_DATE, @END_DATE];
```

**How it works:**
- `%Y` = year (4 digits), `%m` = month (2 digits), `%d` = day (2 digits)
- Scope automatically expands the pattern for every date in the range
- `SPARSE` = skip missing dates instead of crashing (important for weekends, holidays, or gaps)
- `__date` is a built-in column that contains the file's date

**Example — if range is 2024-03-01 to 2024-03-03, Scope reads:**
```
/shares/team/data/Folder/2024/03/01/DataFile.ss
/shares/team/data/Folder/2024/03/02/DataFile.ss
/shares/team/data/Folder/2024/03/03/DataFile.ss
```

All rows from all three files are combined into one virtual table `@mydata`.

---

### 2.4 `VIEW` with `PARAMS` — Read a Logical View

A VIEW is a pre-built data layer that abstracts the physical file locations. Instead of knowing exactly where data lives, you call the view with parameters.

```sql
@result =
    VIEW "/path/to/SomeView.view"
    PARAMS
    (
        ViewName             = "SpecificDataset",
        RequestEndDateTimeUtc = @EndDateTimeUtc,
        ViewType             = 1
    );
```

**Why views are used:**
- The underlying file paths can change — the view handles that, your script does not need to change
- Views can apply standard filters (fraud removal, privacy) before you even see the data
- Views are maintained by a central team; you just call them

**Monetization View pattern** — very common in Microsoft Ads:

```sql
// Impressions view — only read impression data (ReadAdImpressions: true)
ImpData =
    VIEW "/path/MonetizationFacts.view"
    PARAMS
    (
        StartDateTimeUtc  = @StartDateTimeUtc,
        EndDateTimeUtc    = @EndDateTimeUtc,
        ReadAdImpressions = true,
        ReadClicks        = false,
        ReadConversions   = false,
        FraudType         = "NonFraud"     // Always filter fraud at source
    );

// Click view — only read click data
ClickData =
    VIEW "/path/MonetizationFacts.view"
    PARAMS
    (
        StartDateTimeUtc  = @StartDateTimeUtc,
        EndDateTimeUtc    = @EndDateTimeUtc,
        ReadAdImpressions = false,
        ReadClicks        = true,
        ReadConversions   = false,
        FraudType         = "NonFraud"
    );

// Conversion view — only read conversion data
ConvData =
    VIEW "/path/MonetizationFacts.view"
    PARAMS
    (
        StartDateTimeUtc    = @StartDateTimeUtc,
        EndDateTimeUtc      = @EndDateTimeUtc,
        ReadClicks          = false,
        ReadConversions     = true,
        ReadUETConversions  = true
    );
```

> You read the same view three times with different params to get impressions, clicks, and conversions separately, then join them. This is standard practice.

---

## PART 3 — Transforming Data

---

### 3.1 `SELECT` with Aggregation — Production Patterns

```sql
// Basic aggregation with multiple metrics
@summary =
    SELECT  Date,
            CustomerId,
            CampaignId,
            SUM(Impressions)  AS Impressions,
            SUM(Clicks)       AS Clicks,
            SUM(Revenue)      AS Revenue,
            SUM(Conversions)  AS Conversions
    FROM    @rawdata
    GROUP BY Date, CustomerId, CampaignId;
```

---

### 3.2 `COUNTIF()` — Count Rows That Match a Condition

`COUNTIF` is a Scope-specific aggregate that counts rows where a condition is true. This does not exist in standard SQL.

```sql
// Count how many campaigns are PMax (ChannelTypeId == 9) per customer per day
@summary =
    SELECT  Date,
            CustomerId,
            COUNTIF(ChannelTypeId == 9)   AS PMaxCampaignCount,
            COUNTIF(ChannelTypeId == 1)   AS SearchCampaignCount,
            COUNT()                        AS TotalCampaigns
    FROM @campaigns
    GROUP BY Date, CustomerId;
```

> `COUNT()` in Scope counts all rows (no argument needed — unlike SQL's `COUNT(*)`).

---

### 3.3 `FIRST()` — Take the First Value in a Group

When you `GROUP BY` but need a non-aggregated column (like a name that is the same for all rows in the group), use `FIRST()`.

```sql
// Group by CustomerId, but keep the CustomerName (same for all rows of a customer)
@customers =
    SELECT  CustomerId,
            FIRST(CustomerName) AS CustomerName,
            FIRST(Segment)      AS Segment
    FROM    @raw
    GROUP BY CustomerId;
```

> Do not use `ANY_VALUE()` or `MAX()` for names — `FIRST()` is the Scope standard.

---

### 3.4 `IF()` Function — Scope's Inline Conditional

Scope has a built-in `IF()` function (3 arguments: condition, true value, false value). Use this for simple conditionals inside SELECT.

```sql
SELECT
    // If campaign is PMax (type 9), keep AdDisplayTypeId, else use -9999 as placeholder
    IF(CampaignTypeId == 9, AdDisplayTypeId, -9999)  AS AdDisplayTypeId,

    // If campaign is Smart Shopping (type 3, subtype 12), use 312 as a combined ID
    IF(CampaignTypeId == 3 AND CampaignSubTypeId == 12, 312, CampaignTypeId) AS CampaignTypeId,

    // Boolean to int
    IF(ChannelTypeId == 9, true, false)  AS IsPMax

FROM @data;
```

**Nested IF — for multiple conditions:**

```sql
SELECT
    IF(IsSS,   312,               // Smart Shopping = 312
    IF(IsPMax, ChannelTypeId,     // PMax = keep original
               -9999))            // Everything else = -9999
    AS NormalizedChannelTypeId
FROM @data;
```

---

### 3.5 `??` Null Coalescing Operator — Handle Nulls

The `??` operator means "use this value if the left side is null". It is from C# and works in Scope SELECT.

```sql
SELECT
    CustomerName    ?? "Unknown"      AS CustomerName,
    Segment         ?? "Unsegmented"  AS Segment,
    ServiceCountry  ?? "Unknown"      AS ServiceCountry,
    AccountExecutive?? "Unassigned"   AS AE
FROM @customers;
```

> Always use `??` when joining tables with LEFT OUTER JOIN — the right side columns will be null for unmatched rows.

---

### 3.6 `IN()` — Match Against a List of Values

```sql
// Filter for specific channel types
WHERE ChannelTypeId IN (1, 3, 6, 9)

// Filter including null (null means the same as Search in some data models)
WHERE ChannelTypeId IN (null, 1, 3, 6, 9) AND IsServable
```

---

### 3.7 `Convert.*` — Type Casting

Use `Convert.*` to change a column's data type. This is needed when joining two tables where the same ID is stored as different types in different sources.

```sql
SELECT
    Convert.ToInt64(CampaignId)   AS CampaignId,    // int → long (for large IDs)
    Convert.ToInt32(CustomerId)   AS CustomerId,    // long → int
    Convert.ToDateTime(DateStr)   AS Date,           // string → DateTime
    (long)0                       AS Clicks,         // literal cast
    (decimal)0.0                  AS Conversions     // literal cast
FROM @data;
```

> **Common bug:** Two tables both have `CampaignId` but one is `int` and the other is `long`. The JOIN silently fails or crashes. Always cast to the same type before joining.

---

### 3.8 `UNION ALL` vs `UNION` — Combining Result Sets

```sql
// UNION ALL — keep ALL rows including duplicates (faster, use this by default)
@combined =
    SELECT Date, CustomerId, SUM(Impressions) AS Impressions, (long)0 AS Clicks
    FROM @imp_data
    UNION ALL
    SELECT Date, CustomerId, (long)0 AS Impressions, SUM(Clicks) AS Clicks
    FROM @click_data;

// After UNION ALL, aggregate to collapse the two halves into one row per key
@combined =
    SELECT Date, CustomerId,
           SUM(Impressions) AS Impressions,
           SUM(Clicks)      AS Clicks
    FROM @combined
    GROUP BY Date, CustomerId;

// UNION — removes duplicate rows (slower, use only when deduplication needed)
@unique =
    SELECT CustomerId FROM @set_a
    UNION
    SELECT CustomerId FROM @set_b;
```

**UNION ALL + GROUP BY is the standard pattern** for merging impression and click data because they come from different view reads and need to be combined into one row.

---

### 3.9 `SELECT DISTINCT` — Remove Duplicate Rows

```sql
// Keep only unique combinations of these columns
@unique_campaigns =
    SELECT DISTINCT CustomerId,
                    AccountId,
                    CampaignId
    FROM @campaigns;
```

---

## PART 4 — Joining Data

---

### 4.1 `INNER JOIN` vs `LEFT OUTER JOIN`

```sql
// INNER JOIN — only rows that exist in BOTH tables (unmatched rows dropped)
@result =
    SELECT A.CampaignId, A.Revenue, B.CampaignName
    FROM @performance AS A
         INNER JOIN @metadata AS B
         ON A.CampaignId == B.CampaignId;

// LEFT OUTER JOIN — all rows from left, NULLs for right if no match
// Use this when the right table might not have every ID from the left
@result =
    SELECT A.CampaignId, A.Revenue,
           B.Segment ?? "Unknown" AS Segment    // <-- ?? handles the NULL case
    FROM @performance AS A
         LEFT OUTER JOIN @segments AS B
         ON A.CampaignId == B.CampaignId;
```

**Multi-table LEFT JOIN pattern** — very common in dimension table building:

```sql
@dim_customer =
    SELECT  A.CustomerId,
            B.CustomerName  ?? "Unknown"    AS CustomerName,
            C.Segment       ?? "Unknown"    AS Segment,
            D.Country       ?? "Unknown"    AS Country,
            E.AccountExec   ?? "Unassigned" AS AE
    FROM @customer_list    AS A
    LEFT OUTER JOIN @names  AS B ON A.CustomerId == B.CustomerId
    LEFT OUTER JOIN @segs   AS C ON A.CustomerId == C.CustomerId
    LEFT OUTER JOIN @geo    AS D ON A.CustomerId == D.CustomerId
    LEFT OUTER JOIN @ae_map AS E ON A.CustomerId == E.CustomerId;
```

---

### 4.2 `[SKEWJOIN]` — Handling Skewed Joins (Performance Hint)

When one table has a very uneven distribution of join keys (e.g., a few huge advertisers have millions of rows), the join will be extremely slow or crash. `SKEWJOIN` tells Scope how to handle this.

```sql
[SKEWJOIN=(SKEW=FROMLEFT, REPARTITION=FULLJOIN)]
@result =
    SELECT A.ClickId, A.Revenue, B.Conversions
    FROM @clicks AS A
    LEFT OUTER JOIN @conversions AS B
    ON A.RGUID == B.RGUID AND A.ClickId == B.ClickId;
```

**SKEW=FROMLEFT** = the left table (clicks) has the skewed keys (many rows for the same RGUID).
**REPARTITION=FULLJOIN** = repartition both sides to balance the workload.

> You will see this on Click-to-Conversion joins because click IDs for large advertisers repeat millions of times.

---

## PART 5 — Writing Data

---

### 5.1 `OUTPUT` to Text File

Writes a readable text file (CSV or tab-delimited). Used for Power BI inputs, Excel reports, or human-readable outputs.

```sql
OUTPUT @mydata
TO "/output/path/result.txt"
WITH STREAMEXPIRY "30"
USING DefaultTextOutputter(outputHeader: true);
```

---

### 5.2 `OUTPUT` to `SSTREAM` — Structured Stream (Binary)

Writes a high-performance binary `.ss` file. Faster to read, smaller in size. Used for intermediate data or when another Scope script will read the output.

```sql
[Privacy.Asset.NonPersonal]
OUTPUT @mydata
TO SSTREAM "/output/path/result.ss"
    CLUSTERED BY CustomerId, CampaignId
    SORTED BY CustomerId, CampaignId
    WITH STREAMEXPIRY "30";
```

**CLUSTERED BY** = distributes data across nodes by these columns (for parallel read performance).
**SORTED BY** = within each cluster, rows are sorted (for efficient range queries).
**WITH STREAMEXPIRY** = how many days before the file is automatically deleted from the cluster.

> Always set expiry. Files that never expire waste cluster storage and get flagged in code reviews.

---

### 5.3 `WITH STREAMEXPIRY` — File Retention Policy

```sql
// 30 days — standard for final outputs
WITH STREAMEXPIRY "30"

// 7 days — standard for intermediate files (not final outputs)
WITH STREAMEXPIRY "7"

// Common pattern — use declared variables
#DECLARE FinalExp        string = "30";
#DECLARE IntermediateExp string = "7";

OUTPUT @final_data    TO SSTREAM @FinalPath        WITH STREAMEXPIRY @FinalExp;
OUTPUT @interim_data  TO SSTREAM @IntermediatePath WITH STREAMEXPIRY @IntermediateExp;
```

---

## PART 6 — Advanced Concepts

---

### 6.1 `[Privacy.Asset.NonPersonal]` — Privacy Annotation

Every `OUTPUT` statement must be annotated with a privacy classification. This is enforced by the privacy compliance system. Without it, the script either fails or triggers an audit.

```sql
[Privacy.Asset.NonPersonal]
OUTPUT @aggregated_data
TO SSTREAM @OutputPath
    WITH STREAMEXPIRY "30";
```

**Common privacy annotations:**

| Annotation | When to use |
|---|---|
| `[Privacy.Asset.NonPersonal]` | Aggregated/anonymised data — no individual-level rows |
| `[Privacy.Asset.Personal]` | Contains user-identifiable data |
| `[Privacy.Asset.Pseudonymous]` | Contains hashed/pseudonymised IDs |

> In most analytics scripts, you aggregate to Customer/Campaign level so outputs are `NonPersonal`. If your output has RGUID, ClickId, or user-level data, it needs a different annotation.

---

### 6.2 `[LOWDISTINCTNESS(...)]` — Column Annotation for Privacy

When a column has low cardinality (few unique values — like `CampaignTypeId` which only has ~15 values), you annotate it so the privacy system knows it is not a unique identifier.

```sql
[LOWDISTINCTNESS(CampaignTypeId, MediumId, MarketplaceClassificationId, Date, CustomerId)]
@aggregated =
    SELECT Date,
           CustomerId,
           CampaignTypeId,
           MediumId,
           SUM(Impressions) AS Impressions,
           SUM(Revenue)     AS Revenue
    FROM @data
    GROUP BY Date, CustomerId, CampaignTypeId, MediumId;
```

> The annotated columns have few distinct values so they cannot uniquely identify a person. You annotate them so the privacy scanner does not flag them as potential PII.

---

### 6.3 `EXISTS()` — Check if a File Exists Before Reading It

Used in "append history" patterns where you try to read a previous output file, but on the first run it does not exist yet.

```sql
#DECLARE OutputFile string = "/output/data/history.txt";

// Only try to read history if the file already exists
#IF(@AddHistory AND EXISTS(@OutputFile))
    @final =
        SELECT * FROM @new_data
        UNION ALL
        SELECT *
        FROM
        (
            EXTRACT Col1 : string,
                    Col2 : int
            FROM @OutputFile
            USING DefaultTextExtractor()
        )
        WHERE !(Convert.ToDateTime(Date) >= Convert.ToDateTime(@START_DATE)
             AND Convert.ToDateTime(Date) <= Convert.ToDateTime(@END_DATE));
#ENDIF
```

**The pattern explained:**
1. Read today's fresh data into `@new_data`
2. If the history file exists, read it and exclude the date range you are updating (to avoid duplicates)
3. UNION ALL today's data with the old history (minus today's range)
4. OUTPUT the combined result back to the same file path (overwrites it)

This is how **incremental append** works in Scope. There is no `INSERT` — you rebuild the full file each run.

---

### 6.4 `#CS` / `#ENDCS` — Inline C# Code

When Scope's built-in functions are not enough, you write C# helper functions directly in the script. These are compiled alongside the Scope code.

```sql
#CS
// A simple lookup function — return a label for a numeric ID
static string GetChannelName(int channelTypeId)
{
    switch (channelTypeId)
    {
        case 1:  return "Search";
        case 3:  return "Shopping";
        case 6:  return "Smart";
        case 9:  return "PMax";
        default: return "Unknown";
    }
}

// A function that processes a nullable byte
static string GetBiddingScheme(byte? biddingSchemeId)
{
    switch (biddingSchemeId)
    {
        case 1: return "Manual";
        case 2: return "Max Clicks";
        case 3: return "Max Conversions";
        case 4: return "Target CPA";
        case 6: return "Target ROAS";
        case 8: return "Max Conversion Value";
    }
    return "Unknown";
}
#ENDCS

// Call the C# function inside SELECT like a built-in function
@labelled =
    SELECT  CustomerId,
            CampaignId,
            GetChannelName(CampaignTypeId)       AS ChannelName,
            GetBiddingScheme(BiddingSchemeId)     AS BiddingStrategy
    FROM @data;
```

**Rules for `#CS` blocks:**
- Functions must be `static`
- They live at script scope — callable from any SELECT in the script
- Put the `#CS` block at the **bottom** of the script (after all OUTPUT statements)
- Use C# switch/if logic — not Scope syntax
- Handle nulls explicitly (use `byte?` not `byte` for nullable columns)

---

### 6.5 Conversion Join Pattern — RGUID + ClickId

In Microsoft Ads data, connecting clicks to conversions requires joining on **two keys together**: `RGUID` (request GUID) and `ClickId`. Joining on just one is not unique enough.

```sql
// First: cap conversions at 1 per RGUID+ClickId (prevent double-counting)
@conversions_deduped =
    SELECT  RGUID,
            ClickId,
            IF(SUM(ConversionCredit) > 1, 1, SUM(ConversionCredit)) AS Conversions,
            MAX(AdvertiserRevenue) AS GrossProfit
    FROM @conv_view
    WHERE FraudQualityBand >= 2
    GROUP BY RGUID, ClickId;

// Then: LEFT JOIN clicks to capped conversions
[SKEWJOIN=(SKEW=FROMLEFT, REPARTITION=FULLJOIN)]
@click_with_conv =
    SELECT  A.Date,
            A.CustomerId,
            A.CampaignId,
            A.Clicks,
            A.Revenue,
            B.Conversions ?? (decimal)0.0  AS Conversions,
            B.GrossProfit ?? (decimal)0.0  AS GrossProfit
    FROM @clicks AS A
    LEFT OUTER JOIN @conversions_deduped AS B
    ON A.RGUID == B.RGUID AND A.ClickId == B.ClickId;
```

> `FraudQualityBand >= 2` is the standard fraud filter. Never remove this. Band 0 and 1 are fraudulent traffic.

---

### 6.6 Impression + Click Merge Pattern

Impressions and clicks come from separate view reads. To combine them into one fact table per campaign:

```sql
// Step 1: Give impressions zero-valued click columns
@imp_rows =
    SELECT Date, CustomerId, CampaignId,
           Impressions,
           (long)0    AS Clicks,
           Revenue,
           (decimal)0 AS Conversions
    FROM @imp_view;

// Step 2: Give clicks zero-valued impression columns
@clk_rows =
    SELECT Date, CustomerId, CampaignId,
           (long)0    AS Impressions,
           Clicks,
           Revenue,
           Conversions
    FROM @click_conv_view;

// Step 3: UNION ALL both sets
@combined =
    SELECT * FROM @imp_rows
    UNION ALL
    SELECT * FROM @clk_rows;

// Step 4: SUM everything — the zeros cancel out, real values accumulate
@fact =
    SELECT  Date, CustomerId, CampaignId,
            SUM(Impressions) AS Impressions,
            SUM(Clicks)      AS Clicks,
            SUM(Revenue)     AS Revenue,
            SUM(Conversions) AS Conversions
    FROM @combined
    GROUP BY Date, CustomerId, CampaignId;
```

---

## PART 7 — Adoption / Penetration Metric Pattern

A common business metric: "What fraction of customers who run any campaign also run a specific campaign type (e.g., PMax)?"

```sql
// Step 1: Get one row per customer per account per day with all their campaign types
@account_level =
    SELECT  Date,
            CustomerId,
            AccountId,
            COUNTIF(ChannelTypeId == 9)  AS PMaxCount,    // How many PMax campaigns
            SUM(Revenue)                  AS Revenue
    FROM @campaigns
    GROUP BY Date, CustomerId, AccountId;

// Step 2: Roll up to customer level
// Numerator   = accounts that have at least 1 PMax campaign
// Denominator = all accounts (regardless of campaign type)
@adoption =
    SELECT  Date,
            CustomerId,
            COUNTIF(PMaxCount > 0)                        AS Numerator,
            SUM(PMaxCount > 0 ? Revenue : 0)              AS NumeratorRevenue,
            COUNT()                                        AS Denominator,
            SUM(Revenue)                                   AS DenominatorRevenue
    FROM @account_level
    GROUP BY Date, CustomerId;

// Adoption Rate = Numerator / Denominator
// SUI (Spend Under Influence) = NumeratorRevenue / DenominatorRevenue
```

---

## PART 8 — Dimension Table Building Pattern

A dimension table enriches your fact data with descriptive attributes (customer name, segment, country, account manager). The standard pattern uses multiple LEFT JOINs from separate mapping files.

```sql
// Step 1: Read each mapping file separately
@seg_map =
    EXTRACT CustomerId : int, SegmentName : string
    FROM "/mapping/CustomerSegment.txt"
    USING DefaultTextExtractor();

@country_map =
    EXTRACT CustomerId : int, ServiceCountry : string
    FROM "/mapping/CustomerCountry.txt"
    USING DefaultTextExtractor();

@ae_map =
    EXTRACT CustomerId : int, AccountExecutive : string
    FROM "/mapping/CustomerAE.txt"
    USING DefaultTextExtractor();

// Step 2: Deduplicate each mapping (one row per customer)
@seg_map     = SELECT CustomerId, FIRST(SegmentName)     AS Segment    FROM @seg_map     GROUP BY CustomerId;
@country_map = SELECT CustomerId, FIRST(ServiceCountry)  AS Country    FROM @country_map GROUP BY CustomerId;
@ae_map      = SELECT CustomerId, FIRST(AccountExecutive)AS AE         FROM @ae_map      GROUP BY CustomerId;

// Step 3: Build the dimension — LEFT JOIN all mappings onto the customer list
@dim =
    SELECT  A.CustomerId,
            B.CustomerName ?? "Unknown"  AS CustomerName,
            C.Segment      ?? "Unknown"  AS Segment,
            D.Country      ?? "Unknown"  AS Country,
            E.AE           ?? "Unknown"  AS AE
    FROM @customer_list AS A
    LEFT OUTER JOIN @name_view AS B ON A.CustomerId == B.CustomerId
    LEFT OUTER JOIN @seg_map   AS C ON A.CustomerId == C.CustomerId
    LEFT OUTER JOIN @country_map AS D ON A.CustomerId == D.CustomerId
    LEFT OUTER JOIN @ae_map    AS E ON A.CustomerId == E.CustomerId;

// Step 4: Final deduplication (multiple joins can produce duplicates)
@dim =
    SELECT CustomerId,
           FIRST(CustomerName) AS CustomerName,
           FIRST(Segment)      AS Segment,
           FIRST(Country)      AS Country,
           FIRST(AE)           AS AE
    FROM @dim
    GROUP BY CustomerId;
```

---

## PART 9 — How to Work Fast in Production

---

### 9.1 Read the Script Top-to-Bottom — Follow the Variable Names

Every Scope script is a **pipeline**. Each step feeds the next. To understand any script:

1. Find all `#DECLARE` at top — these are the inputs and configs
2. Find all `SSTREAM` / `VIEW` / `EXTRACT` — these are where data enters
3. Follow the variable name (e.g., `Campaigns =` then `Campaigns =` again — it is being refined step by step)
4. Find all `OUTPUT` — these are the deliverables
5. Read top to bottom to see the transformation chain

---

### 9.2 Re-assignment Pattern — The Same Variable is Refined Multiple Times

In Scope, assigning to the same variable name does **not overwrite** — it defines a new logical step. The execution engine builds a DAG (Directed Acyclic Graph).

```sql
// Step 1: Raw extract
Campaigns =
    SSTREAM SPARSE STREAMSET @BasePath PATTERN @Pattern RANGE __date = [@START, @END];

// Step 2: Filter columns
Campaigns =
    SELECT DISTINCT AccountId, Convert.ToInt64(CampaignId) AS CampaignId
    FROM Campaigns;

// Step 3: Join with accounts
Campaigns =
    SELECT A.CustomerId, B.CampaignId, B.AccountId
    FROM Accounts AS A
    INNER JOIN Campaigns AS B ON A.AccountId == B.AccountId;
```

Each `Campaigns =` creates a new logical step. Reading it as "refine the dataset step by step" makes production scripts much easier to understand.

---

### 9.3 Debugging Tips

```sql
// Tip 1: Use #ERROR to print a calculated value during compilation
#DECLARE END_DATE string = DateTime.Parse(@START_DATE).AddDays(6).ToString("yyyy-MM-dd");
#ERROR @END_DATE;   // Intentionally crash and show the date — comment out when done

// Tip 2: Add a COUNT to verify row counts at each step
@check =
    SELECT COUNT() AS RowCount FROM @campaigns;
OUTPUT @check TO "/debug/count.txt" USING DefaultTextOutputter(outputHeader: true);

// Tip 3: Check if a file has data before processing
#IF(EXISTS(@InputFile))
    // process
#ELSE
    #ERROR "Input file not found - check path";
#ENDIF

// Tip 4: Limit rows while testing (comment out in production)
@sample =
    SELECT TOP 1000 * FROM @huge_table;
```

---

### 9.4 Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| Type mismatch on JOIN | `CampaignId` is `int` in one table, `long` in another | `Convert.ToInt64(CampaignId)` before joining |
| Null reference crash | Column has nulls, you did math on it | Use `?? 0` or nullable types `int?` |
| Divide by zero | Calculating CTR, CPC etc. | `(denominator == 0 ? 0.0 : numerator / denominator)` |
| Skew / slow join | A few keys have millions of rows | Add `[SKEWJOIN]` hint |
| File not found | SSTREAM path wrong or date has no data | Use `SPARSE` in STREAMSET |
| Privacy annotation missing | OUTPUT has no `[Privacy.Asset.*]` | Add annotation above OUTPUT |
| Duplicate rows in output | Multiple joins multiplying rows | Add GROUP BY or SELECT DISTINCT at end |
| `#ERROR` in preprocessor | `#IF` condition check fires the error | Either flags are wrong, or fix the condition |

---

### 9.5 Standard Script Structure — Always Follow This Order

```
1.  MODULE imports
2.  #DECLARE date parameters (from @@runtime params@@)
3.  #DECLARE boolean feature flags
4.  #IF validation / #ERROR guards
5.  #DECLARE computed date variables (EndDate, DateTimeUtc etc.)
6.  #DECLARE project paths (BasePath, OutputFolder)
7.  #DECLARE input paths
8.  #DECLARE output paths (all outputs declared at top, used at bottom)

9.  #IF(@Demand)
        SSTREAM / VIEW reads for demand data
        SELECT transforms
        JOINs
        OUTPUT demand results
    #ENDIF

10. #IF(@Performance)
        VIEW reads for impressions
        VIEW reads for clicks
        VIEW reads for conversions
        Conversion deduplication
        Click-conversion join
        Impression-click union + aggregate
        OUTPUT performance results
    #ENDIF

11. #IF(@DimTable)
        EXTRACT mapping files
        Deduplicate each mapping
        Multi-LEFT-JOIN dimension build
        OUTPUT dimension
    #ENDIF

12. #CS
        C# helper functions
    #ENDCS
```

---

## PART 10 — Quick Reference Card

### Scope-Specific Functions (Not in Standard SQL)

| Function | Syntax | Use |
|---|---|---|
| `COUNTIF()` | `COUNTIF(condition)` | Count rows where condition is true |
| `COUNT()` | `COUNT()` | Count all rows in group |
| `FIRST()` | `FIRST(column)` | Take first value in group |
| `IF()` | `IF(cond, true_val, false_val)` | Inline conditional |
| `EXISTS()` | `EXISTS(@path)` | Check if file exists |
| Null coalesce | `column ?? default` | Replace null with default |
| Ternary | `condition ? a : b` | C#-style conditional |

### Preprocessor Directives

| Directive | Use |
|---|---|
| `#DECLARE name type = value` | Declare script variable |
| `@@PARAM@@` | Runtime-injected parameter |
| `#IF(condition)` / `#ENDIF` | Conditional compilation |
| `#ERROR "message"` | Crash script with message |
| `#CS` / `#ENDCS` | Inline C# code block |

### Data Sources

| Source | When to use |
|---|---|
| `EXTRACT FROM "path" USING DefaultTextExtractor()` | Plain text / CSV / TSV file |
| `SSTREAM "path.ss"` | Single structured stream binary file |
| `SSTREAM SPARSE STREAMSET` with PATTERN + RANGE | Multiple daily files across a date range |
| `VIEW "path.view" PARAMS(...)` | Logical view with pre-built schema |

### Output Types

| Output | When to use |
|---|---|
| `OUTPUT TO "path.txt" USING DefaultTextOutputter(outputHeader:true)` | Power BI, Excel, human-readable |
| `OUTPUT TO SSTREAM "path.ss" CLUSTERED BY ... SORTED BY ...` | Next Scope script reads it, or long-term storage |

---

*Covers: `#DECLARE` · `@@runtime params@@` · `#IF/#ENDIF/#ERROR` · `SSTREAM SPARSE STREAMSET` · `VIEW with PARAMS` · `COUNTIF` · `FIRST` · `IF()` · `??` · `IN()` · `Convert.*` · `UNION ALL` · `LEFT OUTER JOIN` · `SKEWJOIN` · `Privacy.Asset` · `LOWDISTINCTNESS` · `EXISTS()` · `#CS/#ENDCS` · Incremental append pattern · Impression-Click-Conversion merge pattern · Adoption metric pattern · Dimension table pattern*
