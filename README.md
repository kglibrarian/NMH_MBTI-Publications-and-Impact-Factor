# MBTI Publications & Journal Impact Factor

## What this project does

This project builds the "MBTI Publications by Year and Average Impact Factor" chart from a spreadsheet of awarded grants (provided by Sarah Carter, Feinberg research administration).

1. **Find publications for each grant.** For each award in the spreadsheet:
   * **NIH awards:** all publications NIH RePORTER links to the grant's core project number (e.g. `5R01NS116190-05` → `R01NS116190`). Journal, ISSN and year come from PubMed.
   * **Non-NIH awards:** publications whose funding acknowledgement cites the award number, found through the Scopus `FUND-NO` field.
2. **Attach a Journal Impact Factor (JIF)** to every publication, using JIFs exported from InCites.
3. **De-duplicate and chart.** Count publications by year, split into JIF < 10 and ≥ 10, and plot the average JIF.

Two notebooks do the work:

| Notebook | Purpose |
|---|---|
| `grant_publications_jif.ipynb` | Main pipeline. **Part 1** finds publications and lists their journal ISSNs. **Part 2** attaches JIFs and builds the chart. |
| `consolidate_incites.ipynb` | Combines the InCites exports into one CSV and lists ISSNs that still need to be searched. |

## Folder layout

```
project folder/
├── grant_publications_jif.ipynb
├── consolidate_incites.ipynb
├── config.py                        ← API keys and email (keep private)
├── data/
│   ├── Publication List for Impact Factor 9-29-26.xlsx   ← Sarah's grants spreadsheet
│   └── incites/                     ← put every InCites export here
├── output/                          ← created by the notebooks
└── api_cache/                       ← saved API responses (created automatically)
```

Run both notebooks from this project folder.

## Before you start

* `config.py` holds `SCOPUS_API_KEY`, `SCOPUS_INSTTOKEN`, `NCBI_API_KEY` and `CONTACT_EMAIL`.
* Be on the **Northwestern network or VPN**, or Scopus will reject the API key. The exception is if `SCOPUS_INSTTOKEN` is filled in.
* Check the settings in the first cell of `grant_publications_jif.ipynb`. In particular, `GRANTS_FILE` should point to the current spreadsheet.

## Step-by-step

### Step 1: Run Part 1 of `grant_publications_jif.ipynb`

Run every cell from the top down to the **"Part 2 — Attach Journal Impact Factors"** heading. Stop there.

This queries RePORTER, PubMed and Scopus. The first run can take a while; re-runs are fast because responses are saved in `api_cache/`.

It produces `output/02_issn_list.txt` and `output/02_journal_issns_for_jcr.csv`, the journals that need JIFs.

### Step 2: Search the ISSNs in InCites

1. Open **InCites Benchmarking & Analytics** and go to **Publication Sources**.
2. Use the **ISSN** filter and paste ISSNs **separated by commas**, **no more than 100 at a time**.
   * **First round:** take the ISSNs from `output/02_issn_list.txt`.
   * **Later rounds:** take them from the list `consolidate_incites.ipynb` prints (see Step 3).
3. Set the **time period to 1980 to present**, so you get as many JIF years as possible.
4. Select the **Trends** box, then export to Excel.
5. Save each export (`.xlsx`) into **`data/incites/`**. File names don't matter.

Repeat for each batch of 100 ISSNs.

### Step 3: Run `consolidate_incites.ipynb`

Run all cells. It will:

* combine every workbook in `data/incites/` into **`output/incites_consolidated.csv`**. Duplicate journal-years from overlapping exports are removed automatically.
* compare against your ISSN list and print each journal's status:
  * **found:** at least one of the journal's ISSNs is in the InCites data.
  * **searched, not found:** the ISSN was in an InCites search (read from each export's *Result Summary* sheet), but no data came back. The journal probably has no JIF; one more search is worth a try.
  * **not searched yet:** the ISSN isn't in any export's search filter, so a batch was probably missed.
* print the ISSNs still to search, comma-separated, ready to paste into InCites. They're also saved in `output/issns_to_research.txt`.

**Repeat Steps 2 and 3** until the only journals left are ones InCites doesn't cover. Those are usually preprint servers, conference proceedings and very new journals.

### Step 4: Run Part 2 of `grant_publications_jif.ipynb`

The first cell must point to the consolidated file:

```python
JIF_FILES = ["output/incites_consolidated.csv"]
JIF_COLUMN_OVERRIDES = {"year": "Publication Year", "jif": "Journal Impact Factor"}
```

If the kernel was restarted, re-run the first two cells (configuration and utilities). Then run everything from the **Part 2** heading to the end.

## Current settings and why

| Setting | Value | Why |
|---|---|---|
| `REQUIRE_PI_AUTHOR_ON_NIH_SUBAWARDS` | `True` | For NIH subawards, RePORTER returns papers from every site on the prime grant. This keeps only papers with a Northwestern PI from the award as an author. When it was `False`, publication counts rose far above previous years' charts. |
| `SCOPUS_REQUIRE_NU_OR_PI` | `False` | Any paper citing the award number in Scopus counts. Bare numeric award numbers (e.g. NSF `1837999`) must also match the sponsor name. |
| `JIF_YEAR_OFFSET` | `-1` | Each publication gets its journal's JIF from the **year before** publication. |
| `JIF_FALLBACK_TO_NEAREST_YEAR` | `True` | Only used when the wanted JIF year hasn't been released yet; the latest JCR year is used instead. Earlier gaps stay blank. |
| `CHART_YEARS` | 2021–2025 | Five complete years with released JIFs. |
| `SHOW_DELTA_BOX` | `False` | The % change box is off; see the JIF note below. |

## Outputs worth checking

| File | Why look at it |
|---|---|
| `01_awards.csv` | One row per award: how it was searched (RePORTER, Scopus, or **not searched**) and how many publications it got. Check awards with 0 publications, and awards with surprisingly many. |
| `01_excluded_matches.csv` | Papers dropped by a rule (e.g. subaward without a Northwestern author), with the reason. |
| `01_scopus_query_log.csv` | The exact Scopus query and hit count for every non-NIH award. |
| `02_publications_without_issn.csv` | Matches with no ISSN (preprints, proceedings). These can never get a JIF. |
| `03_publications_by_grant_with_jif.csv` | Every grant × publication pair with its JIF. A paper appears once per grant it cites. `jif_match_status` explains blanks. |
| `04_publications_deduplicated_with_jif.csv` | One row per publication, with all linked grants. **This is what the chart counts.** |
| `05_chart_data.csv` | The numbers behind the chart, including `n_without_jif` (papers left out of the bars and average). Use it to rebuild the chart natively in PowerPoint. |
| `06_publications_by_year_avg_jif.png / .svg` | The chart. |

## Things to know when presenting

* **Pandemic-era JIFs.** JIFs for 2020–2022 were temporarily inflated across the sciences by COVID-19 citations, then returned to typical levels (e.g. *Nature*: 69.5 in 2021, 50.5 in 2023). With prior-year JIFs, publications from 2021–2023 are scored with those inflated values, so the average drops at 2024 for reasons unrelated to MBTI. The chart marks this with a dashed line, asterisks and a footnote. Avoid % changes that span the correction. **JIF percentile** (in the InCites export as `Average JIF Percentile`) is adjusted for year and field and would avoid the problem.
* **Lurie Cancer Center Support Grant (P30CA060553).** It appears in Sarah's spreadsheet under Platanias. RePORTER links hundreds of papers from across the cancer center to it, most without MBTI authors, and it accounts for most of the publications. Confirm with Sarah whether it belongs.
* **Column L of Sarah's spreadsheet** (`Long Title`) holds **grant** titles, not publication titles. Publications come from RePORTER and Scopus, not from her file.
* **Chart counts differ from previous years' slides**, because the publication list is now built from every award in the spreadsheet using RePORTER and Scopus. Ask Sarah how earlier lists were built before comparing year to year.

## Re-running next year

1. Put the new grants spreadsheet in `data/` and update `GRANTS_FILE`.
2. Delete `api_cache/` so RePORTER, PubMed and Scopus are re-queried with current data.
3. Move the last year forward in `CHART_YEARS`, once that year's JIFs are released (JCR comes out each June).
4. Follow Steps 1–4. Existing InCites exports can stay in `data/incites/`, but re-export journals that need the newest JIF year.
