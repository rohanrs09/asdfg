# Scope Script — Simple Revision Notes
### What it is · Why we use it · How to write it

---

## First — What is Scope Script?

Imagine you have **500 crore rows** of Microsoft Ads data —
every click, every impression, every conversion from every advertiser.

Your laptop cannot handle that.
Excel cannot handle that.
Even a normal database is too slow.

**Scope Script** runs on thousands of computers at the same time.
It splits your data across all those machines and processes everything together.

> Think of it like this — SQL is one person doing the work.
> Scope is 10,000 people doing the same work at the same time.

That is the only big difference. If you know SQL, you already know 70% of Scope.

---

---

# PART 1 — Setting Up the Script

---

## 1. `#DECLARE` — Giving a Name to a Value

### What does it mean?
A `#DECLARE` creates a **named box** that holds one value.
You give it a name, a type, and a value.
Then you use that name everywhere instead of writing the value again and again.

### Why do we use it?
Imagine your script runs for the date **2024-03-01**.
If you write that date in 15 places and the date changes tomorrow, you must fix it 15 times.
With `#DECLARE`, you fix it in **one place** — everything else updates automatically.

### How to write it
```sql
#DECLARE  Name  type  =  value;
```

### Examples
```sql
-- Store a date as text
#DECLARE START_DATE  string  =  "2024-03-01";

-- Store a number
#DECLARE Days  int  =  7;

-- Store true or false (a flag)
#DECLARE Performance  bool  =  true;

-- Store a file path
#DECLARE OutputFolder  string  =  "/output/ads/reports/";
```

### How to USE the variable — write @ before the name
```sql
-- @START_DATE means "use whatever value is stored in START_DATE"
OUTPUT @results TO @OutputFolder;
```

---

## 2. `@@PARAM@@` — Value Given by the Job Scheduler

### What does it mean?
Double `@@` around a name means:
**"I do not know this value now. The job scheduler will give it to me when the script actually runs."**

### Why do we use it?
In production, the script runs every morning automatically.
The scheduler says "today is 2024-03-15" and injects that into the script.
The script does not need to know the date in advance — the scheduler tells it.

### How to write it
```sql
-- Production version — scheduler fills this in
#DECLARE START_DATE  string  =  @@RUN_DATE@@;
#DECLARE Days        int     =  @@Days@@;

-- Testing version — you fill it in manually while testing
#DECLARE START_DATE  string  =  "2024-03-01";
```

> **Rule:** Use `@@PARAM@@` in production. Use a hardcoded value when testing.

---

## 3. Building One Variable From Another

### What does it mean?
You can create a new variable by using the value of one you already declared.

### Why do we use it?
If you declare `START_DATE` and `Days`, you can automatically calculate `END_DATE`.
You never have to manually calculate the end date — the script does it for you.

### Examples
```sql
#DECLARE START_DATE  string  =  "2024-03-01";
#DECLARE Days        int     =  7;

-- END_DATE = START_DATE + (Days - 1)
-- March 1 + 6 days = March 7
#DECLARE END_DATE  string  =
    DateTime.Parse(@START_DATE).AddDays(@Days - 1).ToString("yyyy-MM-dd");

-- Build full DateTime objects (needed when calling VIEWs)
#DECLARE StartDateTime  DateTime  =  DateTime.Parse(@START_DATE + " 00:00");
#DECLARE EndDateTime    DateTime  =  DateTime.Parse(@END_DATE   + " 23:59");
```

> **Read the END_DATE line like English:**
> "Parse the start date as a date, add Days-1 to it, format the result as yyyy-MM-dd."

---

## 4. Feature Flags — Switching Parts of the Script On or Off

### What does it mean?
A feature flag is a `true` or `false` variable that controls whether a whole section of the script runs.

### Why do we use it?
One script can do many jobs — pull demand data, pull performance data, build customer tables.
Sometimes you only need one of those jobs.
Flags let you run only what you need without deleting code.

```sql
#DECLARE Demand      bool  =  true;   -- Run the demand section
#DECLARE Performance bool  =  true;   -- Run the performance section
#DECLARE DimTable    bool  =  true;   -- Run the customer dimension section
#DECLARE AddHistory  bool  =  true;   -- Append to history file
```

---

## 5. `#IF` / `#ENDIF` — Run a Block Only When a Condition is True

### What does it mean?
`#IF` checks a condition before the script runs.
If the condition is true, the code inside runs.
If false, that block is completely skipped — like it does not exist.

### Why do we use it?
It saves compute cost. If `Performance = false`, the entire performance section is skipped.
No wasted time, no wasted resources.

```sql
-- Only process performance data if the flag is true
#IF(@Performance)

    -- All performance code goes here
    -- This whole block is SKIPPED if Performance = false

#ENDIF
```

### `#ERROR` — Crash the Script on Purpose With a Clear Message

### What does it mean?
`#ERROR` immediately stops the script and shows your message.

### Why do we use it?
To catch impossible situations early.
If both Demand and Performance are false, there is nothing for the script to do.
Instead of running and producing empty output silently, crash immediately with a clear message.

```sql
-- The ! means NOT
-- !@Demand means "Demand is false"
#IF(!@Demand AND !@Performance)
    #ERROR "At least one of Demand or Performance must be set to true";
#ENDIF
```

### Using `#IF` Inside a SELECT

You can also use `#IF` to change what a SELECT does based on a flag.

```sql
-- When Days = 1 (single day run), use the declared date
-- When Days > 1 (multi-day run), use the Date column from the data
SELECT
    #IF(@Days == 1)
        @EndDateTime.ToString("MM/dd/yyyy")  AS Date,
    #ELSE
        Date,
    #ENDIF
    CustomerId,
    SUM(Revenue) AS Revenue
FROM @data
GROUP BY Date, CustomerId;
```

---

---

# PART 2 — Reading Data Into the Script

---

## 6. `EXTRACT` — Read a Plain Text File

### What does it mean?
`EXTRACT` reads a plain text file (like CSV or TSV) line by line.
You must tell Scope what columns exist and what type each column is.

### Why do we use it?
Mapping files — like customer segment lists, country lists, account manager assignments — are stored as plain text files.
`EXTRACT` is how you read them into your script.

```sql
@CustomerSegments =
    EXTRACT  CustomerId   : int,
             SegmentName  : string
    FROM  "/local/Team/BI/Mapping/CustomerSegment.txt"
    USING  DefaultTextExtractor();
```

### Data Types You Will See

| Type | Stores | Example |
|---|---|---|
| `string` | Any text | `"Search Campaign"` |
| `int` | Whole number | `12345` |
| `long` | Very large whole number | `987654321098` |
| `double` | Decimal number | `3.14` |
| `decimal` | Money / high precision decimal | `99.9999` |
| `bool` | True or False | `true` |
| `DateTime` | A date and time | `2024-03-01` |
| `int?` | Whole number **or null** | `null` or `42` |
| `byte?` | Small number (0-255) **or null** | `null` or `5` |
| `sbyte?` | Small signed number **or null** | `-1` or `null` |

### What does the `?` mean?
The `?` at the end of a type means **nullable** — the column is allowed to be empty/missing.
If a column might have missing values and you do not mark it nullable, the script **crashes**.
Always use `int?`, `byte?` etc. when a column might not always have a value.

---

## 7. `SSTREAM` — Read a Fast Binary File

### What does it mean?
A Structured Stream (`.ss` file) is Microsoft's own binary file format.
It is compressed, fast to read, and used everywhere on the Cosmos cluster.
You cannot open it in Notepad — it is a binary file.

### Why do we use it?
Text files are slow for big data. An `.ss` file is much faster because it stores data in a column format and is compressed.
You do not need to define the schema — it is already stored inside the file.

```sql
-- Read a single .ss file — no schema needed
@MyData = SSTREAM "/path/to/file.ss";
```

---

## 8. `SSTREAM SPARSE STREAMSET` — Read Many Daily Files in One Command

### What does it mean?
Data on the cluster is stored in daily folders — one folder per day.
`SSTREAM SPARSE STREAMSET` reads **all the daily files across a date range** in a single command.

### Why do we use it?
To analyse 7 days of data, you would normally need 7 separate reads.
`STREAMSET` does all 7 (or 30 or 90) reads in one line.

### How the folder structure looks on the cluster
```
/shares/data/Campaigns/2024/03/01/Campaigns.ss
/shares/data/Campaigns/2024/03/02/Campaigns.ss
/shares/data/Campaigns/2024/03/03/Campaigns.ss
```

### How to write it
```sql
#DECLARE BasePath    string = "/shares/bingAds.BI.OI/AdsOI/DemandMetrics/Data/";
#DECLARE FilePattern string = "UnifiedLayerDemand/%Y/%m/%d/Campaigns.ss";

@Campaigns =
    SSTREAM SPARSE STREAMSET @BasePath
        PATTERN @FilePattern
        RANGE __date = [@START_DATE, @END_DATE];
```

### What do `%Y`, `%m`, `%d` mean?
- `%Y` = 4-digit year → `2024`
- `%m` = 2-digit month → `03`
- `%d` = 2-digit day → `01`

Scope fills these in automatically for every date in your range.

### What does `SPARSE` mean?
If a file is missing for a particular day (weekend, holiday, no data that day),
`SPARSE` **skips** that day instead of crashing.
**Always use SPARSE** — without it, one missing file kills the whole script.

---

## 9. `VIEW with PARAMS` — Read Through a Pre-Built Data Layer

### What does it mean?
A VIEW is like a smart door to data.
A central team maintains the VIEW — they know exactly where the data lives.
You just call the VIEW with the parameters you need.
You do not need to know the physical file paths.

### Why do we use it?
The Monetization VIEW is the core Microsoft Ads data source.
It handles impressions, clicks, and conversions.
File paths change over time — but the VIEW always points to the right place.
It also applies standard filters automatically.

### How to write it
```sql
@Result =
    VIEW "/path/to/MonetizationFacts.view"
    PARAMS
    (
        ParameterName = value,
        ParameterName = value
    );
```

### The 3-Read Pattern — Impressions, Clicks, Conversions Read Separately

In Microsoft Ads analysis, you read the same VIEW three times with different flags:

```sql
-- READ 1: Impressions only
MVImp =
    VIEW @MonetizationViewPath
    PARAMS (
        StartDateTimeUtc  = @StartDateTime,
        EndDateTimeUtc    = @EndDateTime,
        ReadAdImpressions = true,
        ReadClicks        = false,
        FraudType         = "NonFraud"
    );

-- READ 2: Clicks only
MVClick =
    VIEW @MonetizationViewPath
    PARAMS (
        StartDateTimeUtc  = @StartDateTime,
        EndDateTimeUtc    = @EndDateTime,
        ReadAdImpressions = false,
        ReadClicks        = true,
        FraudType         = "NonFraud"
    );

-- READ 3: Conversions only
MVConv =
    VIEW @MonetizationViewPath
    PARAMS (
        StartDateTimeUtc   = @StartDateTime,
        EndDateTimeUtc     = @EndDateTime,
        ReadClicks         = false,
        ReadConversions    = true,
        ReadUETConversions = true
    );
```

### Why 3 separate reads?
Impressions, clicks, and conversions are stored in different internal formats.
You read them separately and then merge them into one fact table later.

---

---

# PART 3 — Working With Data

---

## 10. `SELECT` and Re-assigning the Same Variable

### What does it mean?
`SELECT` works exactly like SQL — pick columns, rename them, calculate new ones, filter, and group.

The important difference in Scope is:
**you can assign to the same variable name multiple times.**
Each time, you refine it one step further.

### Why do we use it?
It makes the script readable.
Instead of one huge complicated query, you build up the result step by step.

```sql
-- STEP 1: Read raw data
Campaigns =
    SSTREAM SPARSE STREAMSET @BasePath PATTERN @Pattern RANGE __date = [@START_DATE, @END_DATE];

-- STEP 2: Clean it — pick needed columns, fix types
Campaigns =
    SELECT DISTINCT
        __date                      AS Date,
        AccountId,
        Convert.ToInt64(CampaignId) AS CampaignId,
        AdvertisingChannelTypeId,
        IsEligible,
        IsServable
    FROM Campaigns;

-- STEP 3: Join with accounts to add customer info
Campaigns =
    SELECT B.Date, A.CustomerId, A.AccountId, B.CampaignId, B.AdvertisingChannelTypeId
    FROM Accounts AS A
    INNER JOIN Campaigns AS B ON A.AccountId == B.AccountId;
```

> Read each `Campaigns =` as: "Now Campaigns becomes this cleaner version."

---

## 11. Aggregate Functions

### `SUM()` — Add up all values in a group
```sql
SUM(Impressions) AS TotalImpressions
SUM(Revenue)     AS TotalRevenue
```

### `COUNT()` — Count all rows — no argument needed in Scope
```sql
COUNT() AS TotalRows
```

### `COUNT(DISTINCT col)` — Count only unique values
```sql
COUNT(DISTINCT CampaignId) AS UniqueCampaigns
```

### `COUNTIF(condition)` — Count rows where condition is true ← Scope only, not in SQL

### What does it mean?
It counts only the rows that match your condition. Other rows are ignored.

### Why do we use it?
In the adoption metric, you need to count how many campaigns in a group are PMax (channel type 9).
`COUNTIF` does this in one line without needing a separate filter step.

```sql
SELECT
    CustomerId,
    COUNTIF(AdvertisingChannelTypeId == 9)  AS PMaxCampaigns,
    COUNTIF(AdvertisingChannelTypeId == 1)  AS SearchCampaigns,
    COUNT()                                  AS TotalCampaigns
FROM @Campaigns
GROUP BY CustomerId;
```

### `FIRST()` — Take the first value in a group ← Scope only, not in SQL

### What does it mean?
When you `GROUP BY` an ID, other columns must be aggregated.
`FIRST()` takes the first value it sees for that column in the group.

### Why do we use it?
`CustomerName` is the same for every row of the same `CustomerId`.
You cannot just `GROUP BY CustomerId, CustomerName` and then also have `CustomerName` in SELECT without an aggregate.
`FIRST(CustomerName)` says "just take the first one — they are all the same anyway."

```sql
SELECT
    CustomerId,
    FIRST(CustomerName) AS CustomerName,
    FIRST(Segment)      AS Segment
FROM @raw
GROUP BY CustomerId;
```

---

## 12. `IF()` — Choose Between Two Values Based on a Condition

### What does it mean?
`IF(condition, value_if_true, value_if_false)` — picks one of two values.

### Why do we use it?
Campaign type IDs are numbers. You need to normalise them:
- If it is Smart Shopping (type 3, subtype 12), use 312 as a combined ID
- If it is PMax (type 9), keep `AdDisplayTypeId`
- Otherwise use -9999 as a placeholder

```sql
SELECT
    -- Smart Shopping gets a special combined ID of 312
    IF(CampaignTypeId == 3 AND CampaignSubTypeId == 12, 312, CampaignTypeId)
        AS NormalizedCampaignType,

    -- PMax: keep the display type. Others: use -9999
    IF(CampaignTypeId == 9, AdDisplayTypeId, -9999)
        AS NormalizedAdDisplayType,

    -- Is this a PMax campaign? Yes or No
    IF(AdvertisingChannelTypeId == 9, true, false)
        AS IsPMax

FROM @data;
```

### Nested IF — When you have more than two options
```sql
-- If SmartShopping → 312, if PMax → keep ID, else → -9999
IF(IsSS,    312,
IF(IsPMax,  AdvertisingChannelTypeId,
            -9999))
AS NormalizedChannelId
```

---

## 13. `??` — Handle Null Values Safely

### What does it mean?
`column ?? "default"` means:
"If this column is null (empty/missing), use this default value instead."

### Why do we use it?
When you do a LEFT OUTER JOIN, some rows from the right table will not have a match.
Those unmatched columns become **null**.
If you leave nulls in your output, Power BI and reports show blanks.
`??` fills them with a proper default.

```sql
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

## 14. `Convert.*` — Change a Column's Data Type

### What does it mean?
Converts a value from one data type to another.

### Why do we use it?
Two tables both have `CampaignId`, but one stores it as `int` and the other as `long`.
When you try to JOIN them, Scope throws a type mismatch error.
You must convert one to match the other before joining.

```sql
Convert.ToInt64(CampaignId)                          -- int → long (for large IDs)
Convert.ToInt32(CustomerId)                          -- long → int
Convert.ToDateTime(MBDate).ToString("MM/dd/yyyy")    -- DateTime → formatted string
```

### Literal Type Casting — Writing a Zero in a Specific Type
```sql
(long)0        AS Clicks       -- the number 0, stored as a long integer
(decimal)0.0   AS Conversions  -- the number 0, stored as a decimal
```

### Why literals?
In the UNION ALL merge pattern, impressions have no clicks.
You put `(long)0 AS Clicks` in the impressions query so that both halves of the UNION have matching column types.

---

## 15. `WHERE` — Filter Rows

```sql
WHERE Revenue > 0                                        -- basic filter
WHERE AdSpend >= 50 AND Clicks > 0                       -- multiple conditions
WHERE AdvertisingChannelTypeId IN (1, 3, 6, 9)          -- match a list
WHERE AdvertisingChannelTypeId IN (null, 1, 3, 6, 9) AND IsServable  -- include null
WHERE FraudQualityBand >= 2                              -- ALWAYS include this
```

### What is `FraudQualityBand >= 2`?
Every click and impression in Microsoft Ads data has a fraud quality score.
- Band 0 and 1 = fraudulent traffic
- Band 2 and above = genuine traffic

**Never remove this filter** when reading Monetization VIEW data.
Without it, your metrics include fake clicks and inflate every number.

---

## 16. `UNION ALL` — Stack Two Tables on Top of Each Other

### What does it mean?
`UNION ALL` takes two tables that have the same columns and combines them vertically — all rows from both tables in one result.

### Why do we use it?
Impressions come from one VIEW read. Clicks come from another VIEW read.
They are in separate tables but you need them in one fact table.
You stack them with `UNION ALL` and then `SUM` to get one row per campaign.

### The Impression + Click Merge Pattern — Most Used in Production

```sql
-- STEP 1: Impressions table — give it zero-value click columns
@imp_rows =
    SELECT Date, CustomerId, CampaignId,
           Impressions,
           (long)0    AS Clicks,         -- no clicks in this read, so put zero
           Revenue,
           (decimal)0 AS Conversions
    FROM @MVImp;

-- STEP 2: Clicks table — give it zero-value impression columns
@clk_rows =
    SELECT Date, CustomerId, CampaignId,
           (long)0    AS Impressions,    -- no impressions in this read, so put zero
           Clicks,
           Revenue,
           Conversions
    FROM @MVClick;

-- STEP 3: Stack them
@combined =
    SELECT * FROM @imp_rows
    UNION ALL
    SELECT * FROM @clk_rows;

-- STEP 4: SUM — zeros cancel out, real values add up → one row per campaign
@fact =
    SELECT Date, CustomerId, CampaignId,
           SUM(Impressions) AS Impressions,
           SUM(Clicks)      AS Clicks,
           SUM(Revenue)     AS Revenue,
           SUM(Conversions) AS Conversions
    FROM @combined
    GROUP BY Date, CustomerId, CampaignId;
```

> Each campaign now has **one row** with real impressions AND real clicks.
> The zeros just disappear when you SUM.

---

## 17. `SELECT DISTINCT` — Remove Duplicate Rows

### What does it mean?
Keeps only unique combinations. If the same row appears twice, one copy is removed.

### Why do we use it?
Some data sources have duplicate rows. Before joining, deduplicate to avoid multiplying rows in your output.

```sql
@unique_campaigns =
    SELECT DISTINCT CustomerId, AccountId, CampaignId
    FROM @raw;
```

---

---

# PART 4 — Joining Tables

---

## 18. `INNER JOIN` — Only Keep Rows That Exist in Both Tables

### What does it mean?
Rows that exist in one table but not the other are **dropped**.
Only rows that match the join condition from **both** tables are kept.

### Why do we use it?
Use INNER JOIN when you only want confirmed matches.
For example — join campaigns with accounts, and only keep campaigns that actually belong to a known account.

```sql
-- Only campaigns that belong to a known account are kept
Campaigns =
    SELECT A.CustomerId, A.CustomerName, B.CampaignId
    FROM @Accounts   AS A
    INNER JOIN @Campaigns AS B
    ON A.AccountId == B.AccountId;
```

---

## 19. `LEFT OUTER JOIN` — Keep All Rows from the Left Table

### What does it mean?
Every row from the **left** table is kept.
If a match exists in the right table, its columns are included.
If no match exists, the right table columns are **null**.

### Why do we use it?
When building dimension tables, not every customer will have a segment mapping or country mapping.
LEFT OUTER JOIN keeps all customers — unmatched ones just get null values, which you fill with `??`.

```sql
-- All campaigns are kept. If no segment found, Segment = null → filled with "Unknown"
@result =
    SELECT A.CampaignId, A.Revenue,
           B.Segment ?? "Unknown" AS Segment
    FROM @Campaigns      AS A
    LEFT OUTER JOIN @Segments AS B
    ON A.CustomerId == B.CustomerId;
```

---

## 20. Multi-Table LEFT JOIN — Building the Customer Dimension Table

### What does it mean?
You LEFT JOIN many mapping files one by one onto a base customer list.
Each join adds one more attribute (segment, country, AE name, AM name).

### Why do we use it?
Customer attributes come from many different mapping files.
You build one unified customer dimension by joining them all.

```sql
@DIM_Customer =
    SELECT  A.CustomerId,
            B.CustomerName   ?? "Unknown"     AS CustomerName,
            C.Segment        ?? "Unsegmented" AS Segment,
            D.L1VerticalName ?? "Unknown"     AS Vertical,
            E.ServiceCountry ?? "Unknown"     AS Country,
            F.AccountExec    ?? "Unassigned"  AS AE,
            G.AccountManager ?? "Unassigned"  AS AM
    FROM @CustomerList      AS A
    LEFT OUTER JOIN @NameView    AS B ON A.CustomerId == B.CustomerId
    LEFT OUTER JOIN @SegmentMap  AS C ON A.CustomerId == C.CustomerId
    LEFT OUTER JOIN @VerticalMap AS D ON A.CustomerId == D.CustomerId
    LEFT OUTER JOIN @CountryMap  AS E ON A.CustomerId == E.CustomerId
    LEFT OUTER JOIN @AEMap       AS F ON A.CustomerId == F.CustomerId
    LEFT OUTER JOIN @AMMap       AS G ON A.CustomerId == G.CustomerId;

-- Always deduplicate after multiple joins (joins can create duplicate rows)
@DIM_Customer =
    SELECT CustomerId,
           FIRST(CustomerName) AS CustomerName,
           FIRST(Segment)      AS Segment,
           FIRST(Vertical)     AS Vertical,
           FIRST(Country)      AS Country,
           FIRST(AE)           AS AE,
           FIRST(AM)           AS AM
    FROM @DIM_Customer
    GROUP BY CustomerId;
```

---

## 21. `[SKEWJOIN]` — Fix Extremely Slow Joins

### What does it mean?
Some join keys appear millions of times in a table — like a top advertiser's click ID.
When Scope tries to process that key, it sends all those millions of rows to **one machine**.
That machine gets overwhelmed, slows down, or crashes.
`SKEWJOIN` tells Scope to spread that work across many machines instead.

### Why do we use it?
The Click-to-Conversion join (joining on RGUID + ClickId) is always skewed.
Top advertisers have millions of clicks under the same keys.
Without SKEWJOIN, this join takes forever or fails.

```sql
[SKEWJOIN=(SKEW=FROMLEFT, REPARTITION=FULLJOIN)]
@ClickConv =
    SELECT A.Date, A.CustomerId, A.CampaignId,
           A.Clicks, A.Revenue,
           B.Conversions ?? (decimal)0.0 AS Conversions,
           B.GrossProfit ?? (decimal)0.0 AS GrossProfit
    FROM @MVClick AS A
    LEFT OUTER JOIN @MVConv AS B
    ON A.RGUID == B.RGUID AND A.ClickId == B.ClickId;
```

- `SKEW=FROMLEFT` — the left table (clicks) has the skewed keys
- `REPARTITION=FULLJOIN` — spread both sides across machines to balance load

---

## 22. Click-to-Conversion Join — Full Pattern

### What does it mean?
A conversion (someone buying something) is connected to the click that caused it.
Both have the same `RGUID` and `ClickId`. You join on both to connect them.

### Why do we deduplicate conversions first?
One click can sometimes generate multiple conversion records.
You cap it at 1 to prevent double counting.

```sql
-- STEP 1: Cap conversions at 1 per RGUID+ClickId
@ConvDeduped =
    SELECT RGUID, ClickId,
           IF(SUM(ConversionCredit) > 1, 1, SUM(ConversionCredit)) AS Conversions,
           MAX(AdvertiserReportedRevenue) AS GrossProfit
    FROM @MVConv
    WHERE FraudQualityBand >= 2
    GROUP BY RGUID, ClickId;

-- STEP 2: LEFT JOIN clicks to capped conversions (with SKEWJOIN)
[SKEWJOIN=(SKEW=FROMLEFT, REPARTITION=FULLJOIN)]
@ClickConv =
    SELECT A.Date, A.CustomerId, A.CampaignId,
           A.Clicks, A.Revenue,
           B.Conversions ?? (decimal)0.0 AS Conversions,
           B.GrossProfit ?? (decimal)0.0 AS GrossProfit
    FROM @MVClick AS A
    LEFT OUTER JOIN @ConvDeduped AS B
    ON A.RGUID == B.RGUID AND A.ClickId == B.ClickId;
```

---

---

# PART 5 — Writing Output

---

## 23. `OUTPUT` — Save Results to a File

### What does it mean?
`OUTPUT` writes your final result to a file on the cluster.
Think of it like `INSERT INTO` in SQL, but instead of a table, you write to a file.

### Text file — for Power BI, Excel, human-readable reports
```sql
[Privacy.Asset.NonPersonal]
OUTPUT @MyData
TO "/output/PowerBi/FACT_Performance.txt"
WITH STREAMEXPIRY "30"
USING DefaultTextOutputter(outputHeader: true);
```
- `outputHeader: true` — write column names in the first row
- `STREAMEXPIRY "30"` — automatically delete this file after 30 days

### Binary SSTREAM — for other Scope scripts to read fast
```sql
[Privacy.Asset.NonPersonal]
OUTPUT @MyData
TO SSTREAM "/output/data.ss"
    CLUSTERED BY CustomerId, CampaignId
    SORTED BY CustomerId, CampaignId
    WITH STREAMEXPIRY "30";
```
- `CLUSTERED BY` — spread data across cluster machines by these columns (makes reads faster)
- `SORTED BY` — within each cluster, sort rows (makes range queries faster)

### Which output type to use?

| Destination | Use |
|---|---|
| Power BI dashboard | Text file with `DefaultTextOutputter` |
| Excel download | Text file with `DefaultTextOutputter` |
| Another Scope script reads it | SSTREAM binary |
| Intermediate step | SSTREAM with `"7"` day expiry |
| Final output | Text or SSTREAM with `"30"` day expiry |

---

## 24. `STREAMEXPIRY` — How Long to Keep the File

### What does it mean?
Files stored on the cluster use resources. `STREAMEXPIRY` sets how many days until the file is automatically deleted.

### Why do we use it?
Without expiry, files stay forever and waste cluster storage.
Teams enforce expiry in code reviews — always set it.

```sql
#DECLARE FinalExpiry        string = "30";   -- Final outputs kept 30 days
#DECLARE IntermediateExpiry string = "7";    -- Intermediate files kept 7 days

OUTPUT @final_data TO SSTREAM @FinalPath    WITH STREAMEXPIRY @FinalExpiry;
OUTPUT @temp_data  TO SSTREAM @TempPath     WITH STREAMEXPIRY @IntermediateExpiry;
```

---

---

# PART 6 — Privacy and Compliance

---

## 25. `[Privacy.Asset.NonPersonal]` — Required Above Every OUTPUT

### What does it mean?
A privacy label that tells Microsoft's compliance system what type of data this output contains.

### Why do we use it?
Microsoft has strict data privacy rules.
Every file written to the cluster must be classified.
Without this annotation, the script fails compliance review or does not run at all.

```sql
[Privacy.Asset.NonPersonal]
OUTPUT @AggregatedData
TO SSTREAM @OutputPath
WITH STREAMEXPIRY "30";
```

| Label | When to use |
|---|---|
| `[Privacy.Asset.NonPersonal]` | Aggregated data — no individual user rows |
| `[Privacy.Asset.Personal]` | Contains user-identifiable data |
| `[Privacy.Asset.Pseudonymous]` | Contains hashed/anonymised user IDs |

> In most ads analytics scripts, data is aggregated to Customer/Campaign level.
> Aggregated = NonPersonal.

---

## 26. `[LOWDISTINCTNESS(...)]` — Annotate Low-Variety Columns

### What does it mean?
Some columns have very few unique values — `CampaignTypeId` only has about 15 possible values.
This annotation tells the privacy scanner those columns cannot uniquely identify a person.

### Why do we use it?
The privacy scanner checks every column for risk.
Without `LOWDISTINCTNESS`, it might flag `CampaignTypeId` as potentially identifying.
With it, the scanner knows these columns are safe dimension values.

```sql
[LOWDISTINCTNESS(CampaignTypeId, MediumId, MarketplaceClassificationId, Date, CustomerId)]
@MVImp =
    SELECT Date, CustomerId, CampaignTypeId, MediumId,
           SUM(Impressions) AS Impressions,
           SUM(Revenue)     AS Revenue
    FROM @MVImp
    WHERE FraudQualityBand >= 2;
```

---

---

# PART 7 — Advanced Patterns

---

## 27. `EXISTS()` — Check if a File Exists Before Reading It

### What does it mean?
`EXISTS(@path)` returns true if the file at that path exists, false if it does not.

### Why do we use it?
Scripts that run every day **append** results to a growing history file.
On the very first run, that history file does not exist yet.
Without `EXISTS()`, the script would crash trying to read a file that is not there.

```sql
-- Only try to read history if the file actually exists already
#IF(@AddHistory AND EXISTS(@HistoryFile))

    @FinalOutput =
        -- Today's fresh data
        SELECT * FROM @TodayData

        UNION ALL

        -- Previous history, but EXCLUDE today's date range
        -- (to avoid duplicates — today's fresh data replaces old today's data)
        SELECT *
        FROM (EXTRACT Date : string, CustomerId : int?, ... FROM @HistoryFile USING DefaultTextExtractor())
        WHERE !(    Convert.ToDateTime(Date) >= Convert.ToDateTime(@START_DATE)
                AND Convert.ToDateTime(Date) <= Convert.ToDateTime(@END_DATE));

#ENDIF
```

### The Incremental Append Pattern — Simply Explained

Think of it like a notebook:
- Every day you add that day's page
- If you need to fix a day, you remove the old page and add the corrected one
- The notebook keeps growing, always with the latest version of each day

```
Day 1 run → file has: Day1
Day 2 run → file has: Day1 + Day2
Day 3 run → file has: Day1 + Day2 + Day3

Day 2 reruns (correction) → file has: Day1 + Fresh Day2 + Day3
(old Day2 excluded by WHERE, fresh Day2 added by UNION ALL)
```

---

## 28. `#CS` / `#ENDCS` — Write C# Functions Inside Your Script

### What does it mean?
When Scope's built-in functions cannot do what you need, you write a C# function directly in the script.
That function then works like any built-in function — you can call it inside any SELECT.

### Why do we use it?
Converting numeric IDs to readable names is easiest with a lookup (switch statement).
For example: BiddingSchemeId 6 → "Target ROAS", AssetTypeId 32 → "Headline".
There is no built-in Scope function for custom lookup tables — so you write C#.

```sql
-- Put #CS at the BOTTOM of your script, after all OUTPUTs
#CS

// Converts BiddingSchemeId number to a readable name
static string GetBiddingScheme(byte? id)
{
    switch (id)
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

// Converts AssetTypeId to a readable asset category
static string GetAssetType(string id)
{
    if (id == null) return "";
    switch (id)
    {
        case "32": return "Headline";
        case "33": return "Long Headline";
        case "34": return "Description";
        case "36": return "CTA";
        default:   return "Image";
    }
}

// Converts ChannelTypeId to a channel name
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

#ENDCS

-- Now call these in SELECT like any built-in function
@labelled =
    SELECT CustomerId,
           CampaignId,
           GetChannelName(CampaignTypeId)    AS ChannelName,
           GetBiddingScheme(BiddingSchemeId) AS BiddingStrategy
    FROM @data;
```

### Rules for `#CS` functions
- All functions must say `static` before them
- Place the entire `#CS` block at the very **bottom** of the script
- Use nullable types (`byte?`, `int?`) for columns that might be null
- Always handle `null` — `if (id == null) return "";`

---

## 29. Adoption / Penetration Metric Pattern

### What does it mean?
Measures: "Out of all customers running any campaign, what fraction are also running a PMax campaign?"

### Why do we use it?
This is a key business metric — it shows how widely a new product (PMax) has been adopted.

```sql
-- STEP 1: Per account — how many PMax campaigns + total revenue
@AccountLevel =
    SELECT Date, CustomerId, AccountId,
           COUNTIF(AdvertisingChannelTypeId == 9) AS PMaxCount,
           SUM(Revenue) AS Revenue
    FROM @AllCampaigns
    WHERE AdvertisingChannelTypeId IN (null, 1, 3, 6, 9) AND IsServable
    GROUP BY Date, CustomerId, AccountId;

-- STEP 2: Per customer — count adopters vs total
@Adoption =
    SELECT Date, CustomerId,
           COUNTIF(PMaxCount > 0)           AS Numerator,       -- accounts WITH PMax
           SUM(PMaxCount > 0 ? Revenue : 0) AS NumeratorRevenue, -- revenue from those accounts
           COUNT()                           AS Denominator,     -- ALL accounts
           SUM(Revenue)                      AS DenominatorRevenue
    FROM @AccountLevel
    GROUP BY Date, CustomerId;
```

- **Adoption Rate** = `Numerator ÷ Denominator`
  → "30% of my accounts are running PMax"

- **SUI (Spend Under Influence)** = `NumeratorRevenue ÷ DenominatorRevenue`
  → "60% of my revenue comes from accounts that have PMax"

---

---

# PART 8 — Working Fast and Avoiding Mistakes

---

## 30. How to Read Any Production Script in 5 Minutes

**Step 1 — Header comment**
Read it. It tells you what the script does, what it reads, what it outputs.

**Step 2 — All `#DECLARE` at the top**
These tell you every date, flag, and file path the script uses.

**Step 3 — `#IF` blocks**
Understand which sections are active (check which flags are `true`).

**Step 4 — Data sources**
Find every `SSTREAM`, `VIEW`, and `EXTRACT` — these are where data enters.

**Step 5 — Follow the variable name through its re-assignments**
`Campaigns =` ... `Campaigns =` ... `Campaigns =` — each one refines the data.

**Step 6 — Find all `OUTPUT`**
These are what the script produces. Everything else is building up to these.

---

## 31. Standard Script Structure — Always Follow This Order

```
── TOP ──────────────────────────────────────────
1. MODULE import (privacy module)
2. #DECLARE runtime params  (@@RUN_DATE@@, @@Days@@)
3. #DECLARE feature flags   (Demand=true, Performance=true, DimTable=true)
4. #IF guard + #ERROR       (crash early if invalid combination)
5. #DECLARE computed dates  (END_DATE, StartDateTime, EndDateTime)
6. #DECLARE project paths   (BasePath, FinalFileFolder, IntermediateFolder)
7. #DECLARE all input paths (views, streamsets)
8. #DECLARE all output paths (all file paths declared together at top)

── SECTIONS ──────────────────────────────────────
9.  #IF(@Demand)
        Read SSTREAM demand data
        Filter and clean
        Join customer + account + campaign
        Calculate adoption metrics
        Incremental append to history
        OUTPUT demand files
    #ENDIF

10. #IF(@Performance)
        VIEW read → Impressions
        VIEW read → Clicks
        VIEW read → Conversions
        Deduplicate conversions (cap at 1 per RGUID+ClickId)
        SKEWJOIN clicks to conversions
        UNION ALL impressions + clicks → GROUP BY to merge
        Normalise campaign type IDs with IF()
        Add history if EXISTS()
        OUTPUT performance files
    #ENDIF

11. #IF(@DimTable)
        EXTRACT each mapping file (segment, vertical, country, AE, AM)
        FIRST() deduplicate each mapping
        Multi-LEFT JOIN all mappings onto customer list
        FIRST() deduplicate final dimension
        OUTPUT dimension table
    #ENDIF

── BOTTOM ────────────────────────────────────────
12. #CS
        C# helper functions
        (GetBiddingScheme, GetAssetType, GetChannelName)
    #ENDCS
```

---

## 32. Common Errors and Fixes

| What goes wrong | Why it happens | How to fix it |
|---|---|---|
| Type mismatch on JOIN | `CampaignId` is `int` in one table, `long` in another | Add `Convert.ToInt64(CampaignId)` before the join |
| Script crashes on null | Column has null and you did math on it | Use `?? 0` default or declare column as `int?` |
| Divide by zero | Calculating CTR or ROAS when clicks or spend is 0 | `(denom == 0 ? 0.0 : num / denom)` |
| Join is very slow or hangs | One key (like RGUID) appears millions of times | Add `[SKEWJOIN=(SKEW=FROMLEFT, REPARTITION=FULLJOIN)]` |
| File not found crashes script | A daily file is missing for a date | Use `SPARSE` in `SSTREAM SPARSE STREAMSET` |
| Privacy annotation error | OUTPUT has no `[Privacy.Asset.*]` above it | Add `[Privacy.Asset.NonPersonal]` |
| Duplicate rows in output | Multiple LEFT JOINs multiply rows | Add `FIRST()` + `GROUP BY` at the end |
| `#ERROR` fires | `#IF` guard detected an invalid flag combination | Check your `#DECLARE` flag values |
| History file crash on first run | Script tries to read a file that does not exist yet | Wrap the read in `#IF(EXISTS(@HistoryFile))` |

---

## 33. Full Quick Reference

### Scope-Only Features

| Feature | What it does |
|---|---|
| `COUNTIF(condition)` | Count rows where condition is true |
| `COUNT()` | Count all rows — no argument |
| `FIRST(col)` | Take first value in group |
| `col ?? "default"` | Replace null with a default |
| `IF(c, a, b)` | Return a if c is true, else b |
| `EXISTS(@path)` | True if file exists |
| `c ? a : b` | Same as IF — C# style |
| `SSTREAM SPARSE STREAMSET` | Read many daily files across a date range |
| `CLUSTERED BY / SORTED BY` | Organise SSTREAM for fast reads |
| `WITH STREAMEXPIRY "30"` | Auto-delete file after N days |
| `#DECLARE` | Script-level variable |
| `@@PARAM@@` | Runtime-injected scheduler value |
| `#IF / #ENDIF / #ERROR` | Compile-time conditionals |
| `#CS / #ENDCS` | Inline C# helper functions |
| `[Privacy.Asset.NonPersonal]` | Required annotation above every OUTPUT |
| `[LOWDISTINCTNESS(...)]` | Mark low-variety columns for privacy scanner |
| `[SKEWJOIN]` | Fix slow joins with uneven data distribution |

---

*All concepts from the real PMax Performance production script — explained simply*
