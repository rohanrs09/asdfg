# Scope Script — Beginner Practice Examples
### 5 Simple Examples | Practice in VS Studio | Covers All Key Concepts

---

> **How to practice:**
> Open VS Studio → New Scope Script file → Copy each example → Run it → Read the output → Understand what happened.
> Each example builds on the previous one.

---

---

## Example 1 — Variables, Dates, and Flags
### Concepts: `#DECLARE` · `@@PARAM@@` · `#IF` · `#ERROR` · `#ENDIF`

---

**What this example does:**
Declares variables for a date range and flags, then prints a message showing the calculated dates.
This is the very first thing every real script does.

```sql
//==========================================================
// EXAMPLE 1: Variables and Flags
// Practice: #DECLARE, #IF, #ENDIF, #ERROR
//==========================================================

// --- Step 1: Declare your run date
// In production this would be @@RUN_DATE@@ from the scheduler
// For practice, hardcode a date
#DECLARE START_DATE  string  =  "2024-03-01";
#DECLARE Days        int     =  7;

// --- Step 2: Declare feature flags
#DECLARE RunSection1  bool  =  true;
#DECLARE RunSection2  bool  =  false;

// --- Step 3: Guard — if BOTH sections are off, crash with a message
#IF(!@RunSection1 AND !@RunSection2)
    #ERROR "At least one section must be true. Please check your flags.";
#ENDIF

// --- Step 4: Calculate end date from start date + days
#DECLARE END_DATE  string  =
    DateTime.Parse(@START_DATE).AddDays(@Days - 1).ToString("yyyy-MM-dd");

// --- Step 5: Build DateTime objects
#DECLARE StartDateTime  DateTime  =  DateTime.Parse(@START_DATE + " 00:00");
#DECLARE EndDateTime    DateTime  =  DateTime.Parse(@END_DATE   + " 23:59");

// --- OPTIONAL: Uncomment next line to see what END_DATE calculated to
// #ERROR @END_DATE;

// --- Step 6: Use #IF to run different sections based on flags

#IF(@RunSection1)

    // This block runs because RunSection1 = true
    @Section1Result =
        SELECT "Section1 is ON"      AS Status,
               @START_DATE           AS StartDate,
               @END_DATE             AS EndDate,
               @Days                 AS TotalDays;

    OUTPUT @Section1Result
    TO "/output/practice/example1_section1.txt"
    USING DefaultTextOutputter(outputHeader: true);

#ENDIF

#IF(@RunSection2)

    // This block is SKIPPED because RunSection2 = false
    @Section2Result =
        SELECT "Section2 is ON" AS Status;

    OUTPUT @Section2Result
    TO "/output/practice/example1_section2.txt"
    USING DefaultTextOutputter(outputHeader: true);

#ENDIF
```

### What to notice when you run this:
- Only `example1_section1.txt` is created — Section 2 is skipped
- `END_DATE` = 2024-03-07 (March 1 + 6 days)
- Try setting `RunSection1 = false` — the `#ERROR` guard fires immediately
- Try uncommenting `#ERROR @END_DATE` to see the calculated date printed as an error message

---

---

## Example 2 — Read a File and Select Columns
### Concepts: `EXTRACT` · `SELECT` · `WHERE` · `string types` · `nullable ?` · `OUTPUT`

---

**What this example does:**
Creates a small fake campaign dataset, reads it, filters it, and outputs a clean version.
This is the basic read → filter → write pattern every script uses.

**First — create this input file manually and save it as `/practice/campaigns.txt`:**
```
CampaignId,CampaignName,ChannelType,AdSpend,Clicks,IsActive
1001,Brand Search Q1,1,12000.50,8500,true
1002,Shopping Feed,3,8500.00,4200,true
1003,Display Retargeting,6,3200.00,0,false
1004,PMax Campaign,9,15000.00,9800,true
1005,Non-Brand Search,1,0,0,true
```

```sql
//==========================================================
// EXAMPLE 2: EXTRACT, SELECT, WHERE, OUTPUT
// Practice: Read a file, filter rows, output result
//==========================================================

#DECLARE START_DATE  string  =  "2024-03-01";
#DECLARE InputFile   string  =  "/practice/campaigns.txt";
#DECLARE OutputFile  string  =  "/output/practice/example2_active.txt";

// --- Step 1: EXTRACT reads the file
// You must define every column name and its type
// Use ? for nullable columns that might be empty
@RawCampaigns =
    EXTRACT CampaignId    : long,
            CampaignName  : string,
            ChannelType   : int,
            AdSpend       : double,
            Clicks        : long,
            IsActive      : bool
    FROM @InputFile
    USING DefaultTextExtractor(silent: true);   // silent: true = skip bad rows

// --- Step 2: SELECT only the columns you need
// Rename columns, calculate a new column
@CleanCampaigns =
    SELECT CampaignId,
           CampaignName,
           ChannelType,
           AdSpend,
           Clicks,
           IsActive,
           // Calculate CPC — but handle divide-by-zero safely
           (Clicks == 0 ? 0.0 : AdSpend / Clicks)  AS CPC
    FROM @RawCampaigns;

// --- Step 3: WHERE — filter to only active campaigns with real spend
@ActiveCampaigns =
    SELECT *
    FROM @CleanCampaigns
    WHERE IsActive == true
      AND AdSpend > 0
      AND Clicks  > 0;

// --- Step 4: OUTPUT the result
OUTPUT @ActiveCampaigns
TO @OutputFile
USING DefaultTextOutputter(outputHeader: true);
```

### What to notice when you run this:
- Campaign 1003 (IsActive=false) is removed by `WHERE`
- Campaign 1005 (AdSpend=0, Clicks=0) is removed by `WHERE`
- CPC is calculated for all remaining campaigns
- Try removing `AND Clicks > 0` — Campaign 1005 will cause divide-by-zero without the ternary `? :` guard
- The `?` on nullable types prevents crashes on empty values

---

---

## Example 3 — GROUP BY, Aggregation, and Labelling
### Concepts: `GROUP BY` · `SUM` · `COUNT` · `COUNTIF` · `IF()` · `??` · `ORDER BY`

---

**What this example does:**
Takes campaign data and rolls it up by channel type.
Then labels each channel's performance. This is the aggregation + labelling pattern used in every report.

```sql
//==========================================================
// EXAMPLE 3: GROUP BY, Aggregation, COUNTIF, IF(), Labels
// Practice: Summarise data and add performance labels
//==========================================================

#DECLARE InputFile   string  =  "/practice/campaigns.txt";
#DECLARE OutputFile  string  =  "/output/practice/example3_summary.txt";

// --- Step 1: Read the file
@RawCampaigns =
    EXTRACT CampaignId    : long,
            CampaignName  : string,
            ChannelType   : int,
            AdSpend       : double,
            Clicks        : long,
            IsActive      : bool
    FROM @InputFile
    USING DefaultTextExtractor(silent: true);

// --- Step 2: Calculate KPIs per row first
@WithKPIs =
    SELECT CampaignId,
           CampaignName,
           ChannelType,
           AdSpend,
           Clicks,
           IsActive,
           (Clicks == 0 ? 0.0 : AdSpend / Clicks)  AS CPC
    FROM @RawCampaigns
    WHERE IsActive == true AND AdSpend > 0;

// --- Step 3: GROUP BY ChannelType — aggregate all campaigns per channel
@ChannelSummary =
    SELECT
        ChannelType,

        // COUNT() — how many campaigns in this channel
        COUNT()                                      AS TotalCampaigns,

        // COUNTIF() — count only campaigns with good CPC (under 2.0)
        COUNTIF(CPC < 2.0)                           AS EfficientCampaigns,

        // Standard aggregates
        SUM(AdSpend)                                 AS TotalSpend,
        SUM(Clicks)                                  AS TotalClicks,

        // Blended CPC across all campaigns in this channel
        (SUM(Clicks) == 0 ? 0.0 : SUM(AdSpend) / SUM(Clicks))  AS BlendedCPC

    FROM @WithKPIs
    GROUP BY ChannelType;

// --- Step 4: Add labels using IF()
@Labelled =
    SELECT
        ChannelType,

        // IF() to convert the number into a readable channel name
        IF(ChannelType == 1,  "Search",
        IF(ChannelType == 3,  "Shopping",
        IF(ChannelType == 6,  "Display",
        IF(ChannelType == 9,  "PMax",
                              "Unknown"))))          AS ChannelName,

        TotalCampaigns,
        EfficientCampaigns,
        TotalSpend,
        TotalClicks,
        BlendedCPC,

        // IF() to add a performance status label based on BlendedCPC
        IF(BlendedCPC > 0 AND BlendedCPC < 1.5,  "Efficient",
        IF(BlendedCPC >= 1.5 AND BlendedCPC < 3.0, "Average",
                                                    "Expensive"))  AS SpendEfficiency

    FROM @ChannelSummary;

// --- Step 5: Sort by TotalSpend highest first
@Sorted =
    SELECT *
    FROM @Labelled
    ORDER BY TotalSpend DESC;

// --- Step 6: OUTPUT
OUTPUT @Sorted
TO @OutputFile
USING DefaultTextOutputter(outputHeader: true);
```

### What to notice when you run this:
- All campaigns in the same `ChannelType` are collapsed into one row
- `COUNTIF(CPC < 2.0)` counts only the efficient campaigns per channel
- Channel numbers become readable names via nested `IF()`
- `BlendedCPC` uses the divide-by-zero guard — try removing it and see what happens
- Change `ORDER BY TotalSpend DESC` to `ASC` and observe the sort flip

---

---

## Example 4 — Joins and Null Handling
### Concepts: `INNER JOIN` · `LEFT OUTER JOIN` · `??` · `FIRST()` · multiple `EXTRACT`

---

**What this example does:**
Joins a campaign performance file with a customer mapping file.
Some campaigns will not have a customer mapping — you handle those nulls with `??`.
This is exactly the dimension-join pattern used in production.

**Create this second file as `/practice/customers.txt`:**
```
CustomerId,CustomerName,Segment,Country
201,Contoso Electronics,Enterprise,USA
202,Fabrikam Retail,SMB,UK
203,Northwind Traders,Enterprise,Australia
```

**Create this third file as `/practice/campaign_customer.txt`:**
```
CampaignId,CustomerId,Revenue
1001,201,65000.00
1002,202,28000.00
1003,201,8000.00
1004,999,45000.00
1005,203,12000.00
```
*(Note: CustomerId 999 does not exist in the customers file — intentional, to practice null handling)*

```sql
//==========================================================
// EXAMPLE 4: INNER JOIN, LEFT OUTER JOIN, ??, FIRST()
// Practice: Join tables, handle missing matches
//==========================================================

#DECLARE CampaignFile string = "/practice/campaign_customer.txt";
#DECLARE CustomerFile  string = "/practice/customers.txt";
#DECLARE OutputInner   string = "/output/practice/example4_inner.txt";
#DECLARE OutputLeft    string = "/output/practice/example4_left.txt";
#DECLARE OutputDim     string = "/output/practice/example4_dim.txt";

// --- Step 1: Read campaign-customer link file
@CampaignData =
    EXTRACT CampaignId  : long,
            CustomerId  : int,
            Revenue     : double
    FROM @CampaignFile
    USING DefaultTextExtractor(silent: true);

// --- Step 2: Read customer mapping file
@CustomerData =
    EXTRACT CustomerId    : int,
            CustomerName  : string,
            Segment       : string,
            Country       : string
    FROM @CustomerFile
    USING DefaultTextExtractor(silent: true);

// ===========================================================
// JOIN TYPE 1 — INNER JOIN
// Only keeps campaigns where CustomerId exists in BOTH tables
// Campaign 1004 (CustomerId 999) will be DROPPED
// ===========================================================
@InnerResult =
    SELECT A.CampaignId,
           A.CustomerId,
           A.Revenue,
           B.CustomerName,
           B.Segment,
           B.Country
    FROM @CampaignData AS A
    INNER JOIN @CustomerData AS B
    ON A.CustomerId == B.CustomerId;

OUTPUT @InnerResult
TO @OutputInner
USING DefaultTextOutputter(outputHeader: true);

// ===========================================================
// JOIN TYPE 2 — LEFT OUTER JOIN
// Keeps ALL campaigns, even Campaign 1004 (CustomerId 999)
// For Campaign 1004, customer columns will be NULL
// ?? fills those nulls with a default value
// ===========================================================
@LeftResult =
    SELECT A.CampaignId,
           A.CustomerId,
           A.Revenue,
           B.CustomerName  ?? "Unknown Customer"  AS CustomerName,
           B.Segment       ?? "Unclassified"      AS Segment,
           B.Country       ?? "Unknown"           AS Country
    FROM @CampaignData AS A
    LEFT OUTER JOIN @CustomerData AS B
    ON A.CustomerId == B.CustomerId;

OUTPUT @LeftResult
TO @OutputLeft
USING DefaultTextOutputter(outputHeader: true);

// ===========================================================
// DIMENSION TABLE BUILD
// One row per customer with FIRST() to collapse duplicates
// ===========================================================
@DimCustomer =
    SELECT CustomerId,
           FIRST(CustomerName) AS CustomerName,
           FIRST(Segment)      AS Segment,
           FIRST(Country)      AS Country
    FROM @CustomerData
    GROUP BY CustomerId;

OUTPUT @DimCustomer
TO @OutputDim
USING DefaultTextOutputter(outputHeader: true);
```

### What to notice when you run this:
- `example4_inner.txt` has **4 rows** — Campaign 1004 (CustomerId 999) is missing
- `example4_left.txt` has **5 rows** — Campaign 1004 is there with "Unknown Customer"
- Remove the `??` from LEFT JOIN and rerun — Campaign 1004 will show blanks/nulls
- `FIRST()` in the dimension table ensures one clean row per customer

---

---

## Example 5 — UNION ALL, Type Casting, and the Merge Pattern
### Concepts: `UNION ALL` · `(long)0` casting · `SUM` to merge · `Convert.*` · re-assignment

---

**What this example does:**
Simulates the real Impression + Click merge pattern.
You have two separate files — one for impressions, one for clicks.
You merge them into one fact table using `UNION ALL + GROUP BY`.
This is the most important pattern in Microsoft Ads performance scripts.

**Create `/practice/impressions.txt`:**
```
CampaignId,Date,Impressions,Revenue
1001,2024-03-01,50000,1200.00
1002,2024-03-01,30000,800.00
1003,2024-03-01,20000,400.00
```

**Create `/practice/clicks.txt`:**
```
CampaignId,Date,Clicks,Revenue,Conversions
1001,2024-03-01,1800,1200.00,45
1002,2024-03-01,900,800.00,22
1003,2024-03-01,200,400.00,5
```

```sql
//==========================================================
// EXAMPLE 5: UNION ALL, Type Casting, Merge Pattern
// Practice: Merge two separate files into one fact table
//==========================================================

#DECLARE ImpFile    string = "/practice/impressions.txt";
#DECLARE ClkFile    string = "/practice/clicks.txt";
#DECLARE OutputFile string = "/output/practice/example5_fact.txt";

// --- Step 1: Read impressions file
@RawImpressions =
    EXTRACT CampaignId   : long,
            Date         : string,
            Impressions  : long,
            Revenue      : double
    FROM @ImpFile
    USING DefaultTextExtractor(silent: true);

// --- Step 2: Read clicks file
@RawClicks =
    EXTRACT CampaignId   : long,
            Date         : string,
            Clicks       : long,
            Revenue      : double,
            Conversions  : long
    FROM @ClkFile
    USING DefaultTextExtractor(silent: true);

// --- Step 3: Give impressions zero-value click columns
// (long)0 = zero as a long type — must match the type of Clicks
// (long)0 for Conversions too — same reason
@ImpRows =
    SELECT CampaignId,
           Date,
           Impressions,
           (long)0  AS Clicks,        // no clicks in this file — put zero
           Revenue,
           (long)0  AS Conversions    // no conversions in this file — put zero
    FROM @RawImpressions;

// --- Step 4: Give clicks zero-value impression columns
@ClkRows =
    SELECT CampaignId,
           Date,
           (long)0  AS Impressions,   // no impressions in this file — put zero
           Clicks,
           Revenue,
           Conversions
    FROM @RawClicks;

// --- Step 5: UNION ALL — stack both tables on top of each other
// Now we have 6 rows total (3 impression rows + 3 click rows)
@Combined =
    SELECT * FROM @ImpRows
    UNION ALL
    SELECT * FROM @ClkRows;

// --- Step 6: GROUP BY + SUM — collapse to one row per CampaignId+Date
// The zeros cancel out — real values add up
@FactTable =
    SELECT CampaignId,
           Date,
           SUM(Impressions)  AS Impressions,
           SUM(Clicks)       AS Clicks,
           SUM(Revenue)      AS Revenue,
           SUM(Conversions)  AS Conversions
    FROM @Combined
    GROUP BY CampaignId, Date;

// --- Step 7: Re-assign to add calculated KPIs (re-assignment pattern)
// The same variable name is used again — this is normal in Scope
@FactTable =
    SELECT CampaignId,
           Date,
           Impressions,
           Clicks,
           Revenue,
           Conversions,

           // CTR — clicks divided by impressions × 100
           (Impressions == 0 ? 0.0 : (double)Clicks / Impressions * 100)  AS CTR_Pct,

           // CPC — spend divided by clicks
           (Clicks == 0 ? 0.0 : Revenue / Clicks)                          AS CPC,

           // Conversion Rate — conversions divided by clicks × 100
           (Clicks == 0 ? 0.0 : (double)Conversions / Clicks * 100)        AS ConvRate_Pct

    FROM @FactTable;

// --- Step 8: OUTPUT the final merged fact table
OUTPUT @FactTable
TO @OutputFile
USING DefaultTextOutputter(outputHeader: true);
```

### What to notice when you run this:
- The output has **3 rows** — one per campaign, not 6
- Each row has **both** impressions and clicks — merged from two separate files
- Campaign 1001: Impressions=50000, Clicks=1800, CTR=3.6%
- The `(double)` cast before `Clicks / Impressions` is important — without it, integer division gives 0
- `@FactTable =` is used twice — second time adds KPI columns. This is the re-assignment pattern
- Try removing `(Impressions == 0 ? 0.0 :` and see the divide-by-zero crash

---

---

## Example 6 — `#CS` Functions, `SSTREAM`, `Privacy Annotation`
### Concepts: `#CS / #ENDCS` · C# switch functions · `SSTREAM` · `[Privacy.Asset.NonPersonal]` · `STREAMEXPIRY`

---

**What this example does:**
Adds C# helper functions to convert numeric IDs to readable labels.
Reads an `.ss` binary file. Writes output with proper privacy annotation.
This combines everything into a near-production pattern.

```sql
//==========================================================
// EXAMPLE 6: #CS Functions, SSTREAM, Privacy Annotation
// Practice: C# helpers, binary file read, compliance output
//==========================================================

#DECLARE START_DATE  string  =  "2024-03-01";
#DECLARE Days        int     =  7;
#DECLARE END_DATE    string  =  DateTime.Parse(@START_DATE).AddDays(@Days - 1).ToString("yyyy-MM-dd");

// Paths
#DECLARE InputSS    string = "/practice/campaign_data.ss";         // binary input
#DECLARE OutputTxt  string = "/output/practice/example6_final.txt"; // text output for Power BI
#DECLARE OutputSS   string = "/output/practice/example6_data.ss";   // binary output for next script
#DECLARE Expiry     string = "30";

// --- Step 1: Read the .ss binary file (no schema needed)
@RawData = SSTREAM @InputSS;

// --- Step 2: Use C# functions to label the IDs
// GetChannelName and GetBiddingStrategy are defined in #CS at the bottom
@Labelled =
    SELECT CampaignId,
           CustomerId,
           Date,
           ChannelTypeId,
           BiddingSchemeId,
           Impressions,
           Clicks,
           Revenue,
           Conversions,

           // Call C# function — converts number to readable name
           GetChannelName(ChannelTypeId)        AS ChannelName,
           GetBiddingStrategy(BiddingSchemeId)  AS BiddingStrategy,

           // Calculated KPIs
           (Clicks == 0 ? 0.0 : Revenue / Clicks)           AS CPC,
           (Impressions == 0 ? 0.0 :
               (double)Clicks / Impressions * 100)           AS CTR_Pct,
           (Clicks == 0 ? 0.0 :
               (double)Conversions / Clicks * 100)           AS ConvRate_Pct,

           // Performance label
           IF(Revenue / (Clicks == 0 ? 1 : Clicks) < 1.5,   "Efficient",
           IF(Revenue / (Clicks == 0 ? 1 : Clicks) < 3.0,   "Average",
                                                              "Expensive")) AS Efficiency

    FROM @RawData
    WHERE Impressions > 0;

// --- Step 3: OUTPUT 1 — Text file for Power BI
// Privacy annotation is REQUIRED above every OUTPUT
[Privacy.Asset.NonPersonal]
OUTPUT @Labelled
TO @OutputTxt
WITH STREAMEXPIRY @Expiry
USING DefaultTextOutputter(outputHeader: true);

// --- Step 4: OUTPUT 2 — SSTREAM binary for another script to read fast
[Privacy.Asset.NonPersonal]
OUTPUT @Labelled
TO SSTREAM @OutputSS
    CLUSTERED BY CustomerId, CampaignId
    SORTED BY CustomerId, CampaignId
    WITH STREAMEXPIRY @Expiry;


//==========================================================
// #CS BLOCK — Always put at the BOTTOM of the script
// Write C# helper functions here
//==========================================================
#CS

// Convert ChannelTypeId number to readable name
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

// Convert BiddingSchemeId to readable bidding strategy name
// byte? means the value can be null — always handle nullable types
static string GetBiddingStrategy(byte? schemeId)
{
    switch (schemeId)
    {
        case 1:  return "Manual CPC";
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

#ENDCS
```

### What to notice when you run this:
- `ChannelTypeId = 9` becomes `"PMax"` — readable in the output
- `BiddingSchemeId = 6` becomes `"Target ROAS"` — readable in the output
- Two output files are created — one `.txt` for Power BI, one `.ss` for another script
- `[Privacy.Asset.NonPersonal]` sits directly above each `OUTPUT` — required
- `CLUSTERED BY / SORTED BY` organises the `.ss` file for fast reads
- `WITH STREAMEXPIRY "30"` auto-deletes both files after 30 days
- `#CS` is at the very bottom — C# functions are called in the SELECT above

---

---

## What Each Example Teaches

| Example | Key Concepts Practised |
|---|---|
| **1** | `#DECLARE`, `@@PARAM@@`, `#IF`, `#ENDIF`, `#ERROR`, date calculation |
| **2** | `EXTRACT`, `SELECT`, `WHERE`, nullable types `?`, `OUTPUT`, divide-by-zero guard |
| **3** | `GROUP BY`, `SUM`, `COUNT`, `COUNTIF`, `IF()` nested, `ORDER BY` |
| **4** | `INNER JOIN`, `LEFT OUTER JOIN`, `??` null handling, `FIRST()`, dimension table |
| **5** | `UNION ALL`, `(long)0` casting, merge pattern, re-assignment, `(double)` cast |
| **6** | `#CS / #ENDCS`, C# switch functions, `SSTREAM`, `[Privacy.Asset.NonPersonal]`, `STREAMEXPIRY`, `CLUSTERED BY` |

---

## Practice Tip — Do These in Order

After each example, try these changes to deepen understanding:

- **Example 1:** Set both flags to `false` — watch `#ERROR` fire
- **Example 2:** Remove `AND Clicks > 0` — see what happens to CPC
- **Example 3:** Change `COUNTIF(CPC < 2.0)` to `COUNTIF(CPC < 1.0)` — fewer campaigns qualify
- **Example 4:** Remove `??` defaults — see nulls appear in output
- **Example 5:** Remove `(double)` cast before division — CTR becomes 0 everywhere
- **Example 6:** Add a new case to `GetChannelName` for a new channel ID

---

*Covers: `#DECLARE` · `#IF/#ENDIF` · `#ERROR` · `EXTRACT` · `SELECT` · `WHERE` · `GROUP BY` · `SUM / COUNT / COUNTIF` · `FIRST()` · `IF()` · `??` · `Convert.*` · `(long)0 cast` · `INNER JOIN` · `LEFT OUTER JOIN` · `UNION ALL` · `SSTREAM` · `#CS/#ENDCS` · `[Privacy.Asset.NonPersonal]` · `STREAMEXPIRY` · `CLUSTERED BY / SORTED BY`*
