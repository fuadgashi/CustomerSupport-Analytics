# DAX Function Library

24 DAX user-defined functions carry every time-window, period-average, and
current-vs-prior comparison in the model — measures compose them rather than
hand-rolling `DATEADD`/`DATESYTD` per KPI. Three families:

## Period-window functions

Each takes any measure expression and re-evaluates it over a shifted or widened date
window.

```dax
/// Year-to-date up to today: evaluates the measure for the current calendar year, dates up to and including TODAY(). KEEPFILTERS preserves any existing date filters.
function YearToDay =
    (cal_funct: anyref expr) =>
        CALCULATE(
            cal_funct,
            KEEPFILTERS(DimDate[Year] = YEAR(TODAY())),
            KEEPFILTERS(DimDate[Date] <= TODAY())
        )

/// Rest-of-year from today: evaluates the measure for the current calendar year, dates from TODAY() onward. The forward-looking counterpart of YearToDay.
function YearToGo =
    (cal_funct: anyref expr) =>
        CALCULATE(
            cal_funct,
            KEEPFILTERS(DimDate[Year] = YEAR(TODAY())),
            KEEPFILTERS(DimDate[Date] >= TODAY())
        )

/// Prior year: shifts the measure back one year using DATEADD and restricts to the previous calendar year. Used for year-over-year comparisons.
function PrevYear =
    (cal_funct: anyref expr) =>
        CALCULATE(
            cal_funct,
            DATEADD(DimDate[Date], -1, YEAR),
            DimDate[Year] = YEAR(TODAY()) - 1
        )

/// Current month to date: restricts to the current month and year, dates up to and including TODAY().
function CurrentMonth =
    (cal_funct: anyref expr) =>
        CALCULATE(
            cal_funct,
            KEEPFILTERS(DimDate[Month] = MONTH(TODAY())),
            KEEPFILTERS(DimDate[Year] = YEAR(TODAY())),
            KEEPFILTERS(DimDate[Date] <= TODAY())
        )

/// Prior month: restricts to the month before the current one, within the current year.
function PrevMonth =
    (cal_funct: anyref expr) =>
        CALCULATE(
            cal_funct,
            KEEPFILTERS(DimDate[Year] = YEAR(TODAY())),
            KEEPFILTERS(DimDate[Month] = MONTH(TODAY()) - 1)
        )

/// Rolling 12 months: evaluates the measure for dates from 12 months before today onward, respecting the current filter context.
function TwelMonth =
    (cal_funct: anyref expr) =>
        VAR Twel_Month = EDATE(TODAY(), -12)
        RETURN
            CALCULATE(cal_funct, KEEPFILTERS(DimDate[Date] >= Twel_Month))

/// Rolling 12 months ignoring external filter context. Accepts any measure expression; typed as anyref to match the windowing family and avoid int64 truncation of decimal inputs.
function TwelMonthConst =
    (cal_funct: anyref expr) =>
        VAR Twel_Month = EDATE(TODAY(), -12)
        RETURN
            CALCULATE(cal_funct, KEEPFILTERS(DimDate[Date] >= Twel_Month), REMOVEFILTERS())

/// Rolling 6 months: evaluates the measure for dates from 6 months before today onward.
function SixMonth =
    (cal_funct: anyref expr) =>
        VAR Six_Month = EDATE(TODAY(), -6)
        RETURN
            CALCULATE(cal_funct, KEEPFILTERS(DimDate[Date] >= Six_Month))

/// Rolling 3 months: evaluates the measure for dates from 3 months before today onward.
function ThreeMonth =
    (cal_funct: anyref expr) =>
        VAR Three_Month = EDATE(TODAY(), -3)
        RETURN
            CALCULATE(cal_funct, KEEPFILTERS(DimDate[Date] >= Three_Month))
```

## Period-average & period-extreme functions

```dax
/// Running daily average: averages the measure across all days up to the latest date in context. Returns blank if the measure is blank.
function AvgDay =
    (cal_funct: anyref expr) =>
        VAR DataMax = MAX(DimDate[Date])
        RETURN
            IF(ISBLANK(cal_funct), BLANK(), CALCULATE(AVERAGEX(FILTER(ALL(DimDate[Date]), DimDate[Date] <= DataMax), cal_funct)))

/// Constant daily average to today: averages the measure across all days up to TODAY(), independent of the date selected in context.
function AvgDayConst =
    (cal_funct: anyref expr) =>
        VAR CurrentDate = TODAY()
        RETURN
            IF(ISBLANK(cal_funct), BLANK(), CALCULATE(AVERAGEX(FILTER(ALL(DimDate[Date]), DimDate[Date] <= CurrentDate), cal_funct)))

/// Weekly average: averages the measure across all weeks up to the current week, removing other date filters. Blank-safe.
function AvgWeek =
    (cal_funct: anyref expr) =>
        IF(ISBLANK(cal_funct), BLANK(), CALCULATE(AVERAGEX(FILTER(ALL(DimDate[Week]), DimDate[Week] <= MAX(DimDate[Week])), cal_funct), REMOVEFILTERS(DimDate)))

/// Monthly average: averages the measure across all months up to the current month, removing other date filters. Blank-safe.
function AvgMonth =
    (cal_funct: anyref expr) =>
        IF(ISBLANK(cal_funct), BLANK(), CALCULATE(AVERAGEX(FILTER(ALL(DimDate[Month]), DimDate[Month] <= MAX(DimDate[Month])), cal_funct), REMOVEFILTERS(DimDate)))

/// Maximum daily value: returns the highest value of the measure across all days. Blank-safe.
function MaxDay =
    (cal_funct: anyref expr) =>
        IF(ISBLANK(cal_funct), BLANK(), MAXX(ALL(DimDate[Date]), CALCULATE(cal_funct)))

/// Maximum weekly value: returns the highest value of the measure across all weeks (ALLEXCEPT keeps the week grain). Blank-safe.
function MaxWeek =
    (cal_funct: anyref expr) =>
        IF(ISBLANK(cal_funct), BLANK(), MAXX(ALL(DimDate[Date]), CALCULATE(cal_funct, ALLEXCEPT(DimDate, DimDate[Week]))))

/// Maximum monthly value: returns the highest value of the measure across all months. Blank-safe.
function MaxMonth =
    (cal_funct: anyref expr) =>
        IF(ISBLANK(cal_funct), BLANK(), MAXX(ALL(DimDate[Month]), CALCULATE(cal_funct, ALLEXCEPT(DimDate, DimDate[Month]))))

/// Maximum yearly value: returns the highest value of the measure across all years. Blank-safe.
function MaxYear =
    (cal_funct: anyref expr) =>
        IF(ISBLANK(cal_funct), BLANK(), MAXX(ALL(DimDate[Year]), CALCULATE(cal_funct, ALLEXCEPT(DimDate, DimDate[Year]))))

/// Minimum daily value: returns the lowest value of the measure across all days. Blank-safe.
function MinDay =
    (cal_funct: anyref expr) =>
        IF(ISBLANK(cal_funct), BLANK(), MINX(ALL(DimDate), CALCULATE(cal_funct)))

/// Minimum monthly value: returns the lowest value of the measure across all months.
function MinMonth =
    (cal_funct: anyref expr) =>
        IF(ISBLANK(cal_funct), BLANK(), MINX(ALL(DimDate[Month]), CALCULATE(cal_funct, ALLEXCEPT(DimDate, DimDate[Month]))))
```

## Comparison functions

Every "vs. prior year/month" pair funnels through one of these, each sharing the
same blank / future-date / past-date guard logic.

```dax
/// Percentage difference (CurrentValue - PriorValue) / PriorValue for normal positive-base measures. Guards: blank when both blank or current-blank on a future date; returns 1 when current exists but prior was blank on a past date, or when prior is negative. For negative-base measures use IndexDifAbsolut instead.
function IndexDif =
    (CurrentValue: decimal val, PriorValue: decimal val) =>
        VAR _IsCurrentBlank = ISBLANK(CurrentValue)
        VAR _IsPriorBlank = ISBLANK(PriorValue)
        VAR _IsFutureDate = MAX(DimDate[Date]) > TODAY()
        VAR _IsPastDate = MAX(DimDate[Date]) < TODAY()
        RETURN
            SWITCH(
                TRUE(),
                _IsCurrentBlank && _IsPriorBlank, BLANK(),
                _IsCurrentBlank && _IsFutureDate && NOT (_IsPriorBlank), BLANK(),
                NOT (_IsCurrentBlank) && _IsPriorBlank && _IsPastDate, 1,
                NOT (_IsCurrentBlank) && PriorValue < 0, 1,
                DIVIDE(CurrentValue - PriorValue, PriorValue)
            )

/// Index (ratio) difference for measures whose base can be NEGATIVE. Uses ABS(PriorValue) in the denominator so the percentage keeps its real direction when the base is below zero.
function IndexDifAbsolut =
    (CurrentValue: decimal val, PriorValue: decimal val) =>
        VAR _IsCurrentBlank = ISBLANK(CurrentValue)
        VAR _IsPriorBlank = ISBLANK(PriorValue)
        VAR _IsFutureDate = MAX(DimDate[Date]) > TODAY()
        VAR _IsPastDate = MAX(DimDate[Date]) < TODAY()
        RETURN
            SWITCH(
                TRUE(),
                _IsCurrentBlank && _IsPriorBlank, BLANK(),
                _IsCurrentBlank && _IsFutureDate && NOT (_IsPriorBlank), BLANK(),
                NOT (_IsCurrentBlank) && _IsPriorBlank && _IsPastDate, -1,
                DIVIDE(CurrentValue - PriorValue, ABS(PriorValue))
            )

/// Achievement ratio CurrentValue / PriorValue (e.g. actual vs. target) for positive-base measures.
function IndexQuot =
    (CurrentValue: decimal val, PriorValue: decimal val) =>
        VAR _IsCurrentBlank = ISBLANK(CurrentValue)
        VAR _IsPriorBlank = ISBLANK(PriorValue)
        VAR _IsFutureDate = MAX(DimDate[Date]) > TODAY()
        VAR _IsPastDate = MAX(DimDate[Date]) < TODAY()
        RETURN
            SWITCH(
                TRUE(),
                _IsCurrentBlank && _IsPriorBlank, BLANK(),
                _IsCurrentBlank && _IsFutureDate && NOT (_IsPriorBlank), BLANK(),
                NOT (_IsCurrentBlank) && _IsPriorBlank && _IsPastDate, 1,
                NOT (_IsCurrentBlank) && PriorValue < 0, 1,
                DIVIDE(CurrentValue, PriorValue)
            )

/// Achievement ratio for NEGATIVE-base measures. Omits the 'prior < 0 -> 1' guard since the ratio of two same-sign negatives comes out positive naturally.
function IndexQuotABS =
    (CurrentValue: decimal val, PriorValue: decimal val) =>
        VAR _IsCurrentBlank = ISBLANK(CurrentValue)
        VAR _IsPriorBlank = ISBLANK(PriorValue)
        VAR _IsFutureDate = MAX(DimDate[Date]) > TODAY()
        VAR _IsPastDate = MAX(DimDate[Date]) < TODAY()
        RETURN
            SWITCH(
                TRUE(),
                _IsCurrentBlank && _IsPriorBlank, BLANK(),
                _IsCurrentBlank && _IsFutureDate && NOT (_IsPriorBlank), BLANK(),
                NOT (_IsCurrentBlank) && _IsPriorBlank && _IsPastDate, 1,
                DIVIDE(CurrentValue, PriorValue)
            )

/// Absolute value difference (CurrentValue - PriorValue). Mirrors the comparison-family guard for blanks and future/past dates. Unlike the Index functions this returns a value delta, so it does NOT special-case negative prior with a literal 1.
function Varianc =
    (CurrentValue: decimal val, PriorValue: decimal val) =>
        VAR _IsCurrentBlank = ISBLANK(CurrentValue)
        VAR _IsPriorBlank = ISBLANK(PriorValue)
        VAR _IsFutureDate = MAX(DimDate[Date]) > TODAY()
        VAR _IsPastDate = MAX(DimDate[Date]) < TODAY()
        RETURN
            SWITCH(
                TRUE(),
                _IsCurrentBlank && _IsPriorBlank, BLANK(),
                _IsCurrentBlank && _IsFutureDate && NOT (_IsPriorBlank), BLANK(),
                NOT (_IsCurrentBlank) && _IsPriorBlank && _IsPastDate, CurrentValue,
                CurrentValue - PriorValue
            )
```

## How measures compose them

```dax
'Service Level (15 min) CY' = YearToDay([Service Level (15 min)])
'Service Level (15 min) PY' = PrevYear([Service Level (15 min)])
'Index Dif Service Level Prev Year' = IndexDif([Service Level (15 min) CY], [Service Level (15 min) PY])

'Service Level (15 min) CM' = CurrentMonth([Service Level (15 min)])
'Service Level (15 min) PM' = PrevMonth([Service Level (15 min)])
'Index Dif Service Level Prev Month' = IndexDif([Service Level (15 min) CM], [Service Level (15 min) PM])
```

One base measure (`Service Level (15 min)`), four function calls, and a full
current-year / prior-year / current-month / prior-month comparison set falls out —
the same composition pattern backs `Total Calls`, `Total Tickets`, and `Total
Customers`.
