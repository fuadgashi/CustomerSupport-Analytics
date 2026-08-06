# DAX Measure Catalog

113 measures across nine functional families, all built on `fact_tickets` and the
[function library](dax-functions.md). Every measure carries a `///` description,
a display folder, and a format string (or is legitimately text-returning and exempt
— see [RAG status](#rag-status-color-as-data-not-as-a-visual-override) and
[agent lookups](#agent--customer-rankings) below).

## 1. Core ticket volume & status

```dax
/// Total count of distinct support tickets in scope.
Total Tickets = DISTINCTCOUNT(fact_tickets[ticket_id])

/// Count of tickets that are neither resolved nor closed.
Total Backlog =
    CALCULATE(
        [Total Tickets],
        KEEPFILTERS(NOT dim_status[status_id] IN {1,2})
    )
```

`Open Tickets`, `Closed Tickets`, `In Progress Tickets`, `Escalated Tickets`,
`Resolved Tickets`, and the priority breakdowns (`Total High/Medium/Low/Urgent
Priority Tickets`) follow the same `CALCULATE([Total Tickets], status/priority
filter)` shape.

## 2. Time & service-level metrics

```dax
/// Average resolution time, in hours, across tickets in scope.
Avg Resolution Time = AVERAGE(fact_tickets[resolution_time_hours])

/// Share of tickets first-responded to within the 15-minute SLA target.
Service Level (15 min) =
    VAR fast_first_response = [Fast First Response]
    VAR total_tickets = [Total Tickets]
    RETURN
        DIVIDE(fast_first_response, total_tickets, 0)

/// Current-year, prior-year, and month-over-month variants — composed, not
/// hand-rolled — plus the growth index between each pair.
Service Level (15 min) CY = YearToDay([Service Level (15 min)])
Service Level (15 min) PY = PrevYear([Service Level (15 min)])
Index Dif Service Level Prev Year =
    IndexDif([Service Level (15 min) CY], [Service Level (15 min) PY])

Service Level (15 min) CM = CurrentMonth([Service Level (15 min)])
Service Level (15 min) PM = PrevMonth([Service Level (15 min)])
Index Dif Service Level Prev Month =
    IndexDif([Service Level (15 min) CM], [Service Level (15 min) PM])

/// Actual service level as a % of the target service level.
SL Performance vs Target = DIVIDE([Service Level (15 min)], [Service Level Target], 0)
```

## 3. Quality & escalation

```dax
/// Share of tickets that were escalated, as a percentage of total tickets.
Escalation Rate = DIVIDE([Escalated Tickets], [Total Tickets], 0)

/// Share of tickets that were reopened, as a percentage of total tickets.
Reopened Rate = DIVIDE([Reopened Tickets], [Total Tickets], 0)

/// Share of tickets closed/resolved, as a percentage of total tickets.
Resolution Rate = DIVIDE([Resolved Tickets], [Total Tickets], 0)
```

## 4. Call analytics

```dax
/// Average number of calls logged per ticket.
Avg Calls per Ticket = AVERAGE(fact_tickets[number_of_calls])

/// Tickets that took more than 5 calls to resolve — the high-effort tail.
High Effort Tickets =
    CALCULATE(
        DISTINCTCOUNT(fact_tickets[ticket_id]),
        KEEPFILTERS(fact_tickets[number_of_calls] > 5)
    )

/// Tickets resolved in exactly one call vs. more than one — the single-touch rate.
Single Calls Tickets =
    CALCULATE(DISTINCTCOUNT(fact_tickets[ticket_id]), fact_tickets[number_of_calls] = 1)

Multiple Calls Tickets =
    CALCULATE(DISTINCTCOUNT(fact_tickets[ticket_id]), fact_tickets[number_of_calls] > 1)
```

## 5. Channel metrics — one filter, four channels

Rather than four independent measure trees, every channel metric reuses the same
base measure through a single `CALCULATE` filter — a change to `Avg Resolution Time`
propagates to all four channels automatically:

```dax
/// Count of tickets received through the Email channel.
Email Total = CALCULATE([Total Tickets], dim_channel[channel_name] = "Email")

/// Average resolution time, in hours, for tickets received through the Email channel.
Email Resolution = CALCULATE([Avg Resolution Time], dim_channel[channel_name] = "Email")

/// Average customer satisfaction score for tickets received through the Email channel.
Email CSAT = CALCULATE([Avg CSAT], dim_channel[channel_name] = "Email")

/// Escalation rate for tickets received through the Email channel.
Email Escalation = CALCULATE([Escalation Rate], dim_channel[channel_name] = "Email")
```

`Phone …`, `Chat …`, and `Social …` repeat the identical four-measure pattern with
their own channel filter.

## 6. Agent & customer rankings

```dax
/// Fixed constant: target ticket load per agent.
Target Tickets per Agent = 170

/// Actual ticket count vs. the per-agent target — a signed variance, not a ratio.
Ticket Performance Gap = [Total Tickets] - [Target Tickets per Agent]

/// Text-returning: the name of the agent with the highest call volume, not a number.
Top Agent Calls =
    VAR TopCalls = MAXX(ALL(dim_agent[agent_name]), CALCULATE([Total Calls]))
    RETURN
        CALCULATE(SELECTEDVALUE(dim_agent[agent_name]), FILTER(ALL(dim_agent), CALCULATE([Total Calls]) = TopCalls))
```

`Lowest Agent Calls`, `Top Agent Tickets`, and `Lowest Agent Tickets` follow the same
`MAXX`/`MINX`-then-`SELECTEDVALUE` shape — each intentionally returns a **name**, so
none of the four carries a format string; that's the correct, documented exemption
for a text-returning measure, not an oversight.

## 7. Customer metrics

```dax
Total Customers CY = YearToDay([Total Customer])
Total Customers PY = PrevYear([Total Customer])
```

`Total Customer`, `Avg Tickets per Customer`, and `Avg Calls per Customer` round out
the family — every ratio built with `DIVIDE`, every time comparison built with the
function library, never inline.

## 8. Share & basic calculations

```dax
/// This category's share of tickets among all categories currently in view.
Volume Share % =
    DIVIDE(
        [Total Tickets],
        CALCULATE([Total Tickets], ALLSELECTED(dim_category))
    )
```

`ALLSELECTED` rather than `ALL` is deliberate here: the share recalculates against
whatever categories are currently filtered (e.g. by a slicer), not silently against
the full unfiltered table.

## RAG status — color as data, not as a visual override

16 measures return a **theme color name** — `"good"` / `"neutral"` / `"bad"` — never
a hex value, so the color survives a theme switch instead of being frozen to one:

```dax
/// Theme color name (good/neutral/bad) for Average Tickets per Agent: >=170 good, >=140 neutral, else bad.
Average Tickets per Agent | RAG Status =
    VAR _v = [Average Tickets per Agent]
    RETURN
        SWITCH(TRUE(), _v >= 170, "good", _v >= 140, "neutral", "bad")
```

Same reasoning as the agent-name lookups above: these are text-returning by design,
so they correctly carry no format string. Every KPI card with a color-band indicator
in the report reads its color from one of these 16, keyed to that card's own
thresholds.
