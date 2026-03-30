# Scope Script — Complete Guide
### Based on Your 8 Real Production Scripts | Beginner to Advanced | Every Concept Explained

---

## What is Scope Script? (Read This First)

You are working on **Microsoft Ads PMax Performance** project.
The data — billions of ad impressions, clicks, conversions — is too big for Excel or a normal database.

**Scope Script** runs on Microsoft's Cosmos cluster — thousands of computers at the same time.
It reads giant files, processes them together, and writes results that Power BI reads.

If you know SQL → you know 70% of Scope already.
The other 30% is what this guide teaches you, with **real examples from your exact scripts**.

---

## How Your 8 Scripts Work Together

```
Script 1 (Base)         → Reads raw PMax data → Creates FACT_PMaxPerformance.txt + DIM_Customer.txt
Script 2 (BidStrategy)  → Same as Script 1 but adds BidStrategyType column
Script 3 (Key Metrics)  → Reads FACT files → Calculates Adoption, SUI, Revenue Share reports
Script 4 (New CID)      → Finds new customers who just started PMax → Tracks them historically
Script 6 (BCC/Budget)   → Reads Budget Constraint data → Tracks budget health per customer
Script 7 (Top100 Budget)→ Finds top 100 customers by weekly PMax budget change
Script 8 (WoW Spend)    → Week-over-Week spend comparison for top customers
Script 9 (MoM Budget)   → Month-over-Month budget comparison for PMax campaigns
```

Think of it as a **pipeline**: Script 1 creates the base data → Scripts 3-9 read it and create reports.

---

---

# CHAPTER 1 — Every Script Starts Here

---

## 1.1 The Header Comment Block

**What is it?**
Every production script must start with this comment block. It is documentation — who wrote it, what it does, where it lives.

**Why do we use it?**
Teams review hundreds of scripts. Without a header, nobody knows what the script does or who to contact when something breaks.

```sql
/**********************************************************
1. Script Name    : ProjectName|Feature|Version|Date
2. Client Name    : Client Name
3. Project Name   : Project Name
4. Author's Name  : Your Name (alias)
5. Reviewer's Name: Reviewer Name (alias)
6. Description    : What this script does in one line
7. Stream expiry  : 30 days
8. GIT Location   : /TeamFolder/Project/Scripts/
9. Script in GIT  : ScriptName.script
10-13. Reasons for not using cubes/UI : Data not available
14. Additional Comments : NA
**********************************************************/
```

---

## 1.2 MODULE and USING — Import the Privacy System

**What is it?**
`MODULE` imports an external library. `USING` makes its namespace available in the script.

**Why do we use it?**
The privacy annotation system (`[Privacy.Asset.NonPersonal]`) is defined in a shared module. Without importing it, the annotations do not work and the script fails compliance review.

```sql
// From your Script 1, 2, 3, 4, 6, 7, 8, 9 — every single script has this at the top
MODULE "/local/Team/BI/PrivacyAnnotation/PrivacyAnnotation.module";
USING Privacy;
```

**Rule:** These two lines go at the very top of every script, immediately after the header comment.

---

## 1.3 `#DECLARE` — Naming a Value

**What is it?**
Creates a named box that holds one value. You give it a name, a data type, and a value. Then you use `@Name` anywhere below.

**Why do we use it?**
If your date is used in 15 places and changes tomorrow, you fix it in one place — not 15.

```sql
// Basic syntax
#DECLARE  VariableName  datatype  =  value;

// From your Script 1:
#DECLARE START_DATE  string  =  @@RUN_DATE@@;
#DECLARE Days        int     =  @@Days@@;

// From your Script 4 (hardcoded dates — used when NOT running via scheduler):
#DECLARE START_DATE  string  =  "2026-02-28";
#DECLARE END_DATE    string  =  "2026-02-28";

// From your Script 9 (monthly script):
#DECLARE CURRENTMONTH_STARTDATE  string  =  "2026-01-01";
```

**Use `@Name` to reference the variable:**
```sql
FROM @FACT_PMaxPerformance   // uses the declared path
WHERE Date == @START_DATE    // uses the declared date
```

---

## 1.4 `@@PARAM@@` — Value Injected by the Job Scheduler

**What is it?**
Double `@@` means: "the job scheduler will fill this in at runtime."
Scripts 1 and 2 use this because they run every day automatically.

**Why do we use it?**
The scheduler says "today is 2026-02-28" — the script does not hardcode the date.

```sql
// From Script 1 and 2 — production scripts run by scheduler
#DECLARE START_DATE  string  =  @@RUN_DATE@@;
#DECLARE Days        int     =  @@Days@@;

// Scripts 3, 4, 6, 7, 8, 9 use hardcoded dates because they run manually
#DECLARE START_DATE  string  =  "2026-02-28";  // Change daily
```

**When testing:** Replace `@@RUN_DATE@@` with a hardcoded date, test, then put it back.

---

## 1.5 Date Math — Building Dates From Other Dates

**What is it?**
Using C# DateTime methods to calculate new dates from declared dates.

**Why do we use it?**
If START_DATE is given, you can automatically calculate END_DATE, PREV_DATE, month boundaries — without hardcoding each one.

```sql
// From Script 1 — END_DATE = START_DATE + (Days - 1)
#DECLARE END_DATE string = DateTime.Parse(@START_DATE).AddDays(@Days - 1).ToString("yyyy-MM-dd");

// From Script 4 — yesterday
#DECLARE PREV_DATE string = Convert.ToDateTime(@START_DATE).AddDays(-1).ToString("yyyy-MM-dd");

// From Script 6 — tomorrow (for GI data)
#DECLARE END_DATE1 string = Convert.ToDateTime(@END_DATE).AddDays(+1).ToString("yyyy-MM-dd");

// From Script 7 — 2-week range, calculating week boundaries
#DECLARE START_DATE    string = Convert.ToDateTime(@END_DATE).AddDays(-13).ToString("yyyy-MM-dd");
#DECLARE PREV_WEEK     string = Convert.ToDateTime(@START_DATE).AddDays(+6).ToString("yyyy-MM-dd");
#DECLARE Current_WEEK  string = Convert.ToDateTime(@END_DATE).AddDays(-6).ToString("yyyy-MM-dd");

// From Script 9 — monthly script
#DECLARE CURRENTMONTH_ENDDATE   string = Convert.ToDateTime(@CURRENTMONTH_STARTDATE).AddMonths(1).AddDays(-1).ToString("yyyy-MM-dd");
#DECLARE PREVIOUSMONTH_STARTDATE string = Convert.ToDateTime(@CURRENTMONTH_STARTDATE).AddMonths(-1).ToString("yyyy-MM-dd");
#DECLARE PREVIOUSMONTH_ENDDATE  string = Convert.ToDateTime(@CURRENTMONTH_STARTDATE).AddDays(-1).ToString("yyyy-MM-dd");

// From Script 3 — extract just year, month, day as strings
#DECLARE Year  string = Convert.ToDateTime(@START_DATE).ToString("yyyy");
#DECLARE Month string = Convert.ToDateTime(@START_DATE).ToString("MM");
#DECLARE Day   string = Convert.ToDateTime(@START_DATE).ToString("dd");

// Full DateTime objects (needed when calling VIEWs)
#DECLARE StartDateTimeUtc  DateTime = DateTime.Parse(@START_DATE + " 00:00");
#DECLARE EndDateTimeUtc    DateTime = DateTime.Parse(@END_DATE   + " 23:59");
#DECLARE EndDateTimeUtc1   DateTime = DateTime.Parse(@END_DATE   + " 23:00");
```

**Key methods:**

| Method | What it does | Example |
|---|---|---|
| `.AddDays(n)` | Add n days (negative = subtract) | `AddDays(-1)` = yesterday |
| `.AddMonths(n)` | Add n months | `AddMonths(-1)` = previous month |
| `.AddYears(n)` | Add n years | `AddYears(-1)` = last year |
| `.ToString("yyyy-MM-dd")` | Format as string | `"2026-02-28"` |
| `.ToString("MM/dd/yyyy")` | US format | `"02/28/2026"` |
| `.ToString("yyyy/MM")` | Year-month only | `"2026/02"` |
| `.ToString("yyyy")` | Year only | `"2026"` |
| `.ToString("MM")` | Month only | `"02"` |

---

## 1.6 Building File Paths Dynamically

**What is it?**
Constructing file path strings that include dates or other variables.

**Why do we use it?**
Each day's output goes to a different file. Instead of hardcoding paths, you build them from date variables.

```sql
// Method 1: string.Format — from Script 1
// {0} = first arg, {1} = second arg, etc.
#DECLARE PMaxPerformance string = string.Format(
    @"{0}/{1}/{2}_{3}/PmaxPerformance.ss",
    @ProjectBasePath,       // {0}
    @FinalFileFolder,       // {1}
    @START_DATE,            // {2}
    @END_DATE               // {3}
);
// Result: /local/Team/MonPro/.../2026-02-28_2026-02-28/PmaxPerformance.ss

// Method 2: String concatenation with + — from Scripts 4, 6, 7
#DECLARE Out string = @"local/Team/MonPro/Projects/.../PMaxCustomerCountV4_" + @END_DATE + ".ss";
// Result: ...PMaxCustomerCountV4_2026-02-28.ss

// From Script 6 — multiple files built the same way
#DECLARE PMaxBCCOutss   string = @"/path/PMaxBCC_new" + @END_DATE + ".ss";
#DECLARE PMaxBCCHist    string = @"/path/PMaxBCC_new" + @PREV_DATE + ".ss";
// Today's output uses END_DATE, yesterday's history uses PREV_DATE
```

---

## 1.7 `#IF` / `#ENDIF` / `#ERROR` — Conditional Compilation

**What is it?**
`#IF` checks a condition before the script compiles. Code inside only runs if the condition is true.

**Why do we use it?**
Script 1 has three sections: Demand, Performance, DimTable. Flags let you run only what you need.

```sql
// From Script 1 — flags declared at top
#DECLARE Demand      bool = true;
#DECLARE Performance bool = true;
#DECLARE DimTable    bool = true;
#DECLARE AddHistory  bool = true;

// Guard — crash early if no valid section is enabled
#IF(!@Demand AND !@Performance)
    #ERROR "Invalid Flags";  // ! means NOT
#ENDIF

// Conditional section — only runs if Demand = true
#IF(@Demand)
    Customers = SSTREAM SPARSE STREAMSET ...;
    // ... all demand processing code
#ENDIF

// Conditional section — only runs if Performance = true
#IF(@Performance)
    MVImp = VIEW @MonetizationView PARAMS (...);
    // ... all performance processing code
#ENDIF

// Conditional section — only runs if DimTable = true
#IF(@DimTable)
    DIM_Customer = VIEW @C2CViewPath PARAMS (...);
    // ... dimension table building code
#ENDIF
```

**`#IF` inside a SELECT** — from Script 1 and 2:

When running a single day (`Days == 1`), use the declared date as the Date column.
When running a range, use the actual Date column from the data.

```sql
FACT_PMaxPerformance =
    SELECT
    #IF(@Days == 1)
        @EndDateTimeUtc.ToString("MM/dd/yyyy") AS Date,  // single day: force the date
    #ELSE
        Date,                                             // range: use column from data
    #ENDIF
    CustomerId,
    SUM(Impressions) AS Impressions
    FROM MVData
    GROUP BY Date, CustomerId;
```

---

## 1.8 File Expiry Settings

**What is it?**
Every output file gets an expiry setting — after that many days, the cluster deletes it automatically.

**Why do we use it?**
Cluster storage is not free. Old files must be cleaned up. Teams enforce this in code review.

```sql
// From Script 1
#DECLARE FinalFileExp        string = "30";   // Final outputs → 30 days
#DECLARE IntermediateFileExp string = "7";    // Intermediate files → 7 days

// From Scripts 3, 4, 6 — longer retention for reports used in Power BI
WITH STREAMEXPIRY "89"   // ~3 months — used for Power BI dashboard files
```

---

---

# CHAPTER 2 — Reading Data

---

## 2.1 `EXTRACT` — Read a Plain Text File

**What is it?**
Reads a text file (CSV, TSV, pipe-delimited) line by line. You define every column name and type.

**Why do we use it?**
Mapping files (customer segments, country lists, AE/AM assignments) are plain text. Also used to read previously output `.txt` files back into a script.

```sql
// From Script 1 — reading customer segment mapping
#DECLARE CustomerSegment string = @"/local/Team/BI/Mapping/AdvertiserDerivedSegment.txt";

CustomerSegment =
    EXTRACT CustomerId  : int,
            SegmentName           // No type = string (default)
    FROM @CustomerSegment
    USING DefaultTextExtractor();


// From Script 3 — reading FACT_PMaxPerformance.txt back into a script
PMaxPerf =
    EXTRACT Date                         : string,
            CustomerId                   : int?,      // ? means nullable — can be null
            MediumId                     : int,
            CampaignTypeId               : uint,      // unsigned int — no negative values
            AdDisplayTypeId              : int,
            MarketplaceClassificationId  sbyte?,      // small nullable signed byte
            Impressions                  : long,      // large number
            Clicks                       : long,
            Revenue                      : decimal,   // money — high precision
            Conversions                  : long,
            ConversionEnabledClicks      : long,
            ConversionEnabledRevenue     : decimal,
            GrossProfit                  : decimal
    FROM @FACT_PMaxPerformance
    USING DefaultTextExtractor();


// From Script 2 — skipping the header row (when your file has a header)
EXTRACT Date         : string,
        CustomerId   : int?,
        CampaignTypeId : uint,
        BidStrategyType : byte
FROM @FACT_PMaxPerformancebid
USING DefaultTextExtractor(skipFirstNRows: 1);   // skip the column header row


// From Script 3 — reading dimension table
DIM_Customers =
    EXTRACT CustomerId    : int?,
            CustomerName  : string,
            Segment       : string,
            L1            : string,
            ServiceCountry: string,
            AE            : string,
            AM            : string
    FROM @DIM_Customer
    USING DefaultTextExtractor();
```

**Data Types Reference:**

| Type | Stores | Use When |
|---|---|---|
| `string` | Any text | Names, dates as text, paths |
| `int` | Whole number up to ~2 billion | CustomerId, AccountId |
| `int?` | int OR null | CustomerId that might be missing |
| `long` | Very large whole number | Impressions, Clicks (can be billions) |
| `double` | Decimal number | Ratios, rates |
| `decimal` | High precision decimal | Revenue, Budget (money) |
| `double?` | decimal OR null | Target percentages that might be missing |
| `uint` | Unsigned int (no negatives) | CampaignTypeId |
| `byte` | Small number 0-255 | BidStrategyType |
| `byte?` | byte OR null | BiddingSchemeId (nullable) |
| `sbyte?` | Small signed byte OR null | MarketplaceClassificationId |
| `bool` | True / False | IsActive, IsServable |
| `DateTime` | Date and time | Run dates |

**The `?` means nullable.** Use it whenever the column might have empty/missing values.
Without it — one null value crashes the entire script.

---

## 2.2 `SSTREAM` — Read One Binary File

**What is it?**
Reads a single `.ss` (structured stream) binary file. No schema needed — it is stored inside.

**Why do we use it?**
Reading previous output files back into a script, or reading intermediate `.ss` files produced by an earlier script run.

```sql
// From Script 3 — read the adoption history file
FACT_AdoptionSUI =
    SELECT *
    FROM
    (
        SSTREAM @FACT_AdoptionSUI     // reads the .ss file declared at top
    );


// From Script 4 — read yesterday's history file
Hist =
    SELECT *
    FROM
    (
        SSTREAM @Hist    // @Hist path was built using PREV_DATE
    );
```

---

## 2.3 `SSTREAM SPARSE STREAMSET` — Read Many Daily Files at Once

**What is it?**
Reads ALL daily `.ss` files across a date range in one command. This is the most used data read pattern in all your scripts.

**Why do we use it?**
Data is organized into daily folders on the cluster:
```
/shares/.../Data/UnifiedLayerDemand/2026/02/28/Campaigns.ss
/shares/.../Data/UnifiedLayerDemand/2026/02/27/Campaigns.ss
/shares/.../Data/UnifiedLayerDemand/2026/02/26/Campaigns.ss
```
Reading each separately would need 28 statements for one month.
STREAMSET does it in one.

```sql
// From Script 1 — reads Customers.ss for every day in range
#DECLARE STCA_Basepath string = @"/shares/bingAds.BI.OI/AdsOI/DemandMetrics/Data/";
#DECLARE Customers     string = "UnifiedLayerDemand/%Y/%m/%d/Customers.ss";

Customers =
    SSTREAM SPARSE STREAMSET @STCA_Basepath
        PATTERN @Customers
        RANGE __date = [@START_DATE, @END_DATE];

// From Script 1 — reads Campaigns.ss for every day in range
#DECLARE Campaigns string = "UnifiedLayerDemand/%Y/%m/%d/Campaigns.ss";

Campaigns =
    SSTREAM SPARSE STREAMSET @STCA_Basepath
        PATTERN @Campaigns
        RANGE __date = [@START_DATE, @END_DATE];

// From Script 6 — reads demand fact data (different base path)
#DECLARE DemandSourceBase string = @"/shares/Bingads.marketplace.prod.tools/Demand/DemandDaily/v2";
#DECLARE Pattern          string = @"/%Y/%m/DemandFact_Serving_%Y%m%d0000.ss";

Perf_data =
    SELECT __date AS Date, Customer AS CustomerId, ...
    FROM
    (
        SSTREAM SPARSE STREAMSET @DemandSourceBase
            PATTERN @Pattern
            RANGE __date = [@START_DATE, @END_DATE]
    );
```

**Pattern placeholders:**
- `%Y` → 4-digit year: `2026`
- `%m` → 2-digit month: `02`
- `%d` → 2-digit day: `28`

**`SPARSE`** = skip missing dates silently. Without it, one missing file crashes the script.

**`__date`** = a special built-in column that contains the date of the file being read.
Used as: `SELECT __date AS Date` to get the file date into your data.

---

## 2.4 `VIEW with PARAMS` — Read a Managed Data Source

**What is it?**
A VIEW is a pre-built logical data source maintained by a central team. You call it with parameters — you do not need to know where the files actually are.

**Why do we use it?**
The Monetization VIEW gives access to impressions, clicks, and conversions. The CampaignAggregates VIEW gives campaign budgets. These are complex data sources — the VIEW handles the complexity.

### The Monetization View — 3 Separate Reads

Scripts 1 and 2 read the same view three times with different flags:

```sql
// From Scripts 1 and 2

// READ 1: Impressions only
MVImp =
    VIEW @MonetizationView
    PARAMS
    (
        StartDateTimeUtc  = @StartDateTimeUtc,
        EndDateTimeUtc    = @EndDateTimeUtc1,
        ReadPageViews     = true,
        ReadAdImpressions = true,     // ← read impressions
        ReadSessions      = false,
        ReadClicks        = false,
        FraudType         = "NonFraud"  // ← always filter fraud at source
    );

// READ 2: Clicks only
MVClick =
    VIEW @MonetizationView
    PARAMS
    (
        StartDateTimeUtc  = @StartDateTimeUtc,
        EndDateTimeUtc    = @EndDateTimeUtc1,
        ReadPageViews     = false,
        ReadAdImpressions = false,
        ReadSessions      = false,
        ReadClicks        = true,     // ← read clicks
        FraudType         = "NonFraud"
    );

// READ 3: Conversions only
MVConv =
    VIEW @MonetizationView
    PARAMS
    (
        StartDateTimeUtc   = @StartDateTimeUtc,
        EndDateTimeUtc     = @EndDateTimeUtc1,
        ReadClicks         = false,
        ReadConversions    = true,    // ← read conversions
        ReadUETConversions = true,    // ← include UET (Universal Event Tracking)
        ReadPageViews      = false,
        ReadAdImpressions  = false
    );
```

### The CampaignAggregates View — Budget Data

Scripts 7 and 9 read campaign budgets from a different view:

```sql
// From Scripts 7 and 9
#DECLARE CampaignAggregates string = @"/shares/adCenter.BICore.SubjectArea/.../DemandCampaignAggregatesV2.view";

CampaignAggregates_Base =
    VIEW @CampaignAggregates
    PARAMS
    (
        StartDateUTC = @StartDateTimeUtc,
        EndDateUTC   = @EndDateTimeUtc1,
        LoadDlls     = true              // required for this view
    );

// Then select from it
Budget =
    SELECT GetDate(Convert.ToString(Date)) AS Date,
           CustomerId,
           Convert.ToString(CampaignId) AS CampaignId,
           SUM(CampaignIncrementalBudgetAmountUSD) AS Budget
    FROM CampaignAggregates_Base
    GROUP BY Date, CustomerId, CampaignId;
```

---

---

# CHAPTER 3 — Transforming Data

---

## 3.1 SELECT and the Re-assignment Pattern

**What is it?**
SELECT works like SQL. The key Scope pattern is assigning to the **same variable name multiple times** — each time refining the data step by step.

**Why do we use it?**
Instead of one giant complicated query, you build the result in steps. Each step is readable on its own.

```sql
// From Script 1 — Campaigns variable is assigned 4 times, each step refines it

// STEP 1: Read raw data
Campaigns =
    SSTREAM SPARSE STREAMSET @STCA_Basepath PATTERN @CampaignPattern RANGE __date = [@START_DATE, @END_DATE];

// STEP 2: Pick only needed columns, convert types
Campaigns =
    SELECT DISTINCT __date                      AS Date,
                    AccountId,
                    Convert.ToInt64(CampaignId) AS CampaignId,  // int → long
                    AdvertisingChannelTypeId,
                    BiddingSchemeId,
                    IsEligible,
                    IsServable
    FROM Campaigns;

// STEP 3: Join with accounts to get customer info
Campaigns =
    SELECT Date, CustomerId, CustomerName, AccountId, CampaignId,
           AdvertisingChannelTypeId, BiddingSchemeId, IsEligible, IsServable
    FROM Accounts AS A
    INNER JOIN Campaigns AS B ON A.AccountId == B.AccountId;

// Now Campaigns has everything: dates, customer info, campaign details
```

---

## 3.2 Aggregate Functions

### `SUM()` — Add up all values

```sql
SUM(Impressions)    AS TotalImpressions
SUM(Revenue)        AS TotalRevenue
SUM(AdSpend)        AS TotalSpend
SUM(PMaxBCC)        AS TotalBCC        // from Script 6
```

### `COUNT()` — Count all rows (no argument in Scope)

```sql
COUNT() AS TotalCampaigns

// NOT: COUNT(*) — in Scope it is just COUNT()
```

### `COUNT(DISTINCT col)` — Count unique values

```sql
// From Script 3
COUNT(DISTINCT CustomerId)  AS NoofCustomers
COUNT(DISTINCT CampaignId)  AS NoOfCampaigns    // from Script 1
```

### `COUNTIF(condition)` — Count rows where condition is true ← Scope Only

**What is it?**
Counts only rows that match your condition. Other rows are ignored. This does NOT exist in standard SQL.

**Why do we use it?**
In Script 1, to count PMax campaigns per customer without a separate filter step.

```sql
// From Script 1 — count PMax campaigns per customer per account
Adoption =
    SELECT Date,
           CustomerId,
           AccountId,
           COUNTIF(AdvertisingChannelTypeId == 9)  AS PMaxCnt,   // how many PMax campaigns
           SUM(Revenue)                              AS Revenue
    FROM Adoption
    GROUP BY Date, CustomerId, AccountId;
```

### `FIRST()` — Take first value in a group ← Scope Only

**What is it?**
When you GROUP BY, all non-aggregated columns must use an aggregate. `FIRST()` takes the first value it sees in the group.

**Why do we use it?**
CustomerName is the same for all rows of the same CustomerId. `FIRST()` picks it without error.

```sql
// From Script 1 — building DIM_Customer with FIRST()
DIM_Customer =
    SELECT CustomerId,
           FIRST(CustomerName) AS CustomerName,
           FIRST(Segment)      AS Segment,
           FIRST(L1)           AS L1,
           FIRST(ServiceCountry) AS ServiceCountry,
           FIRST(AE)           AS AE,
           FIRST(AM)           AS AM
    FROM DIM_Customer
    GROUP BY CustomerId;

// From Script 6 — deduplicating segment mapping
Cust_Segment =
    SELECT CustomerId,
           FIRST(SegmentName)  AS Segment
    FROM Cust_Segment
    GROUP BY CustomerId;
```

---

## 3.3 `IF()` — Choose a Value Based on a Condition

**What is it?**
`IF(condition, value_if_true, value_if_false)` — picks one of two values.

**Why do we use it?**
Normalising campaign type IDs, calculating conditional sums, creating flags.

```sql
// From Script 1 — normalise campaign type IDs
SELECT
    // Smart Shopping (type 3, subtype 12) gets a combined ID of 312
    IF(CampaignTypeId == 3 AND CampaignSubTypeId == 12, 312, CampaignTypeId)
        AS CampaignTypeId,

    // PMax campaigns keep AdDisplayTypeId; others get -9999
    IF(CampaignTypeId == 9, AdDisplayTypeId, -9999)
        AS AdDisplayTypeId,

    // MediumId normalisation: 1 and 3 both map to 3
    IF(MediumId IN(1, 3), 3, MediumId)
        AS MediumId,

    // Is this campaign PMax?
    IF(AdvertisingChannelTypeId != 9, false, true)
        AS IsPMax

FROM MVData;


// From Script 3 — conditional SUM (sum only PMax revenue)
SELECT
    SUM(IF(CampaignTypeId == 9, Revenue, 0))              AS PMaxRevenue,
    SUM(Revenue)                                           AS TotalRevenue,
    SUM(IF(MediumId IN (10, 11) AND CampaignTypeId == 9, Revenue, 0)) AS PMaxMSANRevenue,
    SUM(IF(MediumId IN (10, 11), Revenue, 0))             AS MSANRevenue
FROM PMaxPerf
GROUP BY Date, CustomerId, CampaignTypeId;


// From Script 6 — handle empty/null segment
IF(String.IsNullOrEmpty(Segment), "Unsegmented", Segment) AS Segment


// From Script 8 — calculate WoW delta directly in SELECT
WoWDeltaPMaxRevenue = (CurrentWeekPMaxRevenue - PreviousWeekPMaxRevenue)
```

### Nested IF — Multiple Conditions

```sql
// From Script 1 — IsSS (Smart Shopping) takes priority over IsPMax
IF(IsSS,   312,
IF(IsPMax, AdvertisingChannelTypeId,
           -9999))
AS AdvertisingChannelTypeId
```

### C# Ternary Style `?:` — Same as IF

```sql
// From Script 1 — if IsPMax OR IsSS, use BiddingSchemeId; else -1
(IsPMax | IsSS) ? BiddingSchemeId : -1  AS BiddingSchemeId
```

---

## 3.4 `??` — Replace Null with a Default Value

**What is it?**
`column ?? "default"` means: if null, use default value instead.

**Why do we use it?**
After LEFT OUTER JOINs, unmatched rows have null in the right-table columns. `??` fills them.

```sql
// From Script 1 — dimension table build
DIM_Customer =
    SELECT CustomerId,
           CustomerName    ?? "Unknown"       AS CustomerName,
           Segment         ?? "Unsegmented"   AS Segment,
           L1              ?? "Unknown"       AS L1,
           ServiceCountry  ?? "Unknown"       AS ServiceCountry,
           AE              ?? "Unknown"       AS AE,
           AM              ?? "Unknown"       AS AM
    FROM Customers AS A
    LEFT OUTER JOIN DIM_Customer AS AA ON A.CustomerId == AA.CustomerId
    LEFT OUTER JOIN CustomerSegment AS B  ON A.CustomerId == B.CustomerId
    LEFT OUTER JOIN CustomerVertical AS C ON A.CustomerId == C.CustomerId
    LEFT OUTER JOIN CustomerCountry AS D  ON A.CustomerId == D.CustomerId
    LEFT OUTER JOIN CustomerAE AS E       ON A.CustomerId == E.CustomerId
    LEFT OUTER JOIN CustomerAM AS F       ON A.CustomerId == F.CustomerId;
```

---

## 3.5 `Convert.*` — Change a Column's Data Type

**What is it?**
Changes the data type of a value.

**Why do we use it?**
Type mismatches cause JOIN failures and crashes. `CampaignId` might be `int` in one source and `long` in another.

```sql
// From Script 1
Convert.ToInt64(CampaignId)                              // int → long (large IDs)
Convert.ToInt32(CustomerId)                              // long → int
Convert.ToInt32(BiddingSchemeId)                         // byte → int
Convert.ToString(CampaignId)                             // any → string
Convert.ToString(CustomerId)                             // int → string
Convert.ToDateTime(Date).ToString("MM/dd/yyyy")          // string → DateTime → formatted string
Convert.ToDateTime(Date).ToString("yyyy-MM-dd")          // reformat date string
Convert.ToDateTime(Date).ToString("yyyy/MM")             // year-month only (Script 3)
Convert.ToDateTime(Date).ToString("M/d/yyyy")            // short format, no leading zeros (Script 4)
Convert.ToDouble(PMaxRevenue)                            // decimal → double (Script 3)
```

### Literal Type Casting

Used in the UNION ALL merge pattern:

```sql
(long) 0        AS Clicks        // zero as a long integer — matches Clicks column type
(decimal) 0.0   AS Conversions   // zero as decimal — matches Conversions column type
(double) 0      AS PmaxTargetRevShare  // zero as double — from Script 3
```

---

## 3.6 `WHERE` — Filter Rows

```sql
// Date filters
WHERE Date >= @START_DATE AND Date <= @END_DATE
WHERE Date == @START_DATE              // exact match
WHERE Date >= @Current_WEEK AND Date <= @END_DATE  // from Script 8

// Channel filters
WHERE AdvertisingChannelTypeId == 9    // PMax only
WHERE AdvertisingChannelTypeId IN (null, 1, 3, 6, 9)  // multiple channels incl null
WHERE CampaignType == 9 AND ServedBillableStatus > 0   // from Script 6

// Quality filter — ALWAYS include for Monetization VIEW data
WHERE FraudQualityBand >= 2            // removes fraudulent traffic (bands 0,1)

// Revenue/spend filters
WHERE Revenue > 0
WHERE Budget > 0

// Status filters
WHERE IsServable == true               // from Scripts 7, 9
WHERE IsEligible == true               // from Script 1

// Null safety
WHERE Denominator > 0                  // from Script 1 adoption

// String comparison
WHERE Date == @START_DATE              // exact date match
WHERE Shortdate >= "2025/01"           // from Script 3 — date range on formatted string

// Segment filters
WHERE Segment == "Channel Partner"
WHERE Segment NOT IN ("Channel Partner", "Corporate", "Enterprise", "SMB", "Strategic")
WHERE Region == "APAC"
WHERE Subsegment == "Top SMB"
WHERE Subsegment NOT IN ("Top SMB")    // exclude specific values

// Existence filter — from Script 4
WHERE LogDate == CreatedDateTime       // new customers only (same day created and logged)
```

**The most important filter — never skip it:**
```sql
WHERE FraudQualityBand >= 2    // bands 0 and 1 = fraudulent clicks/impressions
```

---

## 3.7 `SELECT DISTINCT` — Remove Duplicate Rows

```sql
// From Script 1
Customers =
    SELECT DISTINCT CustomerId, CustomerName
    FROM Customers;

Campaigns =
    SELECT DISTINCT __date AS Date, AccountId, CampaignId, AdvertisingChannelTypeId
    FROM Campaigns;

// From Script 4
PMaxCampaigns =
    SELECT DISTINCT CustomerId, __date AS Date
    FROM Campaigns
    WHERE AdvertisingChannelTypeId == 9;
```

---

## 3.8 `SELECT *` — Select All Columns

```sql
// From Script 3
FACT_AdoptionSUI = SELECT * FROM (SSTREAM @FACT_AdoptionSUI);

// From Script 1 with A.* — all columns from table A
CampaignDemand =
    SELECT A.*,
           IF(B.CampaignId != null, true, false) AS IsSS
    FROM CampaignDemand AS A
    LEFT OUTER JOIN SSCampaigns AS B ON A.CampaignId == B.CampaignId;
```

---

## 3.9 `UNION ALL` — Stack Two Tables Together

**What is it?**
Combines two tables with the same columns vertically.

**Why do we use it?**
The most critical use: merging impressions and clicks that come from separate VIEW reads.

### The Impression + Click Merge Pattern (From Scripts 1 and 2)

```sql
// STEP 1: Impressions — give zero values for click columns
MVData =
    SELECT Date, CustomerId, AdvertiserAccountId, CampaignId,
           CampaignTypeId, CampaignSubTypeId, MarketplaceClassificationId, MediumId, AdDisplayTypeId,
           Impressions,
           (long) 0      AS Clicks,
           Revenue,
           (decimal) 0.0 AS Conversions,
           (long) 0      AS ConversionEnabledClicks,
           ConversionEnabledRevenue,
           (decimal) 0.0 AS GrossProfit
    FROM MVImp                 // ← impression source

    UNION ALL

    SELECT Date, CustomerId, AdvertiserAccountId, CampaignId,
           CampaignTypeId, CampaignSubTypeId, MarketplaceClassificationId, MediumId, AdDisplayTypeId,
           (long) 0      AS Impressions,    // ← zero here, real value from MVClkConv
           Clicks,
           Revenue,
           Conversions,
           ConversionEnabledClicks,
           ConversionEnabledRevenue,
           GrossProfit
    FROM MVClkConv;            // ← click source

// STEP 2: Aggregate — SUM collapses to one row per campaign. Zeros cancel out.
MVData =
    SELECT Date, CustomerId, AdvertiserAccountId, CampaignId,
           CampaignTypeId, CampaignSubTypeId, MarketplaceClassificationId, MediumId, AdDisplayTypeId,
           SUM(Impressions)            AS Impressions,
           SUM(Clicks)                 AS Clicks,
           SUM(Revenue)                AS Revenue,
           SUM(Conversions)            AS Conversions,
           SUM(ConversionEnabledClicks)AS ConversionEnabledClicks,
           SUM(ConversionEnabledRevenue) AS ConversionEnabledRevenue,
           SUM(GrossProfit)            AS GrossProfit
    FROM MVData
    GROUP BY Date, CustomerId, AdvertiserAccountId, CampaignId,
             CampaignTypeId, CampaignSubTypeId, MarketplaceClassificationId, MediumId, AdDisplayTypeId;
```

### UNION ALL for Week Comparison (Scripts 8 and 9)

```sql
// From Script 8 — current week and previous week data combined using zero trick
CurrentWeekCust_Rev =
    SELECT CustomerId,
           CurrentWeekTotalRevenue, CurrentWeekPMaxRevenue, CurrentWeekMSANRevenue,
           (decimal) 0 AS PreviousWeekTotalRevenue,
           (decimal) 0 AS PreviousWeekPMaxRevenue,
           (decimal) 0 AS PreviousWeekMSANRevenue
    FROM CurrentWeekCust_Rev;

PreviousWeekCust_Rev =
    SELECT CustomerId,
           (decimal) 0 AS CurrentWeekTotalRevenue,
           (decimal) 0 AS CurrentWeekPMaxRevenue,
           (decimal) 0 AS CurrentWeekMSANRevenue,
           PreviousWeekTotalRevenue, PreviousWeekPMaxRevenue, PreviousWeekMSANRevenue
    FROM PreviousWeekCust_Rev;

// Stack and sum → one row per customer with both weeks
Customer_Rev =
    SELECT * FROM CurrentWeekCust_Rev
    UNION ALL
    SELECT * FROM PreviousWeekCust_Rev;

Customer_Rev =
    SELECT CustomerId,
           SUM(CurrentWeekTotalRevenue)   AS CurrentWeekTotalRevenue,
           SUM(PreviousWeekTotalRevenue)  AS PreviousWeekTotalRevenue,
           ...
    FROM Customer_Rev
    GROUP BY CustomerId;
```

### UNION ALL for Multiple Segments (Script 3)

```sql
// From Script 3 — stack many segment slices together
Final =
    SELECT * FROM GLOBAL_Strategic
    UNION ALL
    SELECT * FROM GLOBAL_Corporate
    UNION ALL
    SELECT * FROM GLOBAL_Enterprise
    UNION ALL
    SELECT * FROM GLOBAL_SMB
    UNION ALL
    SELECT * FROM APAC_Strategic
    UNION ALL
    // ... many more UNION ALL segments
    SELECT * FROM EMEA_LATAM_Totals;
```

### `UNION` (Without ALL) — Removes Duplicates

```sql
// From Script 1 — combining customer IDs from Performance and Demand, deduplicating
Customers =
    SELECT DISTINCT Convert.ToInt32(CustomerId) AS CustomerId
    FROM FACT_PMaxPerformance

    UNION     // ← removes duplicate CustomerIds that appear in both sets

    SELECT CustomerId
    FROM Customers;
```

---

## 3.10 ORDER BY — Sorting Results

```sql
// From Scripts 7, 8 — ranking customers
ORDER BY Budget DESC          // highest budget first
ORDER BY WoWDeltaPMaxRevenue DESC  // biggest revenue increase first
ORDER BY WoWDeltaPMaxRevenue ASC   // biggest revenue decrease first
```

---

---

# CHAPTER 4 — Joining Tables

---

## 4.1 `INNER JOIN` — Keep Only Matching Rows

**What is it?**
Only rows that exist in BOTH tables are kept. Rows with no match in either table are dropped.

**Why do we use it?**
Use when you only want data that has confirmed matches on both sides.

```sql
// From Script 1 — join accounts with customers (only accounts with a known customer)
Accounts =
    SELECT CustomerId, CustomerName, AccountId
    FROM Customers AS A
    INNER JOIN Accounts AS B
    ON A.CustomerId == B.CustomerId;


// From Script 4 — only customers that are ALSO in PMax campaigns
PMaxCustomers =
    SELECT A.CustomerId, A.Date
    FROM Customers AS A
    INNER JOIN PMaxCampaigns AS B
    ON A.Date == B.Date AND A.CustomerId == B.CustomerId;


// From Script 3 — combining adoption metrics from two separately calculated tables
CMSegmentlevelAdoption =
    SELECT A.Segment, A.Date, NoofCustomers, AdoptionDenomenator
    FROM NoofCustomers AS A
    INNER JOIN AdoptionDenomenator AS B
    ON A.Segment == B.Segment;


// From Script 3 — 3-way join (current + previous + 2 months ago)
AllupALdata =
    SELECT A.Segment, A.Numerator, A.Denominator,
           B.Numerator AS LastMonthNumerator, B.Denominator AS LastMonthDenominator,
           C.Numerator AS Last2MonthNumerator, C.Denominator AS Last2MonthDenominator
    FROM CurrentMonth AS A
    INNER JOIN LastMonth AS B ON A.Segment == B.Segment
    INNER JOIN Last2Month AS C ON A.Segment == C.Segment;
```

---

## 4.2 `LEFT OUTER JOIN` — Keep All Left Rows

**What is it?**
Every row from the left table is kept. Matching rows from the right table fill in the columns. If no match, right columns are null.

**Why do we use it?**
When building dimension tables: not every customer has a segment, country, AE mapping. Keep all customers, nulls are handled by `??`.

```sql
// From Script 1 — full dimension table with all mappings
DIM_Customer =
    SELECT CustomerId,
           CustomerName    ?? "Unknown"     AS CustomerName,
           Segment         ?? "Unsegmented" AS Segment,
           L1              ?? "Unknown"     AS L1,
           ServiceCountry  ?? "Unknown"     AS ServiceCountry,
           AE              ?? "Unknown"     AS AE,
           AM              ?? "Unknown"     AS AM
    FROM Customers AS A
    LEFT OUTER JOIN DIM_Customer AS AA ON A.CustomerId == AA.CustomerId
    LEFT OUTER JOIN CustomerSegment AS B  ON A.CustomerId == B.CustomerId
    LEFT OUTER JOIN CustomerVertical AS C ON A.CustomerId == C.CustomerId
    LEFT OUTER JOIN CustomerCountry AS D  ON A.CustomerId == D.CustomerId
    LEFT OUTER JOIN CustomerAE AS E       ON A.CustomerId == E.CustomerId
    LEFT OUTER JOIN CustomerAM AS F       ON A.CustomerId == F.CustomerId;


// From Script 1 — adoption: left join revenue to campaigns
Adoption =
    SELECT Date, CustomerId, AccountId, CampaignId, AdvertisingChannelTypeId, Revenue
    FROM Adoption AS A
    LEFT OUTER JOIN AccountPerformance AS B
    ON A.CampaignId == B.CampaignId AND A.Date == B.Date;


// From Script 6 — left join segment mapping to performance data
PMaxBCC =
    SELECT Segment, Date, A.CustomerId, SpendUSD, PMaxBCC
    FROM Perf_data AS A
    LEFT OUTER JOIN Cust_Segment AS B
    ON A.CustomerId == B.CustomerId;


// From Script 7 — join budget to PMax customers
PmaxCustomers_Budget =
    SELECT Date, CustomerId, CampaignId, AdvertisingChannelTypeId, Budget
    FROM PmaxCustomers AS A
    LEFT OUTER JOIN Budget AS B
    ON A.CampaignId == B.CampaignId AND A.Date == B.Date;


// From Script 3 — adding Region to customer details
DIM_CustDetails =
    SELECT A.*,
           RegionAbbreviatedName AS Region
    FROM DIM_CustDetails AS A
    LEFT OUTER JOIN Region AS B ON A.ServiceCountry == B.CountryName;
```

---

## 4.3 `[SKEWJOIN]` — Fix Slow Joins

**What is it?**
A performance hint that tells Scope to redistribute skewed data across machines before joining.

**Why do we use it?**
The Click-to-Conversion join (on RGUID + ClickId) is always skewed — large advertisers have millions of clicks with the same RGUID. Without this hint, one machine gets overloaded.

```sql
// From Scripts 1 and 2 — the click-to-conversion join always needs this
[SKEWJOIN=(SKEW=FROMLEFT, REPARTITION=FULLJOIN)]
MVClkConv =
    SELECT MBDate, RGUID, ClickId, CustomerId, AdvertiserAccountId,
           CampaignId, CampaignTypeId, CampaignSubTypeId,
           MarketplaceClassificationId, MediumId, AdDisplayTypeId,
           Clicks, Revenue, Conversions,
           ConversionEnabledClicks, ConversionEnabledRevenue, GrossProfit
    FROM MVClick AS A
    LEFT OUTER JOIN MVConv AS B
    ON A.RGUID == B.RGUID AND A.ClickId == B.ClickId;
```

- `SKEW=FROMLEFT` → MVClick (left) has the skewed data (large advertiser clicks)
- `REPARTITION=FULLJOIN` → redistribute both sides across all machines

---

## 4.4 The Click-to-Conversion Join Pattern

**From Scripts 1 and 2 — this is how conversions are connected to clicks:**

```sql
// STEP 1: Deduplicate conversions — cap at 1 per RGUID+ClickId
MVConv =
    SELECT RGUID,
           ClickId,
           // Cap at 1 — one click should produce at most 1 conversion credit
           IF(SUM(ConversionCredit) > 1, 1, SUM(ConversionCredit)) AS Conversions,
           MAX(AdvertiserReportedRevenue) AS GrossProfit
    FROM MVConv
    WHERE FraudQualityBand >= 2
    GROUP BY RGUID, ClickId;

// STEP 2: Join clicks to capped conversions (with SKEWJOIN)
[SKEWJOIN=(SKEW=FROMLEFT, REPARTITION=FULLJOIN)]
MVClkConv =
    SELECT MBDate, RGUID, ClickId, CustomerId, AdvertiserAccountId,
           CampaignId, ..., Clicks, Revenue, Conversions, GrossProfit
    FROM MVClick AS A
    LEFT OUTER JOIN MVConv AS B
    ON A.RGUID == B.RGUID AND A.ClickId == B.ClickId;

// STEP 3: Aggregate to campaign level
MVClkConv =
    SELECT Date, CustomerId, AdvertiserAccountId, CampaignId, ...,
           SUM(Clicks) AS Clicks, SUM(Revenue) AS Revenue,
           SUM(Conversions) AS Conversions, SUM(GrossProfit) AS GrossProfit
    FROM MVClkConv
    GROUP BY Date, CustomerId, AdvertiserAccountId, CampaignId, ...;
```

---

---

# CHAPTER 5 — Writing Output

---

## 5.1 `OUTPUT` — Write to a Text File

**What is it?**
Writes the result to a plain text file readable by Power BI, Excel, or another script.

**Why do we use it?**
Power BI dashboards read from `.txt` files in the output folder.

```sql
// From Script 1
[Privacy.Asset.NonPersonal]
OUTPUT
TO @FACT_Adoption
WITH STREAMEXPIRY @FinalFileExp
USING DefaultTextOutputter(outputHeader: true);    // outputHeader: true = write column names in row 1


// From Script 3 — explicit variable reference and 89-day expiry
[Privacy.Asset.NonPersonal]
OUTPUT
TO @Out1
WITH STREAMEXPIRY "89"
USING DefaultTextOutputter(outputHeader: true);


// With explicit variable name (when not the last computed variable)
[Privacy.Asset.NonPersonal]
OUTPUT FACT_PMaxPerformance    // ← explicitly name the variable to output
TO @FACT_PMaxPerformancebid
WITH STREAMEXPIRY @FinalFileExp;
```

---

## 5.2 `OUTPUT` to `SSTREAM` — Write a Fast Binary File

**What is it?**
Writes a `.ss` binary file — faster to read, smaller in size than text. Used for intermediate data or when another script reads it.

**Why do we use it?**
History files, intermediate joins, and large datasets are stored as `.ss` for performance.

```sql
// From Script 1 — standard SSTREAM output with clustering
[Privacy.Asset.NonPersonal]
OUTPUT MVData
TO SSTREAM @PMaxPerformance
    CLUSTERED BY CustomerId, AdvertiserAccountId, CampaignId
    SORTED BY CustomerId, AdvertiserAccountId, CampaignId
    WITH STREAMEXPIRY @FinalFileExp;


// From Script 1 — adoption history file
[Privacy.Asset.NonPersonal]
OUTPUT
TO SSTREAM @FACT_AdoptionSS
    CLUSTERED BY Date, CustomerId
    SORTED BY Date, CustomerId
    WITH STREAMEXPIRY @FinalFileExp;


// From Script 2 — two outputs from same dataset (text AND ss)
[Privacy.Asset.NonPersonal]
OUTPUT FACT_PMaxPerformance1
TO @FACT_PMaxPerformancebid2       // ← text file
WITH STREAMEXPIRY @FinalFileExp;

[Privacy.Asset.NonPersonal]
OUTPUT FACT_PMaxPerformance1
TO SSTREAM @FACT_PMaxPerformancebid3   // ← ss file
WITH STREAMEXPIRY @FinalFileExp;
```

**`CLUSTERED BY`** — spread data across machines by these columns (fast parallel reads)
**`SORTED BY`** — within each cluster, sort rows (fast range queries)
**`WITH STREAMEXPIRY "30"`** — auto-delete after 30 days

---

---

# CHAPTER 6 — Privacy and Compliance

---

## 6.1 `[Privacy.Asset.NonPersonal]` — Required on Every OUTPUT

**What is it?**
A compliance annotation that classifies what type of data the output contains. Required by Microsoft's privacy system.

**Why do we use it?**
Without this annotation, the script fails compliance review. Every single OUTPUT must have it.

```sql
// Wrong — output without annotation
OUTPUT @MyData TO "/path/file.txt" WITH STREAMEXPIRY "30";

// Correct — annotation immediately above OUTPUT
[Privacy.Asset.NonPersonal]
OUTPUT @MyData
TO "/path/file.txt"
WITH STREAMEXPIRY "30"
USING DefaultTextOutputter(outputHeader: true);
```

**When to use which:**

| Annotation | When |
|---|---|
| `[Privacy.Asset.NonPersonal]` | Aggregated data — no individual-level rows (most reports) |
| `[Privacy.Asset.Personal]` | Contains user-identifiable information |
| `[Privacy.Asset.Pseudonymous]` | Contains hashed/anonymised user IDs |

---

## 6.2 `[LOWDISTINCTNESS(...)]` — Mark Low-Variety Columns

**What is it?**
Tells the privacy scanner that these columns have very few unique values and cannot identify a person.

**Why do we use it?**
`CampaignTypeId` has ~15 values. Without annotation, the scanner might flag it as potentially identifying.

```sql
// From Scripts 1 and 2 — annotate before SELECT that uses these columns
[LOWDISTINCTNESS(CampaignTypeId, CampaignSubTypeId, MarketplaceClassificationId,
                 MediumId, AdDisplayTypeId, MBDate, CustomerId, AdvertiserAccountId)]
MVImp =
    SELECT MBDate, CustomerId, AdvertiserAccountId, CampaignId,
           CampaignTypeId, CampaignSubTypeId, MarketplaceClassificationId, MediumId, AdDisplayTypeId,
           SUM(ImpressionCnt) AS Impressions,
           SUM(AmountChargedConstantUSDExchangeRt) AS Revenue
    FROM MVImp
    WHERE FraudQualityBand >= 2;
```

---

---

# CHAPTER 7 — Advanced Patterns

---

## 7.1 `EXISTS()` — Check if a File Exists Before Reading

**What is it?**
`EXISTS(@path)` returns true if the file exists, false if it does not.

**Why do we use it?**
Scripts that run every day append to a history file. On the first ever run, that history file does not exist. Without `EXISTS()`, the script crashes trying to read something that is not there.

```sql
// From Script 1 — only try to read history if it already exists
#IF(@AddHistory AND EXISTS(@FACT_PMaxPerformance))
    FACT_PMaxPerformance =
        SELECT * FROM FACT_PMaxPerformance
        UNION ALL
        SELECT *
        FROM
        (
            EXTRACT Date : string, CustomerId : int?, MediumId : int, ...
            FROM @FACT_PMaxPerformance
            USING DefaultTextExtractor()
        )
        WHERE !(Convert.ToDateTime(Date) >= Convert.ToDateTime(@START_DATE)
             AND Convert.ToDateTime(Date) <= Convert.ToDateTime(@END_DATE));

    [Privacy.Asset.NonPersonal]
    OUTPUT FACT_PMaxPerformance
    TO @FACT_PMaxPerformance
    WITH STREAMEXPIRY @FinalFileExp;
#ENDIF
```

---

## 7.2 Incremental Append Pattern — Growing a History File

**What is it?**
Every day, the script adds today's data to a growing history file.
When rerunning the same date, the old data for that date is removed first, then fresh data is added.

**Why do we use it?**
Power BI dashboards show historical trends. Without history append, you would only see today's data.

**How it works — simply:**
```
Day 1 run  → History = [Day1 fresh data]
Day 2 run  → History = [Day1 old] + [Day2 fresh]
Day 3 run  → History = [Day1 old] + [Day2 old] + [Day3 fresh]

Day 2 reruns (correction):
→ Read history file
→ EXCLUDE Day2 from history (WHERE NOT in date range)
→ UNION ALL with fresh Day2 data
→ Write back: History = [Day1 old] + [Fresh Day2] + [Day3 old]
```

```sql
// From Script 1 — full pattern
#IF(@AddHistory AND EXISTS(@FACT_PMaxPerformance))

    FACT_PMaxPerformance =
        // Today's fresh processed data
        SELECT * FROM FACT_PMaxPerformance

        UNION ALL

        // Previous history MINUS today's date range
        SELECT *
        FROM
        (
            EXTRACT Date : string, CustomerId : int?, MediumId : int,
                    CampaignTypeId : uint, AdDisplayTypeId : int,
                    MarketplaceClassificationId sbyte?,
                    Impressions : long, Clicks : long,
                    Revenue : decimal, Conversions : decimal,
                    ConversionEnabledClicks : long,
                    ConversionEnabledRevenue : decimal,
                    GrossProfit : decimal
            FROM @FACT_PMaxPerformance
            USING DefaultTextExtractor()
        )
        WHERE !(  Convert.ToDateTime(Date) >= Convert.ToDateTime(@START_DATE)
               AND Convert.ToDateTime(Date) <= Convert.ToDateTime(@END_DATE));
        // ↑ This excludes today's date range from history so we do not duplicate

    [Privacy.Asset.NonPersonal]
    OUTPUT FACT_PMaxPerformance
    TO @FACT_PMaxPerformance
    WITH STREAMEXPIRY @FinalFileExp;

#ENDIF
```

### The SS File Version (Script 1 — Adoption and Campaign history)

```sql
// From Script 1 — adoption history using SSTREAM
Adoption =
    SELECT *
    FROM Adoption          // today's fresh data
    UNION ALL
    SELECT *
    FROM
    (
        SSTREAM @FACT_AdoptionSS    // previous history file
    )
    WHERE !(Convert.ToDateTime(Date) >= Convert.ToDateTime(@START_DATE)
         AND Convert.ToDateTime(Date) <= Convert.ToDateTime(@END_DATE))
       AND Denominator > 0;

[Privacy.Asset.NonPersonal]
OUTPUT
TO SSTREAM @FACT_AdoptionSS
    CLUSTERED BY Date, CustomerId
    SORTED BY Date, CustomerId
    WITH STREAMEXPIRY @FinalFileExp;
```

### Daily History Files Pattern (Scripts 4, 6)

Some scripts write a new file for each day — `file_2026-02-28.ss`, `file_2026-02-27.ss` etc. They keep yesterday's file as the history source.

```sql
// From Script 4
#DECLARE PREV_DATE string = Convert.ToDateTime(@START_DATE).AddDays(-1).ToString("yyyy-MM-dd");
#DECLARE Hist string = @"path/PMaxCustomerCountV4_" + @PREV_DATE + ".ss";  // yesterday
#DECLARE Out  string = @"path/PMaxCustomerCountV4_" + @END_DATE  + ".ss";  // today

// Read yesterday, append today, write today's file
Final =
    SELECT * FROM (SSTREAM @Hist)   // yesterday's history
    UNION ALL
    SELECT * FROM PMaxCustomers;    // today's new data

[Privacy.Asset.NonPersonal]
OUTPUT TO SSTREAM @Out WITH STREAMEXPIRY "89";
```

---

## 7.3 `#CS` / `#ENDCS` — Inline C# Helper Functions

**What is it?**
Write a C# function directly inside the script. The function can then be called inside any SELECT.

**Why do we use it?**
Converting numeric IDs to readable names needs a lookup table (switch statement).
`BiddingSchemeId = 6` → `"Target ROAS"` is easiest to write in C# switch/case.

```sql
// From Script 1 — put #CS block at the VERY BOTTOM of the script

#CS

// FUNCTION 1: Convert AssetTypeId to asset category name
public static string GetAssetType(string AssetTypeId)
{
    if (AssetTypeId == null) return "";   // always handle null first

    switch (AssetTypeId)
    {
        case "36": return "CTA";
        case "34": return "Description";
        case "32": return "Headline";
        case "33": return "Long Headlines";
        case "1":
        case "5":
        case "8":
        case "9":
        case "35":
        case "2":
        case "6":
        case "7":
        case "31":
        case "13":
        case "14": return "Image";    // many IDs map to the same label
        default:   return "Others";
    }
}


// FUNCTION 2: Convert BiddingSchemeId number to readable bidding strategy name
// byte? means the value can be null — always use nullable type for optional columns
static string GetBiddingScheme(byte? BiddingSchemeId)
{
    switch (BiddingSchemeId)
    {
        case 1:  return "Manual";
        case 2:  return "Max Clicks";
        case 3:  return "Max Conversions";
        case 4:  return "Target CPA";
        case 5:  return "ECPC";
        case 6:  return "Target ROAS";
        case 7:  return "Max ROAS";
        case 8:  return "Max Conversion Value";
        case 9:  return "Target Impression Share";
        case 10: return "Use Portfolio Bid Strategy Type";
        case 11: return "Manual CPV";
        case 12: return "Manual CPM";
        case 13: return "Percent CPC";
    }
    return "UnKnown";    // default for any unlisted value
}


// FUNCTION 3: Convert LifeCycleStatusId to readable status
static string GetAssetStatus(byte? LifeCycleStatusId)
{
    switch (LifeCycleStatusId)
    {
        case 71: return "Ad Submitted into the system";
        case 75: return "Ad has been deleted from the system";
        case 92: return "Asset creation process is not complete";
        case 93: return "Submitted into the system";
        case 94: return "Asset has been deleted from the system";
    }
    return "UnKnown";
}

#ENDCS

// HOW TO CALL THEM in SELECT — just like any built-in function:
@labelled =
    SELECT CampaignId,
           GetBiddingScheme(BiddingSchemeId)  AS BiddingStrategy,
           GetAssetType(AssetTypeId)          AS AssetCategory
    FROM @data;
```

**Rules for `#CS` blocks:**
- Must be at the **bottom** of the script (after all OUTPUTs)
- Functions must be `static` (use `public static` or just `static`)
- Use nullable types: `byte?` not `byte`, `int?` not `int`
- Always handle null first: `if (AssetTypeId == null) return "";`

---

## 7.4 The Adoption / SUI Metric Pattern (Script 1)

**What is it?**
Measures: "What fraction of customers who run any campaign also run PMax?"

**SUI (Spend Under Influence)** = "What fraction of total revenue comes from accounts that have PMax?"

```sql
// STEP 1: Per account — count PMax campaigns and get revenue
Adoption =
    SELECT Date, CustomerId, AccountId,
           COUNTIF(AdvertisingChannelTypeId == 9)  AS PMaxCnt,    // how many PMax campaigns
           SUM(Revenue)                             AS Revenue
    FROM Adoption
    GROUP BY Date, CustomerId, AccountId;

// STEP 2: Per customer — adoption numerator/denominator
Adoption =
    SELECT Date,
           CustomerId,
           COUNTIF(PMaxCnt > 0)              AS Numerator,       // accounts WITH at least 1 PMax
           SUM(PMaxCnt > 0 ? Revenue : 0)   AS SUINumerator,    // revenue from PMax accounts
           COUNT()                            AS Denominator,    // ALL accounts
           SUM(Revenue)                       AS SUIDenominator  // ALL revenue
    FROM Adoption
    GROUP BY Date, CustomerId;

// Adoption Rate = Numerator / Denominator
// SUI Rate = SUINumerator / SUIDenominator
```

---

## 7.5 Multi-Period Comparison Pattern (Scripts 3, 8, 9)

**What is it?**
Comparing metrics across multiple time periods (current month vs last month vs 2 months ago, or current week vs previous week).

**Why do we use it?**
PMax performance reports show month-over-month and week-over-week trends.

### Week-over-Week Pattern (Script 8)

```sql
// 1: Calculate current week metrics
CurrentWeekCust_Rev =
    SELECT CustomerId,
           SUM(Revenue)                           AS CurrentWeekTotalRevenue,
           SUM(IF(CampaignTypeId == 9, Revenue, 0)) AS CurrentWeekPMaxRevenue
    FROM PMaxPerformance
    WHERE Date >= @Current_WEEK AND Date <= @END_DATE
    GROUP BY CustomerId;

// 2: Calculate previous week metrics
PreviousWeekCust_Rev =
    SELECT CustomerId,
           SUM(Revenue)                           AS PreviousWeekTotalRevenue,
           SUM(IF(CampaignTypeId == 9, Revenue, 0)) AS PreviousWeekPMaxRevenue
    FROM PMaxPerformance
    WHERE Date >= @START_DATE AND Date <= @PREV_WEEK
    GROUP BY CustomerId;

// 3: Zero-pad each half for UNION ALL
CurrentWeekCust_Rev =
    SELECT CustomerId, CurrentWeekTotalRevenue, CurrentWeekPMaxRevenue,
           (decimal) 0 AS PreviousWeekTotalRevenue, (decimal) 0 AS PreviousWeekPMaxRevenue
    FROM CurrentWeekCust_Rev;

PreviousWeekCust_Rev =
    SELECT CustomerId,
           (decimal) 0 AS CurrentWeekTotalRevenue, (decimal) 0 AS CurrentWeekPMaxRevenue,
           PreviousWeekTotalRevenue, PreviousWeekPMaxRevenue
    FROM PreviousWeekCust_Rev;

// 4: Merge and calculate delta
Customer_Rev =
    SELECT CustomerId,
           SUM(CurrentWeekPMaxRevenue)  AS CurrentWeekPMaxRevenue,
           SUM(PreviousWeekPMaxRevenue) AS PreviousWeekPMaxRevenue
    FROM (SELECT * FROM CurrentWeekCust_Rev UNION ALL SELECT * FROM PreviousWeekCust_Rev)
    GROUP BY CustomerId;

WoWCustomer_Rev =
    SELECT CustomerId, CurrentWeekPMaxRevenue, PreviousWeekPMaxRevenue,
           (CurrentWeekPMaxRevenue - PreviousWeekPMaxRevenue) AS WoWDeltaPMaxRevenue
    FROM Customer_Rev;
```

### 3-Month Comparison Pattern (Script 3)

```sql
// Filter the same dataset to 3 different dates
CM  = SELECT * FROM Adoption WHERE Date == @START_DATE;        // current month
PM  = SELECT * FROM Adoption WHERE Date == @LastMonthEnddate;  // previous month
P2M = SELECT * FROM Adoption WHERE Date == @Last2MonthEnddate; // 2 months ago

// Calculate metrics for each period separately
CMSegmentAdoption  = SELECT Segment, COUNT(DISTINCT CustomerId) AS Customers FROM CM  GROUP BY Segment;
PMSegmentAdoption  = SELECT Segment, COUNT(DISTINCT CustomerId) AS Customers FROM PM  GROUP BY Segment;
P2MSegmentAdoption = SELECT Segment, COUNT(DISTINCT CustomerId) AS Customers FROM P2M GROUP BY Segment;

// Join all three periods into one final row per segment
FinalAdoption =
    SELECT A.Segment, A.Customers, B.Customers AS PMCustomers, C.Customers AS P2MCustomers
    FROM CMSegmentAdoption  AS A
    INNER JOIN PMSegmentAdoption  AS B ON A.Segment == B.Segment
    INNER JOIN P2MSegmentAdoption AS C ON A.Segment == C.Segment;
```

---

---

# CHAPTER 8 — Working Fast in Production

---

## 8.1 How to Read Any Script in 5 Minutes

**Step 1 — Header:**
What does this script do? What does it read? What does it output?

**Step 2 — All `#DECLARE` at the top:**
What dates? What flags? What file paths are input and output?

**Step 3 — Feature flags and `#IF` blocks:**
Which sections are active today?

**Step 4 — Find data sources:**
Every `SSTREAM`, `VIEW`, `EXTRACT` — these are where data enters.

**Step 5 — Follow the variable:**
`Campaigns =` ... `Campaigns =` ... `Campaigns =` — each assignment refines it.

**Step 6 — Find all `OUTPUT`:**
These are what the script produces.

---

## 8.2 How to Modify a Script Safely

When making changes, follow this order:

**1. Change the date first — test with one day:**
```sql
#DECLARE START_DATE string = "2026-02-28";
#DECLARE Days       int    = 1;
```

**2. Comment out OUTPUTs to test without writing files:**
```sql
//[Privacy.Asset.NonPersonal]
//OUTPUT @MyData TO @OutputPath WITH STREAMEXPIRY "30";
```

**3. Use `#ERROR` to print a calculated value:**
```sql
//#ERROR @END_DATE;    // Uncomment to see the value — it crashes the script and shows the date
```

**4. Take a sample to test logic fast:**
```sql
@sample = SELECT TOP 1000 * FROM @HugeTable;
```

**5. Check your row counts before adding history:**
Add a temporary count output to verify the data before appending to history.

---

## 8.3 Common Errors and How to Fix Them

| Error | Cause | Fix |
|---|---|---|
| Type mismatch on JOIN | `CampaignId` is `int` in one table, `long` in another | Add `Convert.ToInt64(CampaignId)` before the JOIN |
| Null crash on column | Column has null, you did math on it | Use `?? 0` default or declare as `int?` nullable |
| Divide by zero | Calculating rates when denominator is 0 | `(denom == 0 ? 0.0 : num / denom)` |
| Join too slow or hangs | One key (RGUID) has millions of rows | Add `[SKEWJOIN=(SKEW=FROMLEFT, REPARTITION=FULLJOIN)]` |
| File not found crash | Daily file missing for a date | Use `SPARSE` in `SSTREAM SPARSE STREAMSET` |
| Privacy annotation error | OUTPUT has no `[Privacy.Asset.*]` | Add `[Privacy.Asset.NonPersonal]` above OUTPUT |
| Duplicate rows in output | Multiple JOINs multiplied rows | Add `FIRST()` + `GROUP BY` at the end |
| `#ERROR` guard fires | Both feature flags are false | Check your `#DECLARE` flag values |
| History file crash on first run | Script reads a file that does not exist | Wrap in `#IF(@AddHistory AND EXISTS(@HistoryFile))` |
| UNION ALL column type mismatch | int in one branch, long in another | Cast with `(long) 0` to match types |

---

## 8.4 Script Structure — Always Follow This Order

```
────────────────────────────────────────
TOP SECTION
────────────────────────────────────────
1.  Header comment block
2.  MODULE import + USING Privacy
3.  #DECLARE runtime params (@@RUN_DATE@@, @@Days@@)
4.  #DECLARE boolean flags (Demand, Performance, DimTable, AddHistory)
5.  #IF guard + #ERROR (fail fast if invalid combination)
6.  #DECLARE computed dates (END_DATE, PREV_DATE, StartDateTime, EndDateTime)
7.  #DECLARE project paths (ProjectBasePath, file folders, expiry settings)
8.  #DECLARE all input paths
9.  #DECLARE all output paths

────────────────────────────────────────
DATA SECTIONS
────────────────────────────────────────
10. #IF(@Demand)
        SSTREAM SPARSE STREAMSET reads
        SELECT to clean and filter
        INNER JOIN to build full campaign list
        LEFT OUTER JOIN for revenue
        COUNTIF for adoption calculation
        Incremental append pattern
        OUTPUT demand results
    #ENDIF

11. #IF(@Performance)
        VIEW read → Impressions (FraudQualityBand >= 2)
        VIEW read → Clicks (FraudQualityBand >= 2)
        VIEW read → Conversions (FraudQualityBand >= 2)
        Conversion dedup (cap at 1 per RGUID+ClickId)
        SKEWJOIN clicks to conversions
        UNION ALL impressions + clicks with (long)0 zeros
        GROUP BY to get final fact table
        Normalise IDs with IF()
        Incremental history append with EXISTS()
        OUTPUT performance results
    #ENDIF

12. #IF(@DimTable)
        VIEW/EXTRACT each mapping file (segment, vertical, country, AE, AM)
        FIRST() deduplicate each mapping
        Multi-LEFT JOIN all onto customer list
        FIRST() + GROUP BY final deduplicate
        OUTPUT dimension table
    #ENDIF

────────────────────────────────────────
BOTTOM
────────────────────────────────────────
13. #CS
        C# helper functions (GetBiddingScheme, GetAssetType, GetAssetStatus)
    #ENDCS
```

---

## 8.5 Complete Quick Reference

### Scope-Only Functions

| Function | Syntax | Use |
|---|---|---|
| `COUNTIF(cond)` | `COUNTIF(ChannelType == 9)` | Count rows matching condition |
| `COUNT()` | `COUNT()` | Count all rows — no argument |
| `FIRST(col)` | `FIRST(CustomerName)` | First value in group |
| `IF(c,a,b)` | `IF(IsPMax, 9, -1)` | Conditional value |
| `EXISTS(path)` | `#IF(EXISTS(@file))` | Check file exists |
| `col ?? default` | `Name ?? "Unknown"` | Replace null |
| `cond ? a : b` | `IsPMax ? Revenue : 0` | C# ternary |
| `String.IsNullOrEmpty(s)` | `String.IsNullOrEmpty(Segment)` | Check empty string |

### Data Sources

| Source | When |
|---|---|
| `EXTRACT ... USING DefaultTextExtractor()` | Plain text / mapping files |
| `EXTRACT ... USING DefaultTextExtractor(skipFirstNRows:1)` | Text file with header row |
| `SSTREAM "path.ss"` | Single binary structured stream |
| `SSTREAM SPARSE STREAMSET @base PATTERN @pat RANGE __date = [x,y]` | Multiple daily files |
| `VIEW "path.view" PARAMS(...)` | Managed logical data source |

### Outputs

| Output | When |
|---|---|
| `TO "path.txt" USING DefaultTextOutputter(outputHeader:true)` | Power BI, text reports |
| `TO SSTREAM "path.ss" CLUSTERED BY ... SORTED BY ...` | Binary for next script or history |

### Privacy

```sql
// Required above every OUTPUT
[Privacy.Asset.NonPersonal]

// Required before SELECT with low-cardinality columns
[LOWDISTINCTNESS(CampaignTypeId, MediumId, ...)]
```

---

*This guide covers every concept from all 8 real PMax production scripts:*
*`#DECLARE` · `@@params@@` · Date math · `string.Format` · `#IF/#ENDIF/#ERROR` · `SSTREAM SPARSE STREAMSET` · `VIEW PARAMS` · `EXTRACT` with all types including `uint sbyte? byte?` · `skipFirstNRows` · `SELECT DISTINCT` · `SELECT A.*` · `GROUP BY` · `SUM/COUNT/COUNTIF/FIRST/COUNT(DISTINCT)` · `IF()` nested · `??` · `Convert.*` · `(long)0 casting` · `String.IsNullOrEmpty` · `WHERE FraudQualityBand >= 2` · `IN/NOT IN` · `UNION ALL` zero-merge pattern · `UNION` dedup · `INNER JOIN` · `LEFT OUTER JOIN` multi-table · `SKEWJOIN` · Click-Conv join · `[Privacy.Asset.NonPersonal]` · `[LOWDISTINCTNESS]` · `EXISTS()` · Incremental append (text + ss + daily file variants) · `#CS/#ENDCS` · `GetBiddingScheme/GetAssetType/GetAssetStatus` · Adoption/SUI metric · WoW/MoM multi-period comparison · `CLUSTERED BY/SORTED BY` · `STREAMEXPIRY` · Multi-segment UNION ALL stack*
