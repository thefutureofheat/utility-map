# Data dictionary — `utility_data_<STATE>.csv`

Downloaded from the Data Download section of the
[gas utility map](https://thefutureofheat.github.io/utility-map/).

One row per **(utility, year, PHMSA filing)**. 79 columns.

Sources are two federal collections, joined but never blended:

| | source | authority |
|---|---|---|
| `eia_*`, bracketed columns | EIA **Form 176**, "Annual Report of Natural and Supplemental Gas Supply and Disposition" | EIA |
| `MMILES_*`, `NUM_SRVS_*`, `PARTD*`, `PERCENT_UNACC_GAS` | PHMSA **Form F 7100.1-1**, "Annual Report for Gas Distribution System" | PHMSA, 49 CFR Part 191 |

## The one thing to know before using this file

**Every value is as filed.** Where the map's click panel fills a utility's early
years from the companies it absorbed, this file does not. Gaps stay gaps, and a
utility's series starts the year it first filed under its own name.

So panel and CSV totals can legitimately differ. Example: Liberty Utilities dba
EnergyNorth shows EIA data for 1997–2024 in the panel and **2012–2024** here,
because 1997–2011 belongs to EnergyNorth Nat Gas Inc, which appears in this file
as its own utility. Nothing has been apportioned across states either — a
combined multi-state filing is kept whole and flagged in `combined_report`.

**Do not sum a column without filtering `phmsa_report_role = primary`.** An
operator can file several reports for one state and year, and all of them are
here. Summing every row double-counts. See rows 11–12.

---

## Identity and provenance (1–16)

Constructed for this file, not taken from either source form.

| # | column | definition |
|---|---|---|
| 1 | `eia_id` | EIA Form 176 respondent id, with a two-letter state suffix (`17675803NH`). Blank on PHMSA-only rows. |
| 2 | `phmsa_id` | PHMSA `OPERATOR_ID`, the operator's federal id. Blank on EIA-only rows. |
| 3 | `phmsa_stop` | PHMSA `STOP` — the state the filing covers. Normally one two-letter code; **two or more codes means one report covering several states** (see `combined_report`). |
| 4 | `state` | The state this file was generated for. Constant within a file. |
| 5 | `year` | Calendar year the data describes: PHMSA `REPORT_YEAR`, EIA `YEAR`. |
| 6 | `operator` | Display name. `eia_name` when present, otherwise `phmsa_name`. Convenience only — join on the ids. |
| 7 | `eia_name` | Name as filed with EIA, taken from the utility's most recent filing, so it is stable across the series rather than changing mid-history. |
| 8 | `phmsa_name` | Name as filed with PHMSA, same rule. The two often differ in spelling and sometimes in corporate identity. |
| 9 | `ownership` | PHMSA `OPERATOR_TYPE`, a fixed vocabulary the operator selects for itself — Investor Owned, Municipal, Cooperative, Private and similar. Preferred over the territory shapefile's ownership field, which mislabels non-municipal utilities as municipal. **PHMSA only collects this from 2015**, so earlier years are blank. Blank on EIA-only rows. |
| 10 | `match_status` | `matched` — the same utility was identified in both sources. `eia_only` — files EIA 176 with no PHMSA gas-distribution counterpart, typically an interstate pipeline, a marketer, or a balancing entry. `phmsa_only` — files with PHMSA but not identified in EIA 176. |
| 11 | `phmsa_report_role` | `primary` — the filing the map treats as the year's figures: the most complete one, then the largest. `additional` — every other filing for that operator, state and year, kept so nothing is dropped. Blank on EIA-only rows. |
| 12 | `n_phmsa_reports` | How many PHMSA filings exist for that operator-state-year. `1` for most. Where it is greater, that many rows appear, one `primary` and the rest `additional`. |
| 13 | `combined_report` | `yes` when `phmsa_stop` names more than one state, i.e. the figures cover a multi-state system and are **not** specific to this state. Not apportioned — deliberately. |
| 14 | `on_map` | `yes` when this utility has a territory polygon on the map. `no` is common and not an error: nationally 176 utilities matched in both sources have no polygon. |
| 15 | `territory_name` | Name of the map polygon, where there is one. Blank when `on_map = no`. |
| 16 | `fid` | The map's stable feature id: `eia:<eia_id>`, or `geo:<territory>|<state>` for polygons carrying no filer id. Use it to tie a row back to a shape. |

## EIA Form 176 (17–25)

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
| 17 | `Residential Sales [VL]` | Volume sold to residential customers, Mcf. **Residential** is EIA's sector of living quarters for private households, including mobile homes and apartment buildings, excluding institutional living quarters. |
| 18 | `Residential Sales [CS]` | Gross revenue from residential sales, dollars. |
| 19 | `Residential Sales [CT]` | Average number of residential consumers during the year. |
| 20 | `Commercial Sales [VL]` | Volume sold to commercial customers, Mcf. **Commercial** covers service-providing facilities of businesses, government at all levels, and other private and public organisations — including hotels, restaurants, retail, schools and universities, churches and hospitals. Size does not affect the class: a large commercial operation stays commercial. |
| 21 | `Commercial Sales [CS]` | Gross revenue from commercial sales, dollars. |
| 22 | `Commercial Sales [CT]` | Average number of commercial consumers during the year. |
| 23 | `Total Supply [VL]` | Form Part 4. Total volume of natural and supplemental gas **physically produced or received** into the company's storage, transportation or distribution facilities in the report state, on a physical-possession basis regardless of ownership. Sums production, storage withdrawals, receipts at state lines and US borders, city-gate receipts (both gas purchased for sale and gas received for third-party transport), other receipts, and supplemental gaseous fuels. Mcf. |
| 24 | `Total Disposition [VL]` | Form Part 6. Total volume that **left** the system in the report state. Sums deliveries to end-use consumers of gas the company owns (sales) and of gas it does not own (transportation), storage injections both underground and LNG, volumes delivered across state lines or US borders, lease use, gas returned to reservoirs for repressuring or reinjection, losses from leaks, and other disposition. Should broadly balance `Total Supply [VL]`. Mcf. |
| 25 | `Losses from Leaks [VL]` | Form Part 6 line 17. Known loss volumes from leaks, damage, accidents, migration and blow-down within the report state, including losses encountered as a natural consequence of distribution activity. May be the filer's best estimate. Mcf. |

Definitions above are from the EIA-176 instructions (OMB 1905-0175, expiring 07/31/2027),
Parts 4 and 6.

**Sales vs transportation.** These are *sales* columns. In states with retail
choice, a utility may only deliver gas a customer bought from a competitive
marketer; that appears in EIA's transportation lines, which are not in this file.
A utility with customers but near-zero sales volume is likely delivery-only.

## PHMSA system totals (26–27)

Form Part B.1, "SYSTEM TOTAL". End-of-year condition, not an annual flow.

| # | column | definition |
|---|---|---|
| 26 | `MMILES_TOTAL` | Total miles of main in the system at end of year. |
| 27 | `NUM_SRVCS_TOTAL` | Total number of service lines in the system at end of year. |

## PHMSA mains by material (28–41)

Form Part B.1 (general) and B.2 (by diameter). Miles of main. Steel is split
four ways because coating and cathodic protection are what determine corrosion
risk: bare and unprotected pipe is the oldest and highest-risk category.

**Which columns add up.** Columns 28–33 and 35–38 — the Part B.1 general table —
reconcile to `MMILES_TOTAL` in **116 of 116** NH filings, median error 0.00%.
That is the set to sum. Column 34 is excluded from it, and the parallel Part B.2
by-diameter totals reconcile in only 11 of 116, because those tables go unfilled
in earlier years.

| # | column | definition |
|---|---|---|
| 28 | `MMILES_STEEL_UNP_BARE` | Steel, unprotected and bare — no coating, no cathodic protection. |
| 29 | `MMILES_STEEL_UNP_COATED` | Steel, coated but not cathodically protected. |
| 30 | `MMILES_STEEL_CP_BARE` | Steel, cathodically protected but bare. |
| 31 | `MMILES_STEEL_CP_COATED` | Steel, cathodically protected and coated — the fully protected category. |
| 32 | `MMILES_PLASTIC` | Plastic, all subtypes. |
| 33 | `MMILES_CI` | Cast/wrought iron, Part B.1 general table. |
| 34 | `MMILES_CI_WR_TOTAL` | Cast/wrought iron, row total of the Part B.2 by-diameter table. **The same quantity as column 33, reported in a second place on the form — do not add them together, and do not add this to the B.1 set.** Carried only as a cross-check: the two agree in 127 of 132 NH filings, the rest differing by about a mile from filer rounding between the tables. |
| 35 | `MMILES_RCI` | **Reconditioned cast iron** — cast iron main rehabilitated in place (typically cement- or epoxy-lined, or internally sealed) rather than replaced. A distinct Part B.1 category. **PHMSA only collects it from 2015**, so 1997–2014 is blank rather than zero. Rare and small where present: 142 operator-state-years above zero, 17 operators, never more than 10.5 miles — concentrated in the old eastern cast-iron systems (PSE&G, Philadelphia Gas Works, Boston Gas, Con Ed, Washington Gas). |
| 36 | `MMILES_DI` | Ductile iron. |
| 37 | `MMILES_CU` | Copper. |
| 38 | `MMILES_OTHER` | Other material, Part B.1 general table, with the type described in a free-text form field not carried here. Distinct from `MMILES_OTHER_TOTAL` (the B.2 by-diameter total, not included here) — the two genuinely disagree, and only this one belongs to the family that reconciles to the system total. |
| 39 | `MMILES_PE_TOTAL` | Polyethylene — in practice almost all modern plastic main. |
| 40 | `MMILES_ABS_TOTAL` | ABS (acrylonitrile butadiene styrene), an early plastic no longer installed. |
| 41 | `MMILES_OTH_PLSTC_TOTAL` | Other plastic. |

**Plastic subtypes and PVC.** The form collects PVC as a fourth plastic subtype,
but PHMSA's published data files carry no PVC column, so it cannot appear here.
In NH, `PE + ABS + OTHER PLASTIC` matches `MMILES_PLASTIC` within 2% in 120 of
121 filings, so little is unaccounted for — but the subtypes are not guaranteed
to sum to the total.

## PHMSA services by material (42–52)

Form Part B.3. Same material categories as above, counted as **numbers of
service lines** rather than miles. Column 48 carries the same caution as 34.

| # | column | definition |
|---|---|---|
| 42 | `NUM_SRVS_STEEL_UNP_BARE` | Service lines of unprotected bare steel. |
| 43 | `NUM_SRVS_STEEL_UNP_COATED` | Service lines of coated but unprotected steel. |
| 44 | `NUM_SRVS_STEEL_CP_BARE` | Service lines of cathodically protected bare steel. |
| 45 | `NUM_SRVS_STEEL_CP_COATED` | Service lines of cathodically protected, coated steel. |
| 46 | `NUM_SRVS_PLASTIC` | Plastic service lines. |
| 47 | `NUM_SRVS_CI` | Cast/wrought iron service lines (Part B.1 general). |
| 48 | `NUM_SRVS_CI_WR_TOTAL` | Cast/wrought iron service lines, by-diameter table total. **Same quantity as 47 — do not add.** |
| 49 | `NUM_SRVS_RCI` | Reconditioned cast iron service lines. |
| 50 | `NUM_SRVS_DI` | Ductile iron service lines. |
| 51 | `NUM_SRVS_CU` | Copper service lines. |
| 52 | `NUM_SRVS_OTHER` | Service lines of other material. |

## PHMSA installation era (53–76)

Form Part B.4, "Miles of main and number of services by decade of installation".
This is what makes the file useful for replacement-programme and vintage
analysis: pre-1970 mileage is the usual proxy for the oldest, highest-risk pipe.

| # | column | definition |
|---|---|---|
| 53 | `MMILES_BY_DCD_UNK` | Miles of main whose decade of installation the operator does not know. Often large for older systems, and it belongs in the denominator of any vintage share. |
| 54 | `MMILES_BY_DCD_PRE1940` | Miles installed before 1940. |
| 55–63 | `MMILES_BY_DCD_1940_TO_1949` … `MMILES_BY_DCD_2020_TO_2029` | Miles installed in each decade: 1940s, 1950s, 1960s, 1970s, 1980s, 1990s, 2000s, 2010s, 2020s. |
| 64 | `MMILES_BY_DCD_TOTAL` | System total for the decade table **as filed**. Not recomputed here, so it need not exactly equal the sum of 53–63. |
| 65 | `NUM_SRVS_BY_DCD_UNK` | Service lines of unknown installation decade. |
| 66 | `NUM_SRVS_BY_DCD_PRE1940` | Service lines installed before 1940. |
| 67–75 | `NUM_SRVS_BY_DCD_1940_TO_1949` … `NUM_SRVS_BY_DCD_2020_TO_2029` | Service lines installed in each decade. |
| 76 | `NUM_SRVS_BY_DCD_TOTAL` | Service-line total for the decade table as filed. |

## PHMSA damage prevention and unaccounted gas (77–79)

| # | column | definition |
|---|---|---|
| 77 | `PARTDTOTDAMAGES` | Form Part D item 1, **total excavation damages** — third-party dig-ins that damaged the operator's facilities. Called `EXCAV_DAMAGES` in PHMSA's files before the 2024 form revision; both are carried here under this name. |
| 78 | `PARTDTOTTICKETS` | Form Part D item 2, **number of excavation tickets** — locate requests received through the One-Call/811 centre. Damages ÷ tickets is the standard damage-rate denominator. Called `EXCAV_TICKETS` before 2024. |
| 79 | `PERCENT_UNACC_GAS` | Form Part G, percent of unaccounted-for gas. Per the form: `[(purchased gas + produced gas) − (customer use + company use + adjustments)] ÷ (customer use + company use + adjustments) × 100`. **Reported for the 12 months ending 30 June of the reporting year, not the calendar year.** Can be negative when more gas is accounted for than received, usually a meter-reading and storage timing effect. |

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
