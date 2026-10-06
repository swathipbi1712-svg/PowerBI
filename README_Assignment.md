# Agency Performance Dashboard -- Power BI Assessment

## What I Built

A four-page Power BI report for a life insurance agency --- executive
summary, agent performance grid, agent profile card, and a persistency
cohort matrix. The data spans 2023--2025 across 200 agents, \~16K sales
transactions, and \~9.5K policy renewal records.

------------------------------------------------------------------------

## Data Model -- Decisions and Why

### The basics

Pretty standard star schema. `Fact Sales` sits in the middle with
`Dim Date`, `Dim Agent`, and `Dim Product` around it. `Fact Persistency`
and `Fact Target` are secondary facts that also hang off `Dim Agent`.

### The Fact Target problem -- and why I used TREATAS

`Fact Target` was the awkward one. It has no `date_key` column --- just
plain `year` and `month` integers. That means I can't create a real
relationship to `Dim Date`, which rules out `USERELATIONSHIP` entirely
(you can only activate a relationship that actually exists in the
model).

My options were:

-   Add a calculated `year_month_key` column to `Fact Target` just to
    force a join --- feels like a hack.
-   Write a bridge table --- overkill for two integer columns.
-   Use `TREATAS` --- clean, purposeful, keeps the model schema honest.

I went with `TREATAS`. It takes the year/month values currently in
filter context from `Dim Date` and applies them as a virtual filter on
`Fact Target` at query time --- no physical relationship needed:

``` dax
CALCULATE(
    SUM('Fact Target'[ape_target_usd]),
    TREATAS(
        SUMMARIZE('Dim Date', 'Dim Date'[year], 'Dim Date'[month]),
        'Fact Target'[year], 'Fact Target'[month]
    )
)
```

### The Fact Persistency date choice -- this one needed a real decision

`Fact Persistency` has two date columns: `issue_date_key` (when the
policy was sold) and `renewal_due_date_key` (when it's up for renewal).
One had to be the active relationship, one inactive.

**I made `issue_date_key` active. Here's why.**

The alternative --- making `renewal_due_date_key` active --- gives you
an operational view: "of all policies coming up for renewal this month,
how many survived?" That's useful for a renewals team managing a
pipeline.

But this report is built for agency leadership who want to understand
retention by cohort --- how policies from a given sales year are holding
up over time. That means grouping by issue year makes more sense. It
also keeps Page 4 (the cohort matrix) consistent: the row axis is issue
year, the column axis is years since issue, and the date slicer controls
which issue cohorts appear. If I'd flipped the active relationship, the
same slicer would filter by renewal date and the cohort logic would
break.

So in plain terms: **"Persistency Rate for 2024" means "of all policies
we sold in 2024, what percentage renewed?"** --- not "of all policies
that came up for renewal in 2024."

### What counts as an "active" policy?

A policy is active if it's still in force --- either successfully
renewed (`renewed`) or inside its grace period (`grace`). Lapsed
policies are out. This is a snapshot count, not something you'd
meaningfully sum across time periods.

The data has no `active` status value --- the three actual values are
`renewed`, `lapsed`, and `grace`. Using `IN {"renewed", "grace"}` was an
intentional decision, not a fallback.

### Those orphan agent IDs

11 agent IDs in `Fact Sales` have no matching row in `Dim Agent`. The
sales data is real --- I didn't drop the rows --- but I had to handle
them carefully in two places:

**Agent Rank by APE:** Without a fix, all 11 orphan IDs pool into a
single blank row. Their combined APE of \~\$2M claims rank #1 ahead of
every real agent. The measure excludes blank agent names from the
ranking pool entirely.

**Top 10 bar chart and agent grid:** Both visuals have a visual-level
filter on `agent_name IS NOT BLANK`, so the orphan bucket doesn't
silently appear at the top of ranked charts.

Total APE figures on Page 1 still include the orphan sales --- they're
real revenue --- but they're excluded from any per-agent breakdown or
ranking.

------------------------------------------------------------------------

## DAX -- The Measures That Needed Careful Thought

### APE vs Target % -- the hardest one

The challenge is that `Fact Target` has no date key, so you can't filter
it through a normal relationship:

``` dax
APE vs Target % =
VAR ActualAPE = SUM('Fact Sales'[ape_usd])
VAR TargetAPE =
    CALCULATE(
        SUM('Fact Target'[ape_target_usd]),
        TREATAS(
            SUMMARIZE('Dim Date', 'Dim Date'[year], 'Dim Date'[month]),
            'Fact Target'[year], 'Fact Target'[month]
        )
    )
RETURN
    DIVIDE(ActualAPE, TargetAPE)
```

`SUMMARIZE` captures which years and months are currently in filter
context. `TREATAS` maps those as a virtual filter onto `Fact Target`'s
integer columns. Works correctly whether the user slices by year,
quarter, or a single month.

### APE Rolling 3 Months -- two bugs fixed

The initial version just used `DATESINPERIOD` inside `CALCULATE`. It
looked right but returned single-month APE every time. The reason:
`DATESINPERIOD` overrides the filter on the `date` column, but Power
BI's filter context also carries `year` and `month` as separate integer
columns on `Dim Date`. Those integer filters survived and choked the
3-month window back down to the current month.

**Fix 1 --- capture the anchor date before `CALCULATE` modifies context,
then use `ALL('Dim Date')` to wipe the integer filters so
`DATESINPERIOD` can actually expand:**

``` dax
VAR _LastDate = MAX('Dim Date'[date])
VAR _YearStart = DATE(YEAR(_LastDate), 1, 1)
RETURN
CALCULATE(
    SUM('Fact Sales'[ape_usd]),
    DATESINPERIOD('Dim Date'[date], _LastDate, -3, MONTH),
    'Dim Date'[date] >= _YearStart,
    ALL('Dim Date')
)
```

**Fix 2 --- the `_YearStart` cap.** Without it, January's rolling window
would reach back into December of the prior year. Adding
`'Dim Date'[date] >= _YearStart` as a second filter argument inside
`CALCULATE` clamps the window to the current year. January returns
January only; March returns Jan + Feb + Mar; October returns Aug + Sep +
Oct.

### Agent Rank by APE -- three things wrong with the first version

This measure went through three rounds of fixes, each uncovering
something new.

**Round 1 -- Orphan IDs claiming rank #1.**

Fixed by using `ALL('Dim Agent')` instead of `ALLSELECTED` in the
`RANKX` table, and filtering out blank agent names from the ranking
pool.

**Round 2 -- 31 agent names shared by 2--3 different agent IDs.**

The original guard `HASONEVALUE('Dim Agent'[agent_id])` returned FALSE
when a slicer selected a name like "Rahul Reddy" (who has 3 distinct
IDs), so those agents always showed blank rank. Changed the guard to
`HASONEVALUE('Dim Agent'[agent_name])` --- the slicer operates on names,
so that's the right boundary.

**Round 3 -- Rank didn't match the YTD APE shown in the visual.**

The `RANKX` expression was `CALCULATE(SUM('Fact Sales'[ape_usd]))` ---
all-time total APE across all years. The card next to it shows
`[APE YTD]` --- current year only. The two numbers came from different
calculations, so the rank was inconsistent with what the user saw. Also,
14 agents with no sales were all sharing the last rank number.

**Final measure:**

``` dax
Agent Rank by APE =
VAR _Pool =
    FILTER(
        ALL('Dim Agent'),
        NOT ISBLANK('Dim Agent'[agent_name])
            && NOT ISBLANK(CALCULATE([APE YTD]))
    )
RETURN
IF(
    HASONEVALUE('Dim Agent'[agent_name]) && NOT ISBLANK([APE YTD]),
    RANKX(_Pool, [APE YTD], , DESC, DENSE)
)
```

The pool only includes named agents who have YTD APE in the current
period. Agents with no sales return blank rank rather than crowding the
bottom. The rank now always reflects the same YTD APE figure shown on
the card.

### Persistency Rate -- date context matters

``` dax
Persistency Rate =
DIVIDE(
    COUNTROWS(
        FILTER(
            'Fact Persistency',
            'Fact Persistency'[renewal_status] = "renewed"
        )
    ),
    COUNTROWS(
        FILTER(
            'Fact Persistency',
            'Fact Persistency'[renewal_status] IN {"renewed", "lapsed", "grace"}
        )
    )
)
```

No relationship override here --- the measure runs in whatever filter
context the visual sets up. Since `issue_date_key` is the active
relationship, date slicers filter policies by when they were issued. See
the data model section above for the reasoning behind that choice.

### Active Policy Count -- snapshot, not additive

``` dax
Active Policy Count =
CALCULATE(
    COUNTROWS('Fact Persistency'),
    'Fact Persistency'[renewal_status] IN {"renewed", "grace"}
)
```

Counts in-force policies only. This is a semi-additive measure ---
summing it across months gives a meaningless number. It should always be
read within a specific date context (e.g., a single year or a date
slicer selection).

------------------------------------------------------------------------

## What I'd Do Differently With More Time


**Better RLS.**

The Territory Manager role matches `USERPRINCIPALNAME()` to
`agent_name`, which breaks the moment someone changes their display
name. A proper `security_mapping` table mapping email addresses to
territories would be much more robust in production.

**Sort out the orphan agents.**

The unmatched IDs need a conversation with whoever owns the source
system. Short-term fix would be adding an "Unassigned" row to
`Dim Agent` so the revenue is visible but clearly labelled, rather than
disappearing into a blank.

