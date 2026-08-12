# Data dictionary — `utility_data_<STATE>.csv`

Downloaded from the Data Download section of the
[gas utility map](https://thefutureofheat.github.io/utility-map/).

One row per **(utility, year, EIA filer, PHMSA operator, filing)**. 91 columns.

Sources are two federal collections, joined but never blended:

| | source | authority |
|---|---|---|
| `eia_*`, bracketed columns | EIA **Form 176**, "Annual Report of Natural and Supplemental Gas Supply and Disposition" | EIA |
| `MMILES_*`, `NUM_SRVS_*`, `PARTD*`, `PERCENT_UNACC_GAS` | PHMSA **Form F 7100.1-1**, "Annual Report for Gas Distribution System" | PHMSA, 49 CFR Part 191 |

## Which utilities are in here

A utility is included if any of these is true:

1. it is **on the map** — it has a territory polygon;
2. it is **matched across EIA and PHMSA**, even without a polygon;
3. it is a **predecessor** of a utility that qualifies under 1 or 2.

Everything else is left out: a utility that appears in only one source, has no
territory, and is nobody's predecessor cannot be reached from the map at all.

Most of what this excludes is not a gas distribution utility — interstate pipelines,
LNG and vehicle-fuel marketers, and EIA's own "Adjustment Company" balancing entry,
which exists to make state totals reconcile. Nationally the exclusion is large:
2,570 of 3,904 utilities fall outside rules 1 and 2.

It does exclude some genuine utilities. In New Hampshire that is **New Hampshire Gas
Corp**, which has 22 years of PHMSA filings but no EIA data, no polygon and no
successor — so nothing on the map leads to it.

### Predecessors share their successor's name

Rule 3 exists because the map already shows a predecessor's data, folded into the
successor's trend lines. A download that omitted it would contradict the map.

A predecessor row therefore carries its **successor's** `operator`, `territory_name`
and `fid`, and is flagged `on_map = yes` on the successor's behalf — you reach it by
clicking the polygon its territory became. Its own `eia_id`, `phmsa_id`, `eia_name`
and `phmsa_name` are untouched, and `predecessor_of` names the successor.

So in New Hampshire, `LIBERTY UTILITIES DBA ENERGY NORTH` covers two filing entities:

| years | `eia_id` | `phmsa_id` | filed as |
|---|---|---|---|
| 1997–2000 | `17604829NH` | — | EnergyNorth Nat Gas Inc |
| 2003–2011 | `17604829NH` | `16667` | EnergyNorth Nat Gas Inc |
| 2012–2025 | `17675803NH` | `16667` | Liberty Utilities dba EnergyNorth |

One row per utility-year throughout. Note EnergyNorth filed nothing for 2001 or 2002,
and PHMSA operator 16667 does not begin until 2003; both gaps are genuine and preserved
rather than smoothed over.

Predecessor chains are followed all the way through, so where A became B became C,
A and B both carry C's name.

### PHMSA filings are attributed by year, not by entity

The two agencies disagree about when a utility becomes a new entity. PHMSA operator
16667 filed continuously from 2003 to 2025, changing only its *name* as ownership
changed. EIA issued a **new respondent id** at the 2012 Liberty acquisition. So there is
no year at which both agencies agree a new company began.

Each PHMSA filing is therefore attributed to whichever entity actually filed with EIA
that year — 16667's 2003–2011 filings sit with EnergyNorth, its 2012+ filings with
Liberty. Years nobody filed with EIA fall back to the current identity, which is why
2025 sits with Liberty.

The practical consequence: **`match_status` describes the row, not the company.** The
same utility can be `matched` in some years and single-source in others, because that is
what the underlying filings do. EnergyNorth is `eia_only` for 1997–2000, when no PHMSA
operator in its lineage was filing, and `matched` from 2003.

## The one thing to know before using this file

**Every value is as filed.** Where the map's click panel fills a utility's early
years from the companies it absorbed, this file does not. Gaps stay gaps, and a
utility's series starts the year it first filed under its own name.

So panel and CSV totals can legitimately differ. The panel shows Liberty Utilities dba
EnergyNorth with EIA data from 1997, backfilled from EnergyNorth Nat Gas Inc; here those
years stay on EnergyNorth's own rows under its own `eia_id`. Group by `operator` to get
the continuous series the panel shows. Nothing has been apportioned across states either
— a combined multi-state filing is kept whole and flagged in `combined_report`.

**Do not sum a column without filtering `phmsa_report_role = primary`.** An
operator can file several reports for one state and year, and all of them are
here. Summing every row double-counts. See rows 12–13.

**Check `n_phmsa_operators`, `n_eia_filers` and `combined_report` before
aggregating.** A merger year splits one utility across several still-separate
filings — on either agency's side — and a multi-state filing is not specific to this
state at all. All three are explained in "Aggregating safely" below, with worked
examples.

## Aggregating safely

**A utility-year can span several rows.** That is the one thing to internalise
before aggregating. A merger leaves the companies that were absorbed still filing
separately for years, and *either* agency's side can be the one that splits.

The rule is simple and symmetric: **sum across the rows of a `(operator, year)`
group.** Each distinct filer's figures appear exactly once, so summing cannot
double-count — that is checked at build time. Two columns tell you when a group
has more than one row:

| column | means |
|---|---|
| `n_phmsa_operators` | how many PHMSA operators the utility-year draws on |
| `n_eia_filers` | how many EIA filers it draws on |

Both are `1` in the ordinary case. The two Black Hills Energy entities happen to
demonstrate one of each.

### Several PHMSA operators — Wyoming

One EIA identity from 1997 to 2025, but **three** PHMSA operators filing until 2016:

| operator | name |
|---|---|
| 10030 | SourceGas LLC |
| 2332 | Cheyenne Light Fuel & Power |
| 2537 | Black Hills Northwest Wyoming Gas Utility |
| 15359 | Black Hills Energy — 2016 onward, the merged filer |

Summing `MMILES_TOTAL` across those rows reproduces the map exactly: 2,723 miles
in 2004, 3,375 in 2015, and from 2016 a single operator needing no summing at all.
The EIA figures sit on the largest contributor's row only, so they are not
multiplied by three.

### Several EIA filers — Arkansas

The mirror image. One PHMSA operator, but **two** EIA filers — both named Arkansas
Western Gas Co (`17600509AR` and `17600512AR`) — filing separately until 2003,
after which Black Hills Energy (`17671155AR`) files alone:

| year | rows | summing residential customers | map shows |
|---|---|---|---|
| 1997 | 2 | 18,243 + 91,940 = **110,183** | 110,183 |
| 2003 | 2 | 16,826 + 105,603 = **122,429** | 122,429 |
| 2004 | 1 | 124,797 | 124,797 |

Again the sum matches the map exactly. The PHMSA filing for those years is attached
to the larger of the two filers, so the mileage is not counted twice either.

### A multi-state filing is not about this state — `combined_report`

Some operators file one report covering several states. `combined_report = yes`
means the figures on that row are the operator's **whole multi-state system**, not
its share of this state, and `phmsa_stop` shows which states (e.g. `CO NE WY`).
Nothing has been apportioned — deliberately, since any split would be an estimate.

This can dominate a total. SourceGas filed Wyoming combined with Colorado,
Kansas and Nebraska for 1997–2003, so for Black Hills Energy:

| year | sum of all rows | excluding combined rows | Wyoming's real system |
|---|---|---|---|
| 1997 | 9,854 mi | 814 mi | ~2,700 mi |
| 2002 | 29,419 mi | 949 mi | ~2,700 mi |
| 2004 | 2,723 mi | 2,723 mi | 2,723 mi |

Neither of the first two columns is Wyoming: summing everything counts a
four-state system, and dropping the combined rows loses the largest operator
entirely. For years like these there is no state-specific figure in the source,
and this file will not invent one. Treat them as unavailable, or use the map,
which apportions and is explicit that the result is an estimate.

Two further caveats on that example: the map shows the same inflated 9,854 for
1997, because with no per-state history to apportion from it copies the filing
whole; and the 2002 filing of 28,470 miles is anomalous against roughly 9,000
either side, a problem in PHMSA's 2002 file that predates any handling here.

---

## Identity and provenance (1–19)

Constructed for this file, not taken from either source form.

| # | column | definition |
|---|---|---|
| 1 | `eia_id` | EIA Form 176 respondent id, with a two-letter state suffix (`17675803NH`). Blank on PHMSA-only rows. |
| 2 | `phmsa_id` | PHMSA `OPERATOR_ID`, the operator's federal id. Blank on EIA-only rows. |
| 3 | `phmsa_stop` | PHMSA `STOP` — the state the filing covers. Normally one two-letter code; **two or more codes means one report covering several states** (see `combined_report`). |
| 4 | `state` | The state this file was generated for. Constant within a file. |
| 5 | `year` | Calendar year the data describes: PHMSA `REPORT_YEAR`, EIA `YEAR`. |
| 6 | `operator` | **Shared display name for the utility as the map presents it.** Normally `eia_name`, or `phmsa_name` where there is no EIA filing — but a predecessor carries its *successor's* name, so both read as one utility (see above). Group on this to get the full history; join on the ids to identify a specific filing entity. |
| 7 | `eia_name` | Name as filed with EIA, taken from the utility's most recent filing, so it is stable across the series rather than changing mid-history. |
| 8 | `phmsa_name` | The PHMSA operator's **current** name, from its most recent filing — the same name the map's Identifiers section shows. Not the name it filed under in this row's year: operator 16667 filed as "Keyspan Energy Delivery - Energy North" in 2003 and "Energy North Natural Gas Inc" from 2007, and every row carries the latter so one operator does not read as several. The EIA and PHMSA names often differ in spelling and sometimes in corporate identity. |
| 9 | `ownership` | PHMSA `OPERATOR_TYPE`, a fixed vocabulary the operator selects for itself — Investor Owned, Municipal, Cooperative, Private and similar. Preferred over the territory shapefile's ownership field, which mislabels non-municipal utilities as municipal. **PHMSA only collects this from 2015**, so earlier years are blank. Blank on EIA-only rows. |
| 10 | `match_status` | Describes **this row's year**, not the company. `matched` — this utility-year ties an EIA identity to a PHMSA filing. `eia_only` — an EIA filing with no PHMSA filing attributed to that year. `phmsa_only` — a PHMSA filing with no EIA identity. One utility can be `eia_only` early and `matched` later, because that is what the filings do; see "attributed by year" above. |
| 11 | `predecessor_of` | The `eia_id` (or PHMSA key) of the utility this one **became**. Blank for a utility filing under its own current identity. When set, this row's `operator`, `territory_name`, `fid` and `on_map` are inherited from that successor. |
| 12 | `phmsa_report_role` | `primary` — the filing the map treats as the year's figures: the most complete one, then the largest. `additional` — every other filing for that operator, state and year, kept so nothing is dropped. Blank on EIA-only rows. |
| 13 | `n_phmsa_reports` | How many PHMSA filings **one operator** made for that state and year. `1` for most. Where it is greater, that many rows appear, one `primary` and the rest `additional`. Not to be confused with column 14. |
| 14 | `n_phmsa_operators` | How many **distinct PHMSA operators** this utility-year draws on. `1` is ordinary. Greater than 1 is a merger year in which the acquired companies were still filing separately, so the PHMSA columns must be summed across those rows to get the whole utility — see "Aggregating safely". `0` where the year has EIA data but no PHMSA filing. |
| 15 | `n_eia_filers` | The same thing on the EIA side: how many **distinct EIA filers** this utility-year draws on. Greater than 1 means several companies that later merged were still filing EIA separately, so the EIA columns must be summed across those rows. `0` where the year has a PHMSA filing but no EIA data. |
| 16 | `combined_report` | `yes` when `phmsa_stop` names more than one state, i.e. the figures are the operator's **whole multi-state system**, not its share of this state. Not apportioned — deliberately, since any split would be an estimate. A `yes` row cannot be summed as though it were state-specific; see "Aggregating safely". |
| 17 | `on_map` | `yes` when this utility is reachable on the map — it has a territory polygon, or it is a predecessor of a utility that does. `no` is not an error: nationally 176 utilities matched in both sources have no polygon. |
| 18 | `territory_name` | Name of the map polygon. A predecessor shows the polygon its territory became. Blank when `on_map = no`. |
| 19 | `fid` | The map's stable feature id: `eia:<eia_id>`, or `geo:<territory>|<state>` for polygons carrying no filer id. Use it to tie a row back to a shape. Shared between a predecessor and its successor, since they describe one territory. |

## EIA Form 176 (20–37)

Five of the form's six sales sectors. The sixth, "Other Sales", is for filers who
cannot categorise a customer; nationally in 2023 two filers used it, for 4,392 Mcf
between them, so it is omitted.

**These are sales, not transportation.** A utility that only *delivers* gas a
customer bought from a marketer reports it on the form's Transport lines, which are
not in this file — so its sales columns are blank rather than filled with delivery
figures. The two are not interchangeable: transport revenue is a delivery fee, and
nationally the medians are $2.26/Mcf against $13.61 for sales. A utility with
customers but no sales volume is likely delivery-only; the map's panel shows its
transport figures under a separate "Fee $/Mcf" heading.

Bracketed suffixes are EIA's value-type codes:

- **`[VL]`** volume, in **Mcf** (thousand cubic feet)
- **`[CS]`** EIA's "cost", in **US dollars**. Per the form instructions these are *gross*
  revenues for gas sold and delivered directly to end-use customers, including demand
  charges, commodity charges, taxes, surcharges, adjustments and any gains or losses on
  financial hedges, rounded to whole dollars. `[CS] ÷ [VL]` gives $/Mcf.
- **`[CT]`** count of consumers — specifically the **average over the year**: the number
  attached to the system at the end of each month, divided by twelve. Not a year-end
  snapshot. Each dwelling, building, plant or location counts as one consumer.

| # | column | definition |
|---|---|---|
| 20 | `Residential Sales [VL]` | Volume sold to residential customers, Mcf. **Residential** is EIA's sector of living quarters for private households, including mobile homes and apartment buildings, excluding institutional living quarters. |
| 21 | `Residential Sales [CS]` | Gross revenue from residential sales, dollars. |
| 22 | `Residential Sales [CT]` | Average number of residential consumers during the year. |
| 23 | `Commercial Sales [VL]` | Volume sold to commercial customers, Mcf. **Commercial** covers service-providing facilities of businesses, government at all levels, and other private and public organisations — including hotels, restaurants, retail, schools and universities, churches and hospitals. Size does not affect the class: a large commercial operation stays commercial. |
| 24 | `Commercial Sales [CS]` | Gross revenue from commercial sales, dollars. |
| 25 | `Commercial Sales [CT]` | Average number of commercial consumers during the year. |
| 26 | `Industrial Sales [VL]` | Volume sold to industrial customers, Mcf. **Industrial** is manufacturing, mining including oil and gas extraction, and agriculture, forestry and fisheries. Size does not affect the class: a small industrial operation stays industrial. |
| 27 | `Industrial Sales [CS]` | Gross revenue from industrial sales, dollars. |
| 28 | `Industrial Sales [CT]` | Average number of industrial consumers during the year. |
| 29 | `Electric Power Sales [VL]` | Volume sold for electric power generation, Mcf — regulated electric utilities and non-regulated generators. |
| 30 | `Electric Power Sales [CS]` | Gross revenue from electric power sales, dollars. |
| 31 | `Electric Power Sales [CT]` | Average number of electric power consumers during the year. Typically a handful of plants. |
| 32 | `Vehicle Fuel Sales [VL]` | Volume sold as vehicle fuel (CNG/LNG fuelling), Mcf. |
| 33 | `Vehicle Fuel Sales [CS]` | Gross revenue from vehicle fuel sales, dollars. |
| 34 | `Vehicle Fuel Sales [CT]` | Average number of vehicle fuel consumers during the year. |
| 35 | `Total Supply [VL]` | Form Part 4. Total volume of natural and supplemental gas **physically produced or received** into the company's storage, transportation or distribution facilities in the report state, on a physical-possession basis regardless of ownership. Sums production, storage withdrawals, receipts at state lines and US borders, city-gate receipts (both gas purchased for sale and gas received for third-party transport), other receipts, and supplemental gaseous fuels. Mcf. |
| 36 | `Total Disposition [VL]` | Form Part 6. Total volume that **left** the system in the report state. Sums deliveries to end-use consumers of gas the company owns (sales) and of gas it does not own (transportation), storage injections both underground and LNG, volumes delivered across state lines or US borders, lease use, gas returned to reservoirs for repressuring or reinjection, losses from leaks, and other disposition. Should broadly balance `Total Supply [VL]`. Mcf. |
| 37 | `Losses from Leaks [VL]` | Form Part 6 line 17. Known loss volumes from leaks, damage, accidents, migration and blow-down within the report state, including losses encountered as a natural consequence of distribution activity. May be the filer's best estimate. Mcf. |

Definitions above are from the EIA-176 instructions (OMB 1905-0175, expiring 07/31/2027),
Parts 4 and 6.

**Sales vs transportation.** These are *sales* columns. In states with retail
choice, a utility may only deliver gas a customer bought from a competitive
marketer; that appears in EIA's transportation lines, which are not in this file.
A utility with customers but near-zero sales volume is likely delivery-only.

## PHMSA system totals (38–39)

Form Part B.1, "SYSTEM TOTAL". End-of-year condition, not an annual flow.

| # | column | definition |
|---|---|---|
| 38 | `MMILES_TOTAL` | Total miles of main in the system at end of year. |
| 39 | `NUM_SRVCS_TOTAL` | Total number of service lines in the system at end of year. |

## PHMSA mains by material (40–53)

Form Part B.1 (general) and B.2 (by diameter). Miles of main. Steel is split
four ways because coating and cathodic protection are what determine corrosion
risk: bare and unprotected pipe is the oldest and highest-risk category.

**Which columns add up.** Columns 40–45 and 47–50 — the Part B.1 general table —
reconcile to `MMILES_TOTAL` in **116 of 116** NH filings, median error 0.00%.
That is the set to sum. Column 46 is excluded from it, and the parallel Part B.2
by-diameter totals reconcile in only 11 of 116, because those tables go unfilled
in earlier years.

| # | column | definition |
|---|---|---|
| 40 | `MMILES_STEEL_UNP_BARE` | Steel, unprotected and bare — no coating, no cathodic protection. |
| 41 | `MMILES_STEEL_UNP_COATED` | Steel, coated but not cathodically protected. |
| 42 | `MMILES_STEEL_CP_BARE` | Steel, cathodically protected but bare. |
| 43 | `MMILES_STEEL_CP_COATED` | Steel, cathodically protected and coated — the fully protected category. |
| 44 | `MMILES_PLASTIC` | Plastic, all subtypes. |
| 45 | `MMILES_CI` | Cast/wrought iron, Part B.1 general table. |
| 46 | `MMILES_CI_WR_TOTAL` | Cast/wrought iron, row total of the Part B.2 by-diameter table. **The same quantity as column 45, reported in a second place on the form — do not add them together, and do not add this to the B.1 set.** Carried only as a cross-check: the two agree in 127 of 132 NH filings, the rest differing by about a mile from filer rounding between the tables. |
| 47 | `MMILES_RCI` | **Reconditioned cast iron** — cast iron main rehabilitated in place (typically cement- or epoxy-lined, or internally sealed) rather than replaced. A distinct Part B.1 category. **PHMSA only collects it from 2015**, so 1997–2014 is blank rather than zero. Rare and small where present: 142 operator-state-years above zero, 17 operators, never more than 10.5 miles — concentrated in the old eastern cast-iron systems (PSE&G, Philadelphia Gas Works, Boston Gas, Con Ed, Washington Gas). |
| 48 | `MMILES_DI` | Ductile iron. |
| 49 | `MMILES_CU` | Copper. |
| 50 | `MMILES_OTHER` | Other material, Part B.1 general table, with the type described in a free-text form field not carried here. Distinct from `MMILES_OTHER_TOTAL` (the B.2 by-diameter total, not included here) — the two genuinely disagree, and only this one belongs to the family that reconciles to the system total. |
| 51 | `MMILES_PE_TOTAL` | Polyethylene — in practice almost all modern plastic main. |
| 52 | `MMILES_ABS_TOTAL` | ABS (acrylonitrile butadiene styrene), an early plastic no longer installed. |
| 53 | `MMILES_OTH_PLSTC_TOTAL` | Other plastic. |

**Plastic subtypes and PVC.** The form collects PVC as a fourth plastic subtype,
but PHMSA's published data files carry no PVC column, so it cannot appear here.
In NH, `PE + ABS + OTHER PLASTIC` matches `MMILES_PLASTIC` within 2% in 120 of
121 filings, so little is unaccounted for — but the subtypes are not guaranteed
to sum to the total.

## PHMSA services by material (54–64)

Form Part B.3. Same material categories as above, counted as **numbers of
service lines** rather than miles. Column 60 carries the same caution as 46.

| # | column | definition |
|---|---|---|
| 54 | `NUM_SRVS_STEEL_UNP_BARE` | Service lines of unprotected bare steel. |
| 55 | `NUM_SRVS_STEEL_UNP_COATED` | Service lines of coated but unprotected steel. |
| 56 | `NUM_SRVS_STEEL_CP_BARE` | Service lines of cathodically protected bare steel. |
| 57 | `NUM_SRVS_STEEL_CP_COATED` | Service lines of cathodically protected, coated steel. |
| 58 | `NUM_SRVS_PLASTIC` | Plastic service lines. |
| 59 | `NUM_SRVS_CI` | Cast/wrought iron service lines (Part B.1 general). |
| 60 | `NUM_SRVS_CI_WR_TOTAL` | Cast/wrought iron service lines, by-diameter table total. **Same quantity as 59 — do not add.** |
| 61 | `NUM_SRVS_RCI` | Reconditioned cast iron service lines. |
| 62 | `NUM_SRVS_DI` | Ductile iron service lines. |
| 63 | `NUM_SRVS_CU` | Copper service lines. |
| 64 | `NUM_SRVS_OTHER` | Service lines of other material. |

## PHMSA installation era (65–88)

Form Part B.4, "Miles of main and number of services by decade of installation".
This is what makes the file useful for replacement-programme and vintage
analysis: pre-1970 mileage is the usual proxy for the oldest, highest-risk pipe.

| # | column | definition |
|---|---|---|
| 65 | `MMILES_BY_DCD_UNK` | Miles of main whose decade of installation the operator does not know. Often large for older systems, and it belongs in the denominator of any vintage share. |
| 66 | `MMILES_BY_DCD_PRE1940` | Miles installed before 1940. |
| 67–75 | `MMILES_BY_DCD_1940_TO_1949` … `MMILES_BY_DCD_2020_TO_2029` | Miles installed in each decade: 1940s, 1950s, 1960s, 1970s, 1980s, 1990s, 2000s, 2010s, 2020s. |
| 76 | `MMILES_BY_DCD_TOTAL` | System total for the decade table **as filed**. Not recomputed here, so it need not exactly equal the sum of 65–75. |
| 77 | `NUM_SRVS_BY_DCD_UNK` | Service lines of unknown installation decade. |
| 78 | `NUM_SRVS_BY_DCD_PRE1940` | Service lines installed before 1940. |
| 79–87 | `NUM_SRVS_BY_DCD_1940_TO_1949` … `NUM_SRVS_BY_DCD_2020_TO_2029` | Service lines installed in each decade. |
| 88 | `NUM_SRVS_BY_DCD_TOTAL` | Service-line total for the decade table as filed. |

## PHMSA damage prevention and unaccounted gas (89–91)

| # | column | definition |
|---|---|---|
| 89 | `PARTDTOTDAMAGES` | Form Part D item 1, **total excavation damages** — third-party dig-ins that damaged the operator's facilities. Called `EXCAV_DAMAGES` in PHMSA's files before the 2024 form revision; both are carried here under this name. |
| 90 | `PARTDTOTTICKETS` | Form Part D item 2, **number of excavation tickets** — locate requests received through the One-Call/811 centre. Damages ÷ tickets is the standard damage-rate denominator. Called `EXCAV_TICKETS` before 2024. |
| 91 | `PERCENT_UNACC_GAS` | Form Part G, percent of unaccounted-for gas. Per the form: `[(purchased gas + produced gas) − (customer use + company use + adjustments)] ÷ (customer use + company use + adjustments) × 100`. **Reported for the 12 months ending 30 June of the reporting year, not the calendar year.** Can be negative when more gas is accounted for than received, usually a meter-reading and storage timing effect. |

---

## Reading notes

**Blank is not zero.** A blank means the operator did not report that field —
often because the form did not collect it that year. PHMSA has no way to record
"not reported", so a literal `0` cannot be distinguished from a missing figure
and neither source has been second-guessed here.

**Coverage by era.** Part D excavation fields begin around 2010; `ownership`
and reconditioned cast iron both begin in **2015**, when the form revision that
added them took effect; and the 2020s decade bucket only fills as that decade
progresses. Pre-2004 filings use an older, sparser form. A column that is blank
across a utility's early years usually means the form did not yet ask.

**Blank EIA columns on `additional` rows.** Deliberate. The EIA figures belong
to the utility-year, not to any one PHMSA filing, so they sit on the `primary`
row only. `eia_id` and the names are still populated so the row is legibly the
same utility.

**Joining across years.** Join on `eia_id` and `phmsa_id`, never on names —
both change spelling, and a merger changes them entirely while the ids persist.
