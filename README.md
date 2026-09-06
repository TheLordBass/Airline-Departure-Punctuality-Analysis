# Airline Departure Punctuality Analysis

Power BI analysis of 18 months of short-haul departure performance for **Northline Air**, a fictional UK carrier operating from Manchester, Gatwick and Edinburgh across 18 European destinations.

**Period:** January 2025 – June 2026  
**Scope:** 45,886 flight legs, 25 aircraft, 18 airports  
**Tools:** Power BI (Power Query / M, DAX)

> The dataset is synthetic and generated for this project. It does not represent any real airline's operations.

**Companion project:** customer complaints for the same carrier and period — [Airline Customer Complaints Analysis](https://github.com/TheLordBass/Airline-Customer-Complaints-Analysis).

---

## The dashboard

![Departure punctuality dashboard](Delay%20Dashboard.png)

One page, driven by the date slicer: six KPI cards across the top, OTP15 against target by month, delay minutes split by controllability, and delay and cancellation rates ranked by station. Station charts use rates rather than counts, for the reason set out below.


---

## Headline findings

**OTP15 averaged 78.2% against an 80% target, missing target in 9 of 18 months.**

Punctuality is strongly seasonal, and the pattern repeats across both years in the dataset:

| | Best months | Worst months |
|---|---|---|
| | Oct 2025 (81.7%) | Feb 2025 (70.9%) |
| | May 2026 (81.4%) | Feb 2026 (71.7%) |
| | Nov 2025 (81.3%) | Jan 2026 (72.5%) |

Both February troughs sit close to 71%, roughly 9 points below the autumn peaks. Winter disruption is the single largest driver of missed target, and it is predictable — which makes it a planning problem rather than an operational surprise.

**Reactionary delay is the most expensive category per event.**

Splitting delay minutes by controllability changes the picture depending on which measure you use, and the distinction matters:

| Controllability | Delayed flights | Total delay minutes | Share of minutes | Avg minutes per delay |
|---|---|---|---|---|
| Controllable | 6,369 | 154,595 | 36.7% | 24.3 |
| Uncontrollable | 6,164 | 149,770 | 35.5% | 24.3 |
| Reactionary | 3,924 | 117,111 | 27.8% | **29.8** |

Reactionary delay — knock-on from a late inbound aircraft — accounts for the fewest events but the longest average delay. It also represents delay that did not originate anywhere: it is inherited from earlier in the same aircraft's day. Roughly 28% of all delay minutes are therefore a consequence of other delays rather than a cause in themselves, which suggests turnaround buffer is the highest-leverage intervention available.

**Station performance does not track station size.**

| Station | Flights | Delay rate | Avg delay (mins) |
|---|---|---|---|
| CDG | 1,453 | 27.9% | 12.5 |
| AMS | 1,519 | 27.3% | 13.2 |
| FCO | 1,126 | 26.3% | 12.5 |
| ... | | | |
| MAN | 8,108 | 19.3% | 8.8 |
| TFS | 1,661 | 18.1% | 9.4 |
| FAO | 1,731 | 18.1% | 8.7 |

Ranked by count of delays, the largest bases dominate simply because they operate the most flights. Ranked by *rate*, the three worst stations are congested continental hubs with modest Northline volume, and Manchester — the second-largest base — sits among the better performers. Any station-level chart on this dashboard therefore uses a rate rather than a count.

**Cancellation rate is 0.59%** (270 of 45,886), consistent with typical European short-haul. ALC is the weakest station at 0.93%; TFS the strongest at 0.24%.

**Load factor is 81.7%** across the period.

---

## Data quality note

**1,106 of 9,898 delayed flights (11.2%) carry no delay code.** These are flights delayed more than 15 minutes where no cause was recorded at the station.

This is worth reporting in its own right. Unallocated delay means roughly one in nine delay events cannot be attributed to a controllable or uncontrollable cause, which puts a confidence band around the controllability split above. Any improvement programme targeting controllable delay is working from incomplete attribution.

---

## Methodology

### Cleaning (Power Query)

The source extract carried the kinds of problems typical of data assembled from several operational systems. Each was handled as a named applied step so the transformation history is auditable.

| Issue | Volume | Treatment |
|---|---|---|
| Duplicate flight records | 642 rows | `Table.Distinct` on `flight_id` |
| Airport codes with leading whitespace and mixed case | 90 distinct values for 18 airports | Trim + uppercase on `origin` and `destination` |
| Aircraft registrations missing hyphen or lowercased | ~1,850 rows | Conditional transform re-inserting the hyphen after the first character |
| Delay causes recorded as free text instead of code | 8 distinct variants | Mapped back to IATA codes via `Record.FieldOrDefault` |
| `-999` sentinel in delay columns | 91 rows | Replaced with `null` |
| Empty trailing column from the CSV | — | Removed |

### Decisions worth explaining

**`-999` was nulled, not filtered.** Removing those rows would have dropped 91 real flights from every count on the dashboard. Replacing the value with null preserves the flight while excluding it from any average, which is the correct behaviour for a sentinel.

**`departure_date` was used as the date key, not `flight_date`.** The two columns disagree on 1,047 flights. This is not an error — it reflects late-evening departures where the scheduled operating day and the actual departure timestamp fall either side of midnight, a standard distinction in airline data. Since every measure here concerns departure punctuality, the date dimension is aligned to the departure timestamp. `flight_date` also arrived in two formats (`dd/mm/yyyy` and `dd-MMM-yy`) and was dropped from the model to remove the risk of it being used by mistake.

**Cancelled flights are excluded from OTP but included in cancellation rate.** OTP measures the punctuality of flights that operated; a cancelled flight has no departure time and cannot be on or off time. Cancellation rate needs cancelled flights in its denominator or the metric is meaningless. The two are reported separately rather than being combined into a single "completion" figure.

**Blank delay values are excluded from the OTP population.** DAX coerces `BLANK()` to zero in a comparison, so `departure_delay_mins <= 15` silently counts unrecorded flights as on time. An explicit `NOT ISBLANK()` filter removes 397 flights from the measurable population, giving 45,489 eligible flights. Without it OTP overstates by roughly 0.3 points.

**OTP15 and Delay Rate share a denominator.** Both are calculated against a single `Eligible Flights` measure so they always sum to exactly 100%. An earlier version had the two measures drifting apart on their filter conditions; sharing the base makes that failure impossible.

### Data model

Star schema with `flights` as the fact table.

- `Date Table` — generated with `CALENDAR()`, marked as a date table, related to `flights[departure_date]`
- `delay_codes` — IATA-style delay reference providing `delay_category` and `controllability`
- `aircraft` — fleet register
- `airports` — station reference

`Month Year` is set to sort by a `yyyy-mm` column so the time axis orders chronologically rather than alphabetically.

### Core measures

```dax
Eligible Flights =
CALCULATE(
    COUNTROWS( flights ),
    flights[cancelled] = 0,
    NOT ISBLANK( flights[departure_delay_mins] )
)

OTP15 % =
VAR OnTime =
    CALCULATE(
        COUNTROWS( flights ),
        flights[cancelled] = 0,
        NOT ISBLANK( flights[departure_delay_mins] ),
        flights[departure_delay_mins] <= 15
    )
RETURN DIVIDE( OnTime, [Eligible Flights] )

Delay Rate % =
VAR Delayed =
    CALCULATE(
        COUNTROWS( flights ),
        flights[cancelled] = 0,
        NOT ISBLANK( flights[departure_delay_mins] ),
        flights[departure_delay_mins] > 15
    )
RETURN DIVIDE( Delayed, [Eligible Flights] )

Cancellation Rate % =
DIVIDE(
    CALCULATE( COUNTROWS( flights ), flights[cancelled] = 1 ),
    COUNTROWS( flights )
)
```

---

## Definitions

**OTP15** — share of operated flights departing within 15 minutes of scheduled departure. The standard industry punctuality measure. Cancelled flights excluded.

**Delay Rate** — share of operated flights departing more than 15 minutes late. The exact complement of OTP15.

**Controllability** — whether the delay cause is within the airline's control (technical, crew, boarding, ramp handling), outside it (weather, ATC, airport congestion), or reactionary (knock-on from a late inbound aircraft).

**Load Factor** — passengers boarded as a share of seats available.

---

## Repository contents

| File | Description |
|---|---|
| `Airline data.pbix` | Power BI report, including all Power Query steps and DAX measures |
| `Delay Dashboard.png` | Screenshot of the report page |

---

## Possible extensions

**Delay propagation.** The reactionary finding above is descriptive. Sequencing flights by aircraft and date would allow the knock-on effect to be quantified directly — how many downstream minutes an average morning delay generates, and at what turnaround buffer propagation stops. This is the most operationally useful question the dataset can answer and is the intended next iteration.

**Cost of delay.** The source data includes fuel, landing and handling costs at flight level, which would support a cost-per-delay-minute view and a return-on-investment case for additional turnaround buffer.
