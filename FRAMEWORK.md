---
title: Systematic Research Framework (build specification)
version: 1.0.0
author: "@negvol"
year: 2026
audience: an AI coding agent (Claude Code or similar) and the person directing it
---

# Systematic Research Framework

This file is a **build specification**. An agent reads it end to end and builds a
research workspace: an Obsidian vault and a Python engine. In that workspace a
trading or forecasting idea is tested so that a false positive is as hard to
produce as possible, and a dead idea is closed cleanly, recorded, and never
quietly re-run.

It covers intraday trading, longer-horizon portfolio work, prediction markets, and
any other domain that can be described with the adapter in §14. It assumes
the person starts with nothing: no data, no code, no notes.

The companion paper (`Systematic-Research-Framework-negvol-2026.pdf`) explains
the reasoning behind each rule for a human reader. This file says what to build.

---

## 0. Instructions to the agent

### 0.1 Your role

- You build the workspace, then you operate it. You implement ideas, attack them,
  run the protocol and report verdicts.
- By default **the user supplies ideas and you attack them** (`IDEA_SOURCE = "user"`
  in config). If the user sets `IDEA_SOURCE = "both"`, every idea you propose is
  labelled `origin: agent` and is booked like any other. Never generate ideas by
  sampling popular or published strategies. The prior that those still work is
  close to zero.
- Your job is to find the reason a result is wrong before the market does.
  Spend most of your effort on adversarial review of the idea and of your own
  implementation.

### 0.2 Hard rules for the agent (never broken, even when asked casually)

1. **Never edit the ledger.** `ledger/chain.jsonl` and the blocks in `LEDGER.md`
   are append-only. A correction is a new entry.
2. **Never read TEST or HOLDOUT data outside the protocol.** That includes
   "a quick look", a plot, a describe(), or a notebook cell. The data loader
   enforces this (§7.2). Do not bypass it.
3. **Book every configuration you evaluate**, including ones from exploration,
   notebooks, plots and smoke tests on real data (`registry.log_hidden`).
4. **Never resolve an ambiguity in the direction that favours the current
   result.** Log it as a conflict (`C-xx` in `LEDGER.md`) and ask the user.
5. **Never re-tune after TEST.** A failed TEST is a negative finding and is
   written up as one.
6. **Every test outputs at least one PNG** in the house style (§13). A test
   with no PNG is not finished. Look at the PNG before you report it, and name its
   path in your reply.
7. **Rules change only by amendment** (§5, R0.4). Amendments apply going forward,
   are hashed into the ledger, and never re-score past verdicts.
8. **Report faithfully.** State failures, skipped steps and power limits plainly.
   "Not detected" without a power statement is not a result.

### 0.3 Kickoff interview (ask before building)

Ask the user these questions in one message. Record the answers in
`config/project.yaml` and in `Home.md`. Use the default when the user has no preference.

| # | question | default |
|---|---|---|
| 1 | Which domains? (`intraday`, `portfolio`, `prediction_markets`, `other:<name>`) | the ones the user names |
| 2 | What data do you have or plan to buy? Where is it on disk? | none yet; build adapters for free sources (§6.6) |
| 3 | Tier dates per domain (DEV / TEST / HOLDOUT). Has any period already been looked at? | DEV = first 60%, TEST = next 25%, HOLDOUT = last 15% and forward |
| 4 | Venue / broker / exchange you would actually trade on (fees, permitted products) | none: use the reference cost model, label verdicts `no-venue` |
| 5 | Base currency; include tax? | USD; pre-tax, labelled pre-tax |
| 6 | Capital and size constraints | ask; size every suggestion against it |
| 7 | Who supplies ideas? (`user` / `both`) | `user` |
| 8 | Python version and environment manager | Python ≥ 3.11, `venv` + `pip` |
| 9 | Git remote for the workspace (optional) | local git only |

### 0.4 Build phases and definition of done

Build in this order. Do not start a phase until the previous one passes its
tests. After each phase, run `pytest -q`, commit, and append a `build` entry to
the ledger.

| phase | build | done when |
|---|---|---|
| **P0** | Vault skeleton (§2), `.obsidian/` config, templates (§18), `Home.md`, `RULES.md` written from §5 with the user's answers, `config/` | vault opens in Obsidian, templates insert, Home dashboards render (empty) |
| **P1** | `srf/ledger.py`, `srf/registry.py` (§15) | tamper test fails a modified line; registry refuses a registration missing any required field; rules-hash is committed |
| **P2** | Data layer (§6): raw store + manifest, ingest adapters, cleaning pipeline, quality report + PNG, `DATA.md`, dataset notes | re-running ingest is bit-identical; every dataset has a data card and a quality PNG |
| **P3** | Leakage guards (§7): tier guard, as-of joins, causality truncation harness, purged / embargoed CV | TEST read without a freeze is refused; the leaky centred-MA fixture fails the causality test; the trailing one passes |
| **P4** | Statistics (§9) with unit tests against known values | every test in §16.2 passes |
| **P5** | Evaluation engines per domain (§11, §14) with closed-form checks | zero-edge simulations return zero within CI; barrier simulator matches closed forms (§14.1.6) |
| **P6** | Controls and placebos (§10) | the planted-edge control is detected at t ≥ 4; the noise control is not |
| **P7** | Protocol, verdicts, reports, PNG verdict sheet (§8, §12, §13) | the end-to-end acceptance suite (§16.3) passes |
| **P8** | Knowledge store seeded (§4): one concept note per method used by the engine, linked from `Home.md` | every `[[link]]` in `Home.md` resolves |
| **P9** | Hand-over: `README.md` in the workspace with the daily workflow (§8.6) and commands | the user can register and run the bundled example hypothesis end to end |

---

## 1. Principles

1. **The default answer is "no edge".** A result must survive every attempt to
   explain it as noise, leakage, selection, cost or construction before it is
   believed.
2. **Decide before you look.** The hypothesis, trade rule, grid, statistics, minimum
   useful effect (MUE) and kill criteria are written and hashed before any
   evaluation.
3. **Count every look.** Every configuration evaluated on any data is a trial.
   The number of trials sets the bar a result has to clear.
4. **Effect size, not significance.** Verdicts compare a confidence interval
   with the MUE. A p-value alone never kills or confirms anything.
5. **Out-of-sample data is spent once.** TEST is read once per frozen
   configuration. HOLDOUT is read once, and only by a PASS. Forward data is the
   only truly unseen data.
6. **Power before testing.** If the sample cannot detect the MUE, the best
   possible verdict is INCONCLUSIVE, and the study is registered as underpowered.
7. **Costs are part of the hypothesis.** A gross edge smaller than the cost of
   trading it is not an edge.
8. **Inconclusive is not dead, and dead stays dead.** Only KILL-CLEAN is
   terminal. A reopened idea is a new registration that adds to the trial count,
   and it is tested only on data the original never touched.
9. **Everything is reproducible.** Seeds, hashes, data manifests and
   configuration hashes go into the ledger. Any number in a report can be
   regenerated.
10. **Precedence when rules conflict:** data integrity and causality >
    pre-registration > validation design > statistics > reporting.

---

## 2. Workspace layout

```
<workspace>/
├── .obsidian/                  Obsidian config (§3.5)
├── Home.md                     dashboard / map of content (Dataview queries)
├── RULES.md                    the ruleset (§5), cite as R#.#; hashed into the ledger
├── LEDGER.md                   human ledger: amendments, conflicts, one row per verdict
├── DATA.md                     data inventory: every dataset, schema, coverage, hash
├── README.md                   workflow and commands (written in P9)
│
├── hypotheses/                 one note per hypothesis   H-0001 <slug>.md
├── experiments/                one note per run          E-0001 <slug>.md (+ code in code/experiments)
├── findings/                   one note per verdict      F-0001 <slug>.md
├── datasets/                   one data card per dataset D-<slug>.md
├── knowledge/                  the information store (§4)
│   ├── concepts/               methods and ideas         K-<slug>.md
│   ├── sources/                papers, books, posts      S-<author-year-slug>.md
│   ├── venues/                 exchanges, brokers, PM venues; dated rule/fee sheets  V-<slug>.md
│   ├── markets/                instrument / market mechanics notes  M-<slug>.md
│   └── lessons/                method lessons (how a test went wrong), never market claims  L-<slug>.md
├── journal/                    dated research log         YYYY-MM-DD.md
├── templates/                  Obsidian templates (§18)
├── attachments/                every PNG embedded in notes (symlink or copy of results/figures)
│
├── config/
│   ├── project.yaml            kickoff answers (§0.3)
│   ├── tiers.yaml              DEV/TEST/HOLDOUT per domain (hashed)
│   ├── costs/                  dated venue cost sheets (yaml/json)
│   └── domains/                one yaml per domain adapter
├── ledger/
│   ├── chain.jsonl             append-only hash chain (machine record)
│   └── HEAD                    hash of the last line
├── data/
│   ├── raw/                    immutable as received; never edited
│   ├── manifest.jsonl          one line per raw file: path, sha256, source, retrieved_at, licence
│   ├── clean/                  cleaned, typed, timezone-normalised (rebuildable)
│   └── derived/                features, panels, bars (rebuildable, never hand-edited)
├── code/
│   ├── srf/                    the engine package (§15)
│   ├── experiments/            eNNNN_<slug>.py, each exposing run(ctx)
│   └── tests/                  pytest suite (§16)
└── results/
    ├── figures/                PNGs (house style, §13)
    └── tables/                 csv / parquet outputs
```

Notes:
- Notes and code are separated. Obsidian indexes the notes. Set
  `code/`, `data/`, `ledger/` and `results/tables/` as excluded folders (§3.5).
- IDs are permanent and never reused. Hypothesis `H-0001`, experiment `E-0001`,
  finding `F-0001`, amendment `A-01`, conflict `C-01`.
- Raw data is git-ignored. The manifest is committed.

---

## 3. Obsidian conventions

### 3.1 Properties (frontmatter)

Every note starts with YAML properties. Required keys per type:

| type | required properties |
|---|---|
| all | `id`, `type`, `title`, `created`, `tags` |
| `hypothesis` | `domain`, `status`, `verdict`, `origin`, `mechanism`, `mue`, `primary_metric`, `datasets`, `tiers_hash`, `spec_hash`, `ledger_seq`, `family` |
| `experiment` | `hypothesis`, `stage` (`dev`/`controls`/`placebos`/`test`/`holdout`/`incubation`), `config_hash`, `seed`, `ledger_seq`, `figures` |
| `finding` | `hypothesis`, `verdict`, `n_total`, `n_eff`, `mde`, `mue`, `cost_model`, `ledger_seq`, `figures` |
| `dataset` | `source`, `licence`, `coverage_start`, `coverage_end`, `frequency`, `timezone`, `sha256`, `rows`, `quality` |
| `concept` | `domain`, `status` (`stub`/`draft`/`solid`), `sources` |
| `source` | `authors`, `year`, `url`, `doi`, `read` (bool), `relevance` |
| `venue` | `kind` (`exchange`/`broker`/`pm-venue`/`firm`), `sheet_date`, `verified_url`, `sheet_hash` |
| `lesson` | `from` (experiment / finding id), `rule` (R# it informs) |

Enumerations:
- `status` (hypothesis): `draft → registered → dev → frozen → tested → holdout → incubating → deployed`, or `closed`.
- `verdict`: `none`, `PASS`, `INCONCLUSIVE`, `KILL-CLEAN`, `KILL-COST`, `KILL-ERA`; for
  measurement studies `SUPPORTED`, `NOT-SUPPORTED`, `INCONCLUSIVE`.
- `domain`: `intraday`, `portfolio`, `prediction-markets`, or `other/<name>`.

Example:

```yaml
---
id: H-0007
type: hypothesis
title: Opening-range break continuation, equity index futures
created: 2026-01-15
domain: intraday
status: registered
verdict: none
origin: user
mechanism: "none: pattern only"
mue: { metric: net_expectancy_per_trade, value: 1.5, unit: ticks }
primary_metric: per_session_net_pnl
datasets: ["[[D-es-1m]]"]
tiers_hash: 3f1c…
spec_hash: 9ab0…
ledger_seq: 42
family: ["[[H-0003]]"]
tags: [type/hypothesis, domain/intraday, status/registered, verdict/none]
---
```

### 3.2 Tags

Nested tags only, so the tag pane stays a tree:
`type/*`, `domain/*`, `status/*`, `verdict/*`, `stage/*`, `topic/*` (free, for example
`topic/microstructure`), `data/*` (`data/raw`, `data/derived`). The engine keeps
`status/*` and `verdict/*` in step with the properties whenever it writes a note.

### 3.3 Links

- Hypothesis ↔ experiments ↔ finding are linked both ways (`[[E-0012 …]]`).
- Every rule citation in a note is plain text `R6.3`. Every concept is a
  wikilink (`[[K-deflated-sharpe-ratio]]`).
- Figures are embedded with `![[attachments/F-0004_verdict.png]]`.
- `Home.md` links every top-level folder and every MOC.

### 3.4 Dashboards (Dataview)

`Home.md` contains these queries. They need the Dataview community plugin; the
vault still works without it.

````markdown
## Open hypotheses
```dataview
TABLE domain, status, verdict, mue.value AS MUE, ledger_seq AS seq
FROM "hypotheses" WHERE status != "closed" SORT id ASC
```
## Verdicts
```dataview
TABLE hypothesis, verdict, n_total, n_eff, mde, mue, cost_model
FROM "findings" SORT id DESC
```
## Reopen queue (INCONCLUSIVE, KILL-COST, KILL-ERA)
```dataview
LIST FROM "hypotheses" WHERE contains(list("INCONCLUSIVE","KILL-COST","KILL-ERA"), verdict)
```
## Stub concepts
```dataview
LIST FROM "knowledge/concepts" WHERE status = "stub"
```
````

### 3.5 `.obsidian/` config to write

- `app.json`: `{"attachmentFolderPath": "attachments", "newFileLocation": "folder", "newFileFolderPath": "journal", "userIgnoreFilters": ["code/", "data/", "ledger/", "results/tables/"], "alwaysUpdateLinks": true}`
- `core-plugins.json`: enable `file-explorer`, `global-search`, `graph`, `backlink`, `outgoing-link`, `tag-pane`, `properties`, `templates`, `daily-notes`, `outline`.
- `templates.json`: `{"folder": "templates", "dateFormat": "YYYY-MM-DD"}`
- `daily-notes.json`: `{"folder": "journal", "template": "templates/Journal"}`
- `graph.json`: colour groups by tag: `type/hypothesis`, `type/finding`, `type/concept`, `type/dataset`.
- Recommended community plugins: **Dataview**, optionally **Templater**. List them in
  `README.md`. Do not install plugins yourself.

---

## 4. The knowledge store

`knowledge/` holds information, not results. It exists so that every question the user or
the agent answers once is answered for good.

| folder | holds | rule |
|---|---|---|
| `concepts/` | methods, statistics, market concepts (one idea per note) | cites sources; has a "used by" section listing engine modules and hypotheses |
| `sources/` | papers, books, documentation, posts | one note per source: claim, method, data, sample, our critique, tradeability; never paste copyrighted text |
| `venues/` | exchanges, brokers, PM venues, prop firms | dated rule and fee snapshot, link to the official page, sheet hash; re-verify before any decision that uses it |
| `markets/` | instrument mechanics (sessions, rolls, settlement, resolution rules) | facts with a source and a date; "unverified" until checked against the official source |
| `lessons/` | how a test went wrong and the check that now catches it | a lesson is about method; never "X works" or "X doesn't work" |

Rules:
- A **finding** about a market lives in `findings/`, tied to a verdict and a
  ledger sequence number. It never lives in `knowledge/`. This keeps "what we
  believe about the world" separate from "what we know how to do".
- Every note that states a fact carries a `sources:` property or an inline
  citation.
- Seed in P8: one concept note per method in §9 and §10 (for example `K-deflated-sharpe-ratio`,
  `K-pbo-cscv`, `K-purged-cv`, `K-block-bootstrap`, `K-benjamini-hochberg`,
  `K-minimum-detectable-effect`, `K-sprt`, `K-placebo-tests`), and one source note per
  reference in §19.

---

## 5. The ruleset (write this to `RULES.md`)

Write the rules below to `RULES.md` with numbering intact, substitute the
user's kickoff answers where marked ⟨…⟩, and commit its sha256 to the ledger
(`kind: rules`). Every registration records the rules hash it was made under.

### R0 — Scope, precedence, amendments
1. **Scope:** the domains in `config/project.yaml` ⟨domains⟩. A domain is admitted when its
   adapter (§14) exists with a calendar, a cost model, a fill model and tiers.
   Admission depends on these inputs, not on a whitelist. The registry refuses
   a registration only for a missing input, and names it.
2. **Precedence:** data integrity and causality > pre-registration > validation
   design > statistics > reporting.
3. **Ambiguity:** never resolved in the direction that favours the current result. Log
   `C-xx` in `LEDGER.md`, flag it to the user, and wait.
4. **Amendments:** a rule changes only through an amendment block `A-xx` in `LEDGER.md`
   (text, reason, date, which rules), hashed into the chain. It applies to
   registrations after it, never retroactively.

### R1 — Ideas
1. Ideas come from ⟨IDEA_SOURCE⟩. The agent attacks ideas. When it proposes one,
   it is labelled `origin: agent`.
2. Prefer ideas specific to the user's situation (access, contract terms, capital,
   tax wrapper). Nobody else shares those constraints, so few people search there.
3. Useful asks: "try to kill this", "what data would falsify it, and do I have it?",
   "who else has looked at this and why did they stop?", "build and stress-test it".

### R2 — Idea gates (run in order, before any code)
1. **Mechanism (recorded, optional).** Try to finish: "____ must buy/sell ____ at ____
   because ____, regardless of price." If you can't, record `mechanism: "none: <pattern>"`.
   A pattern without a mechanism can be registered and is confirmed the same way.
2. **Observable (drops the idea).** Name the dataset, column and frequency that would
   show the effect. Before answering "no", list every dataset and print the
   candidate schemas (`DATA.md` plus the files themselves). If nothing on disk can
   show it, buy the data or drop the idea. Never proceed on a proxy you hope
   will do. When the mechanism implies a flow, test the flow, not only the
   price.
3. **Searcher density (recorded).** Is it in a textbook, a well-cited paper, or a popular
   post? If yes, the burden of proof is higher: say why it survived being found.
4. **Persistence (recorded).** Why can't it be competed away? Choose one: forced flow
   (strong), compensated risk premium (strong, cyclical), structural / categorical
   barrier (account type, mandate, regulation), too small for institutions (name the
   capacity), or unknown (weak).
5. **MUE and kill criterion (drops the idea if missing).** State the minimum useful
   effect on the primary metric, net of costs, and the kill criterion in effect-size
   terms. The MUE floor is domain-specific (§14). Default floor: net expectancy of at
   least one round-trip cost per trade (gross at least two costs).
6. **Second prediction (when there is a mechanism).** State one independent
   prediction the mechanism makes that a fitted pattern would not (for example the
   flow reverses after it ends). Test it with the main hypothesis.
7. **Start date.** The earliest date the mechanism's preconditions held, or the
   date the rule's inputs first existed. Earlier data is labelled pre-mechanism
   and is reported but not gated.

### R3 — Pre-registration and the ledger
1. Before any evaluation, the hypothesis note and the registry entry hold:
   mechanism (or none), gate answers, falsifiable prediction with its sign, data and
   tiers, trade or measurement rule, parameter grid and its size, primary metric,
   statistics and controls, MUE, kill criteria, power plan, and the `family`
   (earlier work that chose this idea).
2. The trade rule is fixed before looking: entries, exits, stops, sizing, fill
   rules, timeouts and costs.
3. The registry hashes the spec (`spec_hash`) and refuses to evaluate an unregistered
   or changed spec. Changes before evaluation are pre-registration amendments, with
   a reason. Changes after evaluation are new registrations.
4. **Trial counting:** every configuration evaluated on any data adds to
   `N_total` (programme-wide) and to the study's selection family. The same frozen
   configuration re-run on another tier or cost model is not a new trial. Report
   `N_eff` (§9.6) per study and cumulatively.
5. Bugs found before reporting are recorded with their effect on the number.
6. **Reopening:** a new ID, approved by the user. It adds to `N_total`, and it may
   only be tested on data dated after the reopening, or data the original never
   touched.
7. **Verdict classes:** exactly one per hypothesis at any time (§12).

### R4 — Data hygiene
1. Raw data is immutable. Every file is in `data/manifest.jsonl` with its sha256,
   source, retrieval time and licence.
2. Normalise every timestamp to UTC at ingest and keep the venue's local time as a
   derived column. Session logic uses the venue calendar (holidays, early closes,
   DST). Tests cover the DST switch weeks.
3. Every column carries a knowledge time (`available_at`): the moment the value could
   have been known, which is not the moment it describes. Joins are as-of on
   `available_at` (§7.3).
4. Domain hygiene rules apply: continuous futures stitching (§14.1), total
   returns, survivorship and revisions (§14.2), resolution and snapshots (§14.3).
5. Measure the noise floor at your horizon before building any denoising.
6. **Data status:** any period that has been looked at, by anyone, for this
   idea family is not a holdout. Record what has been seen in `DATA.md`
   (`seen_by`). Before using HOLDOUT, compute its MDE. If MDE > MUE, HOLDOUT can kill on a
   clear negative but cannot confirm; confirmation then needs forward incubation.

### R5 — Splits and validation
1. Three tiers by date per domain, fixed in `config/tiers.yaml` before the first
   registration and hashed: **DEV**, **TEST**, **HOLDOUT** (plus forward data).
   The protocol refuses a study whose tiers changed after it was registered.
2. Split on whole clusters (sessions, rebalance periods, events). Purge training rows
   whose label window overlaps a validation fold, and embargo at least the label horizon
   (default 2 clusters) on each side of a tier boundary.
3. Select on DEV only, using purged walk-forward or purged k-fold / CPCV inside DEV.
4. **TEST is read once** per frozen configuration. If it fails, the finding is
   negative. No re-tuning and re-testing.
5. **HOLDOUT is sealed:** read at most once, only by a PASS, and only if the frozen config
   hash matches.
6. **Forward incubation is the confirmatory stage:** paper or minimum size, with a
   pre-registered sequential test (SPRT or CUSUM, §9.10) whose boundaries and maximum
   length are in the ledger before the first observation.
7. **Order of operations** (enforced by `srf/protocol.py`): select on DEV (every config a
   trial) → engine controls → causality checks → placebos on DEV (judged gross) → PBO →
   freeze (config hash, power check) → TEST once → placebos on TEST → statistics →
   cost sweep → verdict → HOLDOUT (PASS only) → incubation.
8. **Hard gates stop the run** (data integrity, causality, engine controls, a
   placebo failure on a powered sample). **Soft gates feed the verdict** and never stop
   a run alone (significance, DSR, PBO, FDR, era sign agreement).

### R6 — Statistics
1. **Primary series:** the domain's cluster-level net P&L (per session, per rebalance
   period, per event), the SUM of that cluster's trades, zero-filled for clusters
   with no trade. Trade-weighted figures with cluster-robust standard errors are
   secondary. Report both and flag any disagreement in sign or significance.
2. One estimator throughout: a stationary block bootstrap over clusters, with the
   block length chosen by Politis–White and floored at ⟨BLOCK_FLOOR⟩ clusters.
3. **One primary multiplicity control per study type:** a single hypothesis
   with a sweep uses DSR ≥ 0.95, with `N_eff` taken over the selection family. A
   screen of many candidates uses BH-FDR at q = 0.10. Hansen's SPA is a reported
   diagnostic. PBO ≤ 0.30 is a soft gate.
4. Ranks are taken globally, never within a cluster. (Ranking within a session
   produces a negative IC on a pure random walk.)
5. **Sign agreement across eras** after the start date is required before
   anything is called an effect. Report effect sizes in native units and in
   sigma.
6. Report effective N: the raw count, the cluster count, and the outcome correlation
   across instruments or markets. Near-duplicates are not independent replications.
7. Claims about tails are distributional: medians with bootstrap CIs, KS / Anderson–Darling,
   and the two tails treated separately, never pooled.
8. **Power first:** compute the MDE at 80% power from the cluster count before TEST is touched.
   If MDE > MUE, extend the data or register as underpowered (best verdict
   INCONCLUSIVE). Floors: ⟨MIN_TRADES⟩ trades and ⟨MIN_CLUSTERS⟩ clusters.
9. No discretionary haircut. The DSR-implied Sharpe is the haircut. Never deflate
   twice.
10. Kill decisions use 90% CIs on the primary metric against the MUE (§12). A p-value alone
    never produces a verdict.

### R7 — Controls and placebos
1. **Engine controls before any real example:** a positive control (a planted
   edge of known size, detected at t ≥ 4) and a negative control (noise, not
   detected). A harness that fails either cannot rule anything in or out.
2. **Every trading hypothesis gets three placebos and must beat all three:** random
   entries matched on time and count; the rule applied at the wrong time (wrong day,
   wrong event, shifted timestamps), which preserves every marginal; and the
   signal sign inverted. Judge placebos on gross results, one-sided at 0.05.
   Report net but never gate on it.
3. **Level studies:** compare against distance-matched random levels, matched per
   type and never pooled.
4. **Discontinuities** are tested as regression discontinuities with placebo cutoffs.
   **Time-asymmetry claims** use a statistic that flips under time reversal,
   tested against reversed surrogates.
5. **Spreads and hedges** use an imposed hedge (dollar- or beta-neutral, rolling), never
   fitted weights. Both legs pay costs.
6. **Path-selected extremes** (the high, the low) need a mirror control.

### R8 — Artefact checklist (none of these shows up in a t-stat)
Check every conditional result for each item before believing it:
1. **Sizing look-ahead:** the size depends on something not known at entry.
2. **A conditioner that is secretly a clock:** a running high or low, session VWAP, or
   cumulative mean starts pinned to price and fans out. Measure the distance from
   the anchor by time of session, or measure the result as excess over the same
   clock time on other days.
3. **A contrast is not a P&L;** **a drift is not a trade.** Both must still clear costs.
4. **Unadjusted stitches** (futures rolls, splits, re-based indices).
5. **Look-ahead by construction:** zigzags, swing points and confirmed pivots exist only
   after confirmation. Timestamp them at confirmation.
6. **Regime labels** need an observable, forecastable referent.
7. **Magnitude without direction:** a level or feature that predicts volatility but
   not sign.
8. **Too good to be true:** add the diagnostic that would expose the construction
   (mass at zero, tick alignment, touch rate against distance, identical fills).
9. **Stop exits:** report their P&L separately; flag the result if the edge disappears
   when stop slippage doubles.
10. **Hidden trials:** anything seen in a notebook, plot or chat counts. Book it.
11. **Survivorship and selection on the outcome:** the universe, the market list or
    the sample was chosen using information from after the decision.

### R9 — Costs, fills, execution
1. Two cost models per domain: **primary** (observed spread + empirical slippage +
   fees from a dated venue sheet) and **stress** (a fixed conservative round trip).
   Decisions use primary. Failing only stress gives KILL-COST.
2. Measure cost in sigma of the move traded, not in currency.
3. Fills: a decision at bar t fills at t+1 crossing the spread; resting limits fill
   only on a trade-through; stops fill at least one tick through; any same-bar
   ambiguity is resolved against the trade; a timeout exits at the mark, one tick
   adverse.
4. **Frequency budget:** with cost c per round trip, daily sigma σ_d, and a fraction k of a
   daily sigma captured at one trade a day, break-even N_max = (kσ_d / c)² and the best
   net N_opt = N_max / 4, assuming edge per trade scales as σ/√N. Where possible,
   estimate the decay empirically. Report the field `edge_scaling`.
5. Keep grids small. A larger grid finds more noise, and the DSR bar rises to
   swallow it.

### R10 — Simulation
1. Validate every simulator against closed forms before using it on a strategy.
   Closed forms are unit tests of the simulator, never evidence about a strategy.
2. Use common random numbers for paired comparisons, and demean exactly when comparing
   at zero drift.
3. When a policy is chosen on a grid, re-simulate it on a fresh seed before
   reporting (winner's curse).

### R11 — Published research
1. Score a paper before replicating it: data, sample, costs, multiple testing,
   out-of-sample period, and tradeability at the user's size.
2. A model evaluated on its own data-generating process is not evidence about
   markets.

### R12 — Reporting
1. Every report states: the weighting, clustering, cost model, fill rules, `N_total`,
   `N_eff`, MDE, MUE, the verdict class, which cost model drove the decision, the start
   date, and the path of at least one PNG. `srf/report.py` refuses a report without them.
2. Explain failures in the protocol's own terms: negative net with positive gross means
   real but smaller than costs. PBO > 0.3 means the parameter choice carries
   too little information. DSR < 0.95 with a good p-value means too much
   searching for the evidence available. A failed wrong-time placebo means
   there is no timing information.
3. A null result carries a power statement: the effect size the test could have
   detected.

### R13 — Operations
1. The chain and `LEDGER.md` are verified on every run. A mismatch stops everything.
2. Chunked or parallel builds must be bit-identical to sequential ones.
3. After stopping a parallel job, kill the parent first, then the children, so the
   parent does not relaunch stale work.
4. Secrets live only in `.env` (git-ignored). Never put them in notes, logs or figures.

---

## 6. Data layer: aggregation and cleaning

### 6.1 Pipeline

```
source → raw/ (immutable, hashed, manifest) → clean/ (typed, UTC, deduped, flagged) → derived/ (bars, panels, features)
```

Each step is a pure function of its inputs. Each output records the hashes of its inputs
and the code version in a sidecar `*.meta.json`. Rebuilding from raw must be
bit-identical.

### 6.2 Ingest adapter contract

```python
class Source(Protocol):
    name: str                       # "yahoo_daily", "databento_tbbo", "kalshi_markets", ...
    licence: str                    # terms, redistribution allowed?
    def fetch(self, start, end, **kw) -> list[Path]: ...       # writes into data/raw/<name>/, appends manifest
    def schema(self) -> dict[str, str]: ...                     # expected columns and dtypes
    def to_clean(self, raw_paths) -> pd.DataFrame: ...          # typed, UTC, with available_at
```

### 6.3 Cleaning checklist (each step logged with counts before and after)

1. Schema check: columns, dtypes, units. Refuse unknown columns silently changing.
2. Timezone: parse, localise, convert to UTC; keep `ts_local`. Assert DST weeks.
3. Duplicates: exact duplicates dropped; same-timestamp distinct rows kept (multiple fills
   of one sweep are real).
4. Gaps: compare against the venue calendar; flag missing sessions and partial
   sessions; never forward-fill prices across a gap for return calculations.
5. Outliers: flag (do not delete) prints outside a robust band (for example 10 × MAD
   of returns, or outside the concurrent bid/ask); log every removal with reason.
6. Bad values: non-positive prices, crossed quotes, zero-volume bars marked, wrapped
   integers (unsigned overflow), sentinel values (`-999`, `0` for missing).
7. Corporate actions / rolls / re-bases: detect and adjust (§14).
8. Cross-source reconciliation: when two sources overlap, report the agreement rate
   and the largest disagreements; choose the primary source before looking at any
   returns.
9. Quality report per dataset: coverage heatmap (sessions × years), gap list,
   outlier count, reconciliation table, saved as a PNG and embedded in the dataset note.

### 6.4 The data card (`datasets/D-<slug>.md`)
Source and licence, retrieval date, sha256, coverage, frequency, timezone of the raw
data, schema table, known problems, cleaning steps applied with counts, `available_at`
rule (for example "daily close known at 16:00 venue time; macro release known at its
release timestamp, first vintage"), `seen_by` (hypotheses that have read
each tier).

### 6.5 Aggregation
- Bars (time, volume, dollar, tick) are built from the cleanest available level; record
  the construction. Business-time bars buy statistical precision, not edge.
- Panels (session × feature, market × snapshot) are keyed on the cluster unit
  of the domain.
- Cross-asset alignment is done as-of on `available_at`, never on the nominal date.

### 6.6 Free and low-cost starting sources (verify terms before use)
- Daily equities / ETFs: Yahoo Finance via `yfinance` (unofficial; cross-check), Stooq,
  Tiingo (free tier), Alpha Vantage (key).
- Macro: FRED / ALFRED (ALFRED keeps vintages, which prevents revision look-ahead).
- Futures intraday: Databento (paid, per-GB), CME DataMine; FirstRate Data (paid).
- FX: Dukascopy (tick history), central bank reference rates.
- Crypto: exchange REST APIs (Binance, Coinbase, Kraken), funding rates per exchange.
- Prediction markets: Kalshi API (markets, trades, order books, historical),
  Polymarket APIs (Gamma for markets, CLOB for prices and order books) and on-chain
  data; Manifold and Metaculus for calibration-only research (play money /
  forecasts, not tradeable).
- Options: OPRA via Databento, CBOE DataShop (paid); free EOD chains are often
  snapshot-biased. Check timestamps.

---

## 7. Leakage prevention (enforced in code)

### 7.1 Kinds of leakage the engine must catch
| kind | example | guard |
|---|---|---|
| look-ahead feature | centred moving average, full-sample z-score, two-sided filter | causality truncation test (7.4) |
| look-ahead label overlap | 5-day forward label, folds adjacent | purge + embargo (7.5) |
| publication lag | a macro print used on its reference date | `available_at` + as-of join (7.3) |
| revision | today's revised GDP used for 2009 | vintage data (ALFRED) or first-release only |
| survivorship | today's index members, today's listed ETFs, markets that later became popular | point-in-time universe tables |
| selection on outcome | only resolved markets, only liquid days chosen with hindsight | universe defined by `available_at` information |
| preprocessing | scaler / PCA / imputer fitted on all data | fit inside the training fold only (pipeline objects) |
| tier peeking | "just a plot" of TEST | tier guard (7.2) + hidden-trial booking |
| duplicate rows | same event in train and test (multi-listed markets, ETF and its index) | cross-tier duplicate key check |
| construction | zigzag pivots, confirmed swings | timestamp at confirmation; R8.5 |
| execution | fill at the signal bar's close | fill model t+1 (R9.3) |

### 7.2 Tier guard
`srf.data.load(dataset, tier, ctx)` returns rows only for the tier requested.
- DEV: allowed after registration. Logs `data_touch`.
- TEST: allowed only inside `protocol.run_test(h)` after a `freeze` entry for `h`
  whose config hash matches, and only if no earlier `test_open` exists for that hash.
- HOLDOUT: allowed only inside `protocol.run_holdout(h)` after a PASS, once.
- Any other access raises `TierLocked`. There is no override flag. An override is an
  amendment.

### 7.3 As-of joins
Every feature table has `(key, available_at, value)`. Joins use
`merge_asof(left_on="decision_time", right_on="available_at", direction="backward",
allow_exact_matches=False)` unless the domain declares that values become known
exactly at `available_at` and can be acted on in the same instant (state it in the adapter).

### 7.4 Causality truncation test
For every feature function `f` and K random cut points t (default K = 50) in DEV:
`f(data[:t])[-1] == f(data)[t]` to floating tolerance. A second check: shuffle every
row after t and recompute; values up to t must not change. Any failure is a hard
gate (KILL-CLEAN as "failed causality") and names the feature. Run it at
registration time on DEV and again before TEST.

### 7.5 Purged, embargoed cross-validation
Implement purged k-fold and combinatorial purged CV (CPCV) inside DEV. Each label has a
`[t_start, t_end]` window; training rows whose window intersects a validation fold
are purged; an embargo of `h` clusters follows each validation fold.

### 7.6 Target-leak scan
On DEV, flag any single feature whose information about the label is implausibly
high (|rank IC| > 0.3 at intraday horizons, AUC > 0.8, or a feature equal to a
shifted label). A flag is not a verdict; the agent must explain it before going on.

---

## 8. Validation design and the protocol

### 8.1 Tiers
`config/tiers.yaml`:
```yaml
intraday:   { DEV: [2015-01-01, 2020-12-31], TEST: [2021-01-01, 2023-12-31], HOLDOUT: [2024-01-01, null] }
portfolio:  { DEV: [1995-01-01, 2010-12-31], TEST: [2011-01-01, 2019-12-31], HOLDOUT: [2020-01-01, null] }
prediction_markets: { DEV: [...], TEST: [...], HOLDOUT: [...] }   # by market close time, not creation time
embargo_clusters: 2
```
The dates above are placeholders. Use the user's answer to kickoff question 3.
Optional extra tiers: **out-of-sample markets** (a cross-sectional split: markets,
assets or venues never used in selection) and **deep history** (pre-DEV data used
only for sign agreement).

### 8.2 Order of operations (`srf/protocol.py`)
```
register(h)                      → ledger: register (spec_hash, rules_hash, tiers_hash)
dev_select(h)                    → ledger: trial_batch (every config), data_touch(DEV)
controls(h)                      → ledger: control (positive, negative)   [hard gate]
causality(h)                     → ledger: causality                       [hard gate]
placebos(h, tier=DEV)            → ledger: placebo (gross)                 [hard gate if powered]
pbo(h)                           → ledger: pbo                             [soft]
freeze(h)                        → ledger: freeze (config_hash), power (MDE vs MUE)
run_test(h)                      → ledger: test_open, data_touch(TEST)     [once]
placebos(h, tier=TEST)           → ledger: placebo
stats(h)                         → CIs, DSR, FDR/SPA, era signs
cost_sweep(h)                    → primary, stress, 2× slippage on stops
verdict(h)                       → ledger: verdict; LEDGER.md row; finding note; PNG
run_holdout(h)                   → PASS only; once
incubate(h)                      → incubation_start (boundaries), incubation_obs…, incubation_end
```

### 8.3 Measurement studies
Some studies have no trade rule: "the probability that X given Y", or "is Z
calibrated?". They pre-register the estimand, its MUE (the smallest difference
that matters) and the power plan. They still go through tiers, controls and
placebos where they apply. Verdicts: `SUPPORTED` (90% CI beyond the MUE),
`NOT-SUPPORTED` (CI inside ±MUE), or `INCONCLUSIVE`.

### 8.4 Walk-forward
Where a model refits through time, use expanding or rolling walk-forward inside
DEV, refit at a fixed cadence that is pre-registered, and purge and embargo
around every refit boundary. The frozen artefact is the **procedure** (with its
hyperparameters), not one fitted model.

### 8.5 Incubation
Register before the first forward observation: the sequential test (SPRT on cluster P&L,
H0: mean 0 vs H1: mean = frozen expectation, α = 0.05, β = 0.20; or CUSUM on
drawdown), the maximum length, and the size. Every forward cluster is appended to the
ledger as it happens. Crossing the kill boundary closes the hypothesis.
Crossing the confirm boundary allows deployment at the minimum size.

### 8.6 The researcher's loop (put this in the workspace `README.md`)
1. Write the idea in `journal/` → the agent runs the gates (R2) and answers in the note.
2. Create the hypothesis from the template → the agent fills the spec → `srf register H-xxxx`.
3. `srf run H-xxxx --stage dev` → review the DEV PNGs and controls.
4. `srf freeze H-xxxx` → power statement → `srf run H-xxxx --stage test` (once).
5. Read the finding note: verdict, CI against MUE, DSR, PBO, placebos, cost sweep, PNG.
6. PASS → `srf holdout H-xxxx` → `srf incubate H-xxxx`.
7. Anything else → the finding is filed, and the reopen queue on `Home.md` updates.

---

## 9. Statistics and the anti-luck stack

Implement each item in `srf/stats.py` with the stated formula and the unit tests in §16.2.

1. **Per-period Sharpe** `SR = mean / sd` of the primary series; annualise by √(periods per year)
   only for display.
2. **Sharpe standard error (non-normal):** `SE(SR) = sqrt((1 − γ₃·SR + (γ₄ − 1)/4 · SR²) / (T − 1))`,
   where γ₃ is skewness and γ₄ is kurtosis (not excess).
3. **Probabilistic Sharpe Ratio:** `PSR(SR*) = Φ( (SR − SR*) · sqrt(T − 1) / sqrt(1 − γ₃·SR + (γ₄ − 1)/4 · SR²) )`.
4. **Expected maximum Sharpe under the null across N trials:**
   `SR₀ = sqrt(V[SR_n]) · ( (1 − γ)·Φ⁻¹(1 − 1/N) + γ·Φ⁻¹(1 − 1/(N·e)) )`, with γ ≈ 0.5772
   (Euler–Mascheroni) and V[SR_n] the variance of the trial Sharpes (per period).
5. **Deflated Sharpe Ratio:** `DSR = PSR(SR₀)` with N = `N_eff` of the selection
   family. Gate: DSR ≥ 0.95. Report the DSR-implied Sharpe `SR − SR₀` as the haircut.
6. **Effective number of trials `N_eff`:** from the correlation matrix of the
   candidates' return streams, eigenvalues λ: `N_eff = (Σ√λ)² / Σλ`
   (Galwey). Also report the participation ratio `(Σλ)² / Σλ²`. Always report raw N.
7. **PBO by CSCV:** split the T × N matrix of candidate returns into S = 16 blocks. For
   every half/half combination, pick the in-sample best, find its out-of-sample
   relative rank ω, and take λ = ln(ω/(1 − ω)). PBO = share of λ ≤ 0. Soft gate:
   PBO ≤ 0.30. Pure noise gives about 0.5.
8. **Benjamini–Hochberg FDR** at q = 0.10 for screens: sort the p-values, find the
   largest k with p₍k₎ ≤ k·q/m, and reject 1…k. Use Benjamini–Yekutieli when the tests are
   strongly dependent and that dependence is unknown.
9. **Stationary block bootstrap** (Politis–Romano) with the automatic block length of
   Politis–White, using the Patton–Politis–White correction, floored at 5 clusters.
   Use `BOOT_N = 5000` draws for verdicts and 2,000 for diagnostics. Use percentile or BCa
   intervals at 90%.
10. **Power and MDE:** `MDE = (z₁₋α + z_power) · σ / sqrt(n_eff_clusters)`, one-sided
    α = 0.05, power 0.80. For autocorrelated series, inflate σ by the bootstrap's
    long-run variance ratio.
11. **SPRT (incubation):** with per-cluster P&L x ~ N(μ, σ²), H0: μ = 0, H1: μ = μ₁,
    `LLR_n = Σ (μ₁·x_i − μ₁²/2) / σ²`; stop for H1 at `ln((1 − β)/α)` and for H0 at `ln(β/(1 − α))`.
12. **Hansen SPA** (diagnostic): the consistent p-value of the best candidate against
    a benchmark across the family.
13. **Era sign agreement:** split post-start-date DEV + TEST into ≥ 3 equal eras; the
    sign of the primary metric must agree in all eras (soft gate).
14. **Parameter stability:** report the neighbourhood of the chosen configuration (±1
    grid step on each axis). A lone peak surrounded by losers is flagged
    `isolated_optimum`.
15. **Winner's curse:** re-run the chosen configuration on a fresh seed or fresh resample
    before reporting.

Reading guide (written to `knowledge/concepts/K-reading-results.md`):
a t-stat of 2 after 100 trials is expected from noise (about 2.3% of noise trials
clear a one-sided 2.0, so 2 or 3 of 100 will). Harvey, Liu and Zhu argue that
new factors should clear t ≈ 3. Annual Sharpe above about 3 on daily data, a very
smooth equity curve, win rates far above the payoff ratio's break-even, or
results concentrated in one era should all prompt an audit before they are
believed.

---

## 10. Controls and placebos (`srf/controls.py`, `srf/placebos.py`)

1. **Positive control:** inject a known edge (for example +0.15σ per trade on a
   random subset of DEV signals, or a synthetic series with a planted AR
   coefficient). The engine must detect it at t ≥ 4 with the correct sign and size
   within its CI.
2. **Negative control:** a random walk, or DEV data with the signal replaced by random
   timestamps. It must not be detected (|t| < 2 in 95% of seeds).
3. **Placebos (trading):** (a) random entries matched on time-of-day / time-to-event and
   count; (b) wrong-time: the rule applied to a different session, event or market with
   the same marginal distributions; (c) inverted sign. Use ≥ 1,000 draws, judged on gross.
4. **Placebos (prediction markets):** random side matched on price bucket and
   time-to-resolution; the signal shifted to an unrelated market in the same category; the
   timestamp shifted to before the information existed (it must lose its edge).
5. **Placebos (portfolio):** random portfolios matched on turnover, number of
   holdings and volatility; the signal lagged by one extra period (a real slow signal
   survives a lag; a leak usually does not); and the signal permuted across
   assets within each date.
6. **Placebo power:** a placebo failure is a hard KILL-CLEAN only on a sample powered for
   the MUE. Otherwise it is INCONCLUSIVE.

---

## 11. Costs and execution (`srf/costs.py`, `srf/fills.py`)

- Cost sheets live in `config/costs/<venue>/<date>.yaml`, copied from the venue's own
  pages with the URL and retrieval date. `srf/costs.py` refuses a missing or
  unverified value. It never falls back to a guess.
- When no venue is named, use the reference cost model of the domain adapter and
  label every verdict `no-venue`. Before trading, re-check the verdict against the
  real sheet.
- Components: commission and fees, spread (observed at the fill time), slippage
  (empirical distribution conditioned on volatility and time of day where data allows),
  borrow / funding / financing, FX conversion, taxes (optional layer, labelled).
- Report a cost sweep: 0×, 1×, 1.5×, 2× primary, plus stress. Report the cost at
  which the net edge reaches zero ("break-even cost") in sigma units.
- Two execution views for overlapping signals: **TRADE** (each signal is its own round
  trip) and **NET** (target position; only the change trades). Declare which one drives
  the verdict at registration.

---

## 12. Verdicts (`srf/verdict.py`)

Inputs: the 90% CI of the primary metric (net, primary cost model), the gross CI,
the MUE, gate results and power.

| class | rule | terminal? |
|---|---|---|
| **PASS** | CI lower bound ≥ MUE on primary costs; every soft gate passed; powered; and the stress model still positive | no: goes to HOLDOUT, then incubation |
| **KILL-CLEAN** | CI upper bound < MUE; or gross CI upper bound ≤ 0; or a hard gate failed (causality, controls, integrity); or a powered placebo failure | **yes** |
| **INCONCLUSIVE** | CI contains both 0 and the MUE (underpowered); or reaches the MUE without establishing it; or a soft gate failed; or registered underpowered; or a placebo failed on an unpowered sample | no: reopen queue |
| **KILL-COST** | gross and primary pass, stress fails | no: reopen if costs change |
| **KILL-ERA** | fails only because of pre-start-date eras | no |

Every verdict writes: a ledger entry, a row in `LEDGER.md`, a finding note from
the template, the verdict-sheet PNG, and status and verdict updates on the hypothesis note.

---

## 13. Reporting and the mandatory PNG

### 13.1 Rule
Every test, in every domain, writes at least one PNG through `srf/plotting.py`.
The report embeds it, and the finding note embeds it with `![[…]]`. Before
reporting, the agent looks at the image (not just the file). A test with no
PNG is not finished.

### 13.2 The verdict sheet (`report.verdict_sheet(h)`) — one PNG, 2 × 3 panels
1. Equity curve of the primary series: DEV in mid grey, TEST in near-black, with
   5–95% and 25–75% bootstrap bands.
2. Drawdown (red).
3. 90% CI of the primary metric against the zero and MUE reference lines (refline tags).
4. Placebo distributions with the actual value (vertical line).
5. Cost sweep (net expectancy against the cost multiple, break-even marked in amber).
6. Era bars (metric per era, gradient bars) plus the parameter neighbourhood heat map.
Caption: hypothesis id, verdict, N_total, N_eff, MDE, MUE, DSR, PBO. Footer: data
file, rows, tiers.

### 13.3 House style (light)
- White plane. No spines, no grid, no tick marks. Small grey tick labels.
- Series are monochrome: near-black is primary / actual / OOS, mid grey is
  secondary / in-sample, light greys are distribution bands.
- Colour only where it means something: **red** = drawdown / loss / failure,
  **amber** = limit / threshold, **pale blue** fill = positive area under a line,
  **green** = positive numbers / pass, **steel blue** = a third series when the greys run out.
- A legend of small square swatches in one row above the plot. A small dark
  title, a grey subtitle, a grey caption line for the run summary.
- Dashed reference lines carry a boxed value tag on the y-axis.
- Bars fade from their colour at the far end to white at the baseline.
- A footer status line: source file, rows, split.
- Use the helpers. Never hand-roll matplotlib styling. The reference
  implementation is in Appendix A; copy it to `code/srf/plotting.py`.

---

## 14. Domain adapters

Every domain implements:

```python
@dataclass
class Domain:
    name: str
    cluster_unit: str                 # "session", "rebalance_period", "event"
    calendar: Calendar                # trading days/sessions, holidays, early closes, tz
    primary_metric: str               # e.g. "net_pnl_per_session"
    mue_floor: Callable[[Spec], float]
    cost_model: CostModel             # primary + stress
    fill_model: FillModel
    placebos: list[Placebo]
    hygiene_checks: list[Check]       # domain-specific data checks, run at ingest and at registration
    min_trades: int
    min_clusters: int
```

### 14.1 Intraday (futures, equities, FX, crypto)
1. **Sessions in exchange time.** Define each session's open and close in the venue's
   time zone, with holidays, early closes and halts. The cluster is the session.
2. **Continuous futures:** returns only between bars of one contract, or
   back-adjusted (additive "Panama" for point P&L, ratio for log returns, stated per
   project). Levels on an adjusted series are not quotable prices. Detect roll
   steps (an outsized move at the stitch on low volume) and screen sessions trading
   the expiring contract.
3. **Quotes and trades:** use the exchange aggressor flag where available; do not
   de-duplicate same-millisecond fills; quote returns off the mid to remove
   bid-ask bounce; measure the noise floor first.
4. **Fills:** next bar, crossing the spread. Stops fill at least one tick through
   with empirical slippage. Market orders carry latency slippage. Any same-bar
   ambiguity goes against the trade.
5. **MUE floor:** net expectancy ≥ 1 tick + commission per round trip, and
   per-session Sharpe stated.
6. **Optional: evaluation accounts and barrier payoffs (prop firms).** Simulate the
   intraday path against every rule on the dated sheet (trailing or static drawdown, daily loss,
   consistency, flat time, payout rules). Validate against the closed forms first:
   a zero-drift static floor passes with P = D/(T + D), and a continuously trailing
   floor with P = exp(−T/D). The objective is θ = 2μ/σ² (equity units), not
   Sharpe. Price a ticket as V = −fee − activation·P(pass) + P(pass)·E[payouts], and
   always against the zero-edge baseline under the same rules.
7. **Crypto specifics:** trades 24/7 (define an artificial session, for example 00:00 UTC);
   exchange fragmentation (choose a primary venue in advance); funding
   rates for perpetuals are part of P&L; delisted coins create survivorship.

### 14.2 Portfolio / long-horizon (PM)
1. **Total return** (price + distributions), daily. Never mix price-only with total
   return. No fills on zero-volume days.
2. **Decision time:** a signal uses closes published before the decision. Fills are at the
   next tradable price.
3. **Point in time:** universe membership, index constituents, fundamentals and
   macro values as they were known (vintages). No back-calculated index history
   treated as investable. Record each instrument's investable-from date.
4. **Survivorship:** a rule that selects from today's list of instruments is
   biased. Use point-in-time lists, or restrict the claim.
5. **Currency:** convert at the time-stamped rate; never mix hedged and unhedged.
6. **Cluster unit:** the rebalance period (default monthly). Small samples are normal.
   Lean on out-of-sample markets, deep-history sign agreement, start-date
   sensitivity (rerun with every start year) and one-episode dependence (drop the
   best and worst year).
7. **Benchmark and risk matching:** compare against a deployable benchmark scaled to
   the same volatility. Never compare a strategy with an unscaled benchmark.
8. **Extra verdict class:** `NON-INFERIOR` means not worse than the benchmark by more
   than δ (pre-registered) at 90%, with lower cost, drawdown or complexity. It is
   useful when superiority cannot be powered.
9. **Optional layers:** tax (a lot-level engine, a dated tax sheet, verdicts
   labelled after-tax) and account realism (minimum trade size, fractional shares,
   contributions). The default is pre-tax, labelled pre-tax.
10. **Placebos** as §10.5. Leveraged and inverse ETF decay is not a signal.

### 14.3 Prediction markets
1. **Unit and clustering:** the observation is a position in a market, and the cluster is
   the **event**. Mutually exclusive outcomes of one event are one cluster. Also
   cluster by resolution date and category when outcomes are correlated (elections,
   macro releases, sports leagues).
2. **Price is a probability only net of fees and spread.** Model the venue's fee formula
   from a dated sheet. Fees often depend on price (for example proportional to p·(1 − p))
   and on maker or taker status. Include deposit and withdrawal costs.
3. **Point-in-time snapshots:** use order-book or trade data timestamped before the
   decision. Never use the last price, the settlement price, or the "closing" probability
   to make the decision. Mid-prices in thin books are not executable. Fill at the
   book's ask or bid for the size traded, with partial fills.
4. **Selection leakage:** a market list pulled today contains only markets that still
   exist, were listed in a certain way, or became popular. Build the universe from
   point-in-time listings (creation time and status at the decision time). Do
   not filter on final volume.
5. **Resolution risk:** ambiguous or disputed resolutions, voids, rule changes,
   early resolution. Record the resolution source and rules per market. Treat disputed
   markets as their own bucket.
6. **Capital lock-up:** returns are per dollar per day held until resolution or
   exit. Annualise on time locked, and compare against the risk-free rate.
7. **Metrics:** net return per dollar staked (primary), Brier score and log loss
   against the market's own forecast (a skill score > 0 is needed), calibration
   (a reliability diagram with CIs), and closing-line value (did the price move
   towards your side after entry).
8. **Capacity:** the maximum size from book depth at entry. An edge that exists only
   at $5 is a measurement, not a strategy.
9. **Jurisdiction and access:** record in the venue note whether the user can
   legally access the venue and products. This is recorded, not tested.
10. **MUE floor:** expected net return per dollar above fees + spread + the risk-free carry over
    the lock-up, at the size the book supports.
11. **Placebos** as §10.4.

### 14.4 Other domains
Sports betting (closing-line value as the primary skill metric, bookmaker limits as
capacity), options (point-in-time chains, early-exercise and dividend handling, bid/ask
fills, never mid), macro / FX (vintages, release timestamps, carry). Write a new adapter
with the same fields, plus a `markets/` note on mechanics, before the first registration.

---

## 15. Engine API (`code/srf/`)

| module | responsibilities |
|---|---|
| `config.py` | load `project.yaml`, `tiers.yaml`, constants (§17); expose `TIERS`, seeds, paths |
| `ledger.py` | `append(kind, payload) -> seq`; each line `{seq, ts, kind, hypothesis, payload, payload_sha256, prev, hash}` with `hash = sha256(prev + canonical_json(line_without_hash))`; `verify()`; HEAD file; refuses writes if verify fails |
| `registry.py` | `register(spec) -> H-id` (checks required fields, MUE ≥ domain floor, tiers and rules hashes, family); `log_trials(h, configs)`; `log_hidden(h, description, n)`; `n_total()`, `n_eff(h)`; `freeze(h, config)`; `reopen(h, reason)`; `reopen_queue()` |
| `data/` | `sources/` adapters, `ingest.py`, `clean.py`, `quality.py` (report + PNG), `calendar.py`, `load.py` (tier guard), `asof.py` |
| `features.py` | feature registry with `available_at` lag; `assert_causal(f, df, k=50)` |
| `cv.py` | purged k-fold, CPCV, walk-forward with embargo |
| `engines/` | `trades.py` (event backtest, TRADE / NET), `portfolio.py` (weights → returns with costs), `pm.py` (market positions to resolution / exit), `barrier.py` (path rule simulator) |
| `costs.py`, `fills.py` | §11 |
| `stats.py` | §9 |
| `controls.py`, `placebos.py` | §10 |
| `protocol.py` | §8.2 order of operations; each stage writes to the ledger |
| `verdict.py` | §12 |
| `report.py` | required fields (R12.1), finding note writer, verdict sheet |
| `notes.py` | read and write Obsidian notes with frontmatter; keep tags in step with properties |
| `plotting.py` | Appendix A |
| `cli.py` | `srf register | run | freeze | holdout | incubate | verify | status | new <type>` |

Spec object (dataclass, hashed canonically):

```python
@dataclass(frozen=True)
class Spec:
    title: str; domain: str; origin: str
    mechanism: str; gates: dict; prediction: str; second_prediction: str | None
    start_date: str
    datasets: list[str]; tiers_hash: str
    rule: dict                       # entries/exits/sizing/fills/timeouts, or estimand for measurement
    grid: dict                       # parameter -> values; size = product
    primary_metric: str; mue: float; mue_unit: str
    execution_view: str              # TRADE or NET
    controls: list[str]; placebos: list[str]
    power_plan: dict                 # expected clusters, sigma estimate source
    family: list[str]                # earlier H-ids whose results chose this
    kill_criteria: str
```

---

## 16. Test suite (`code/tests/`)

### 16.1 Integrity
- The ledger tamper test: edit one byte of a past line and `verify()` fails; `cli run` refuses.
- The registry refuses a missing field, an MUE below the floor, and changed tiers.
- Tier guard: TEST before freeze → `TierLocked`; a second TEST read → `TierLocked`; HOLDOUT
  without PASS → `TierLocked`.
- Ingest is idempotent: same raw in → same hashes out.

### 16.2 Statistics (known answers)
- PSR / DSR: reproduce the worked example in Bailey & López de Prado (2014), to 3 decimals.
- Expected max Sharpe: simulation of N ∈ {10, 100} i.i.d. noise strategies matches the formula within 5%.
- PBO: the noise matrix gives 0.5 ± 0.1; a matrix with a strictly dominant column gives < 0.1.
- BH: the textbook example (Benjamini & Hochberg 1995, 15 p-values) gives the published rejections.
- Bootstrap coverage: on an AR(1) with φ = 0.5, the 90% CI of the mean covers the truth in 88–92% of 500 runs.
- N_eff: identical streams give 1; independent streams give N.
- Power: the closed-form MDE matches simulated power within 3 points at 80%.

### 16.3 End-to-end acceptance (synthetic data shipped as fixtures)
1. **Planted edge:** synthetic intraday data with an edge of 2× MUE → PASS.
2. **Noise grid:** 200 configurations on a random walk → never PASS; DSR < 0.95; PBO ≈ 0.5.
3. **Leak:** a centred-MA feature → causality hard gate → KILL-CLEAN (failed causality).
4. **Underpowered:** a true edge = MUE with 30 clusters → INCONCLUSIVE with an MDE statement.
5. **Cost:** a gross edge of 0.5 × cost → KILL-CLEAN with "negative net, positive gross".
6. **Prediction markets:** simulated markets with a favourite–longshot bias of
   known size, before and after fees → the verdict matches the known net value.
7. **Portfolio:** a survivorship-biased universe fixture → the hygiene check flags it.
8. Every acceptance run produces its PNG and a finding note.

---

## 17. Constants (defaults in `config/constants.yaml`)

| name | default | use |
|---|---|---|
| `SEED` | 20260101 | all randomness |
| `CI_LEVEL` | 0.90 | every decision CI |
| `POWER` | 0.80 | MDE |
| `ALPHA_PLACEBO` | 0.05 | one-sided, gross |
| `BOOT_N` / `BOOT_N_DIAG` | 5000 / 2000 | bootstrap draws |
| `BLOCK_FLOOR` | 5 | clusters |
| `DSR_MIN` | 0.95 | selection-family N_eff |
| `PBO_MAX` | 0.30 | soft gate |
| `FDR_Q` | 0.10 | screens |
| `CSCV_S` | 16 | PBO blocks |
| `CONTROL_T` | 4.0 | positive control |
| `EMBARGO_CLUSTERS` | 2 | tier and fold embargo (≥ label horizon) |
| `MIN_TRADES` / `MIN_CLUSTERS` | 200 / 100 | floors (domain may override, declared in RULES before the first registration) |
| `CAUSALITY_CUTS` | 50 | truncation test points |
| `ERAS` | 3 | sign agreement |

---

## 18. Templates (write each to `templates/`)

### `templates/Hypothesis.md`
```markdown
---
id: H-XXXX
type: hypothesis
title:
created: {{date}}
domain:
status: draft
verdict: none
origin: user
mechanism:
mue: { metric: , value: , unit: }
primary_metric:
datasets: []
tiers_hash:
spec_hash:
ledger_seq:
family: []
tags: [type/hypothesis, status/draft, verdict/none]
---
# {{title}}

## Idea (one paragraph, plain words)
## Gates (R2)
| gate | answer |
|---|---|
| mechanism | |
| observable (dataset, column, frequency) | |
| searcher density | |
| persistence | |
| MUE + kill criterion | |
| second prediction | |
| start date | |
## Prediction (with sign)
## Rule (fixed before looking)
## Grid (parameters, values, size)
## Statistics, controls, placebos
## Power plan
## Kill criteria
## Experiments
## Verdict
```

### `templates/Experiment.md`
```markdown
---
id: E-XXXX
type: experiment
title:
created: {{date}}
hypothesis: "[[H-XXXX]]"
stage: dev
config_hash:
seed:
ledger_seq:
figures: []
tags: [type/experiment, stage/dev]
---
# {{title}}
## What ran (code path, config, data, tier)
## Output
![[attachments/E-XXXX.png]]
## Checks run (R8 checklist items)
## Bugs caught (and effect on the number)
```

### `templates/Finding.md`
```markdown
---
id: F-XXXX
type: finding
title:
created: {{date}}
hypothesis: "[[H-XXXX]]"
verdict:
n_total:
n_eff:
mde:
mue:
cost_model:
ledger_seq:
figures: []
tags: [type/finding]
---
# {{title}}
![[attachments/F-XXXX_verdict.png]]
## Verdict and why (in the protocol's terms, R12.2)
## Numbers (CI, DSR, PBO, placebos, cost sweep, eras)
## Power statement
## What would reopen it
```

### `templates/Dataset.md`, `templates/Concept.md`, `templates/Source.md`, `templates/Venue.md`, `templates/Lesson.md`, `templates/Journal.md`
Same pattern: the required properties from §3.1, then the sections named in §4 and
§6.4. `Concept.md` sections: Summary, Intuition, Formula, How the engine uses it, Pitfalls,
Sources, Used by. `Source.md` sections: Claim, Method, Data and sample, Costs and
multiple testing, Our critique, Tradeability, Related.

---

## 19. References (seed one source note each)

- Bailey, D. H. & López de Prado, M. (2014). The Deflated Sharpe Ratio. *Journal of Portfolio Management* 40(5).
- Bailey, D. H., Borwein, J., López de Prado, M. & Zhu, Q. J. (2017). The Probability of Backtest Overfitting. *Journal of Computational Finance* 20(4).
- Bailey, D. H. & López de Prado, M. (2012). The Sharpe Ratio Efficient Frontier. *Journal of Risk* 15(2).
- López de Prado, M. (2018). *Advances in Financial Machine Learning*. Wiley. (purged k-fold, CPCV, meta-labelling)
- Harvey, C. R., Liu, Y. & Zhu, H. (2016). …and the Cross-Section of Expected Returns. *Review of Financial Studies* 29(1).
- Harvey, C. R. & Liu, Y. (2015). Backtesting. *Journal of Portfolio Management* 42(1).
- White, H. (2000). A Reality Check for Data Snooping. *Econometrica* 68(5).
- Hansen, P. R. (2005). A Test for Superior Predictive Ability. *Journal of Business & Economic Statistics* 23(4).
- Benjamini, Y. & Hochberg, Y. (1995). Controlling the False Discovery Rate. *JRSS B* 57(1).
- Benjamini, Y. & Yekutieli, D. (2001). The control of the FDR under dependency. *Annals of Statistics* 29(4).
- Politis, D. N. & Romano, J. P. (1994). The Stationary Bootstrap. *JASA* 89(428).
- Politis, D. N. & White, H. (2004); Patton, A., Politis, D. N. & White, H. (2009) correction. *Econometric Reviews*.
- Lo, A. W. (2002). The Statistics of Sharpe Ratios. *Financial Analysts Journal* 58(4).
- Mertens, E. (2002). Comments on variance of the IID estimator in Lo (2002). Working note.
- Wald, A. (1945). Sequential Tests of Statistical Hypotheses. *Annals of Mathematical Statistics* 16(2).
- Ioannidis, J. P. A. (2005). Why Most Published Research Findings Are False. *PLoS Medicine* 2(8).
- McLean, R. D. & Pontiff, J. (2016). Does Academic Research Destroy Stock Return Predictability? *Journal of Finance* 71(1).
- Galwey, N. W. (2009). A new measure of the effective number of tests. *Genetic Epidemiology* 33(7).
- Broadie, M., Glasserman, P. & Kou, S. (1997). A continuity correction for discrete barrier options. *Mathematical Finance* 7(4).
- Snowberg, E. & Wolfers, J. (2010). Explaining the Favorite–Long Shot Bias. *Journal of Political Economy* 118(4).
- Wolfers, J. & Zitzewitz, E. (2004). Prediction Markets. *Journal of Economic Perspectives* 18(2).
- Brier, G. W. (1950). Verification of forecasts expressed in terms of probability. *Monthly Weather Review* 78(1).
- Arnott, R., Harvey, C. R. & Markowitz, H. (2019). A Backtesting Protocol in the Era of Machine Learning. *Journal of Financial Data Science* 1(1).

---

## Appendix A — `code/srf/plotting.py` (reference implementation, light house style)

Copy verbatim. Set `FIGURES` from `config.py`. Example use:

```python
import srf.plotting as PL
f, ax = PL.fig(1, 1, w=10, h=5)
ax.plot(x, equity_dev, color=PL.SECONDARY, label="DEV")
ax.plot(x, equity_test, color=PL.PRIMARY, label="TEST")
PL.band(ax, x, lo5, hi95); PL.band(ax, x, lo25, hi75, inner=True)
PL.refline(ax, 0, "0")
PL.title(ax, "H-0007 equity, net of primary costs", "per-session sum, bootstrap bands 5-95 / 25-75")
PL.legend(ax)
PL.caption(f, "INCONCLUSIVE · N_total 38 · N_eff 11.2 · MDE 1.9 t · MUE 1.5 t · DSR 0.71 · PBO 0.22")
PL.save(f, "F-0004_verdict", note=["D-es-1m", "212,880 rows", "DEV 2015-2020 | TEST 2021-2023"])
```

```python
"""House chart style -- "paper" light look. Every test PNG goes through these helpers.

  * white plane, no spines, no grid, ticks without marks, small grey tick labels
  * series are MONOCHROME: near-black = primary/actual/OOS, mid grey = secondary/IS,
    light greys = distribution bands (25-75% inner, 5-95% outer)
  * colour only carries meaning: RED = drawdown/loss, AMBER = limit/threshold,
    pale BLUE fill = positive area under a rolling line, GREEN = positive numbers,
    STEEL blue = a third series when greys run out
  * legend = row of small square swatches above the plot, left-aligned (legend())
  * small dark panel title above the legend; grey caption line for run summary (caption())
  * dashed reference lines carry a boxed value tag on the y axis (refline())
  * bars fade from their colour at the far end to white at the baseline (bars())
  * footer status line: source file, rows, split (save(note=[...]))
"""
from __future__ import annotations
from pathlib import Path
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
import matplotlib.ticker as mticker
import numpy as np
from matplotlib.colors import LinearSegmentedColormap, to_rgba, to_rgba_array
from matplotlib.patches import Patch

FIGURES = Path("results/figures")   # overridden from srf.config at import in the workspace

# surfaces and ink
PLANE = SURFACE = "#ffffff"
INK, INK2, MUTED = "#1a1a1a", "#4a4a4a", "#8a8a8a"
GRID, AXIS, REF = "#eeeeee", "#c8c8c8", "#9e9e9e"
TAG_BG = "#e9e9e9"

# monochrome series slots
PRIMARY, SECONDARY, TERTIARY, QUATERNARY = "#1a1a1a", "#7d7d7d", "#b5b5b5", "#d6d6d6"
BAND_INNER, BAND_OUTER = "#d4d4d4", "#ececec"

# semantic accents (only when the colour means something)
RED, AMBER, BLUEFILL, GREEN, STEEL = "#c62828", "#d18f00", "#d9e3f5", "#1b8a5a", "#3d64b8"
LIMIT = AMBER
SERIES5 = [PRIMARY, SECONDARY, STEEL, TERTIARY, AMBER]
DASHES = ["-", (0, (4, 2)), "-", (0, (1, 1.5)), "-"]

SEQ = LinearSegmentedColormap.from_list("rf_seq", ["#ffffff", "#d9e3f5", "#8fa8dc", "#3d64b8", "#1c2f66", "#0d0d0d"])
DIV = LinearSegmentedColormap.from_list("rf_div", ["#3d64b8", "#9fb3e6", "#f4f4f4", "#f08a85", "#c62828"])

plt.rcParams.update({
    "figure.facecolor": PLANE, "axes.facecolor": SURFACE, "savefig.facecolor": PLANE,
    "axes.edgecolor": AXIS, "axes.labelcolor": MUTED, "axes.titlecolor": INK,
    "text.color": INK2, "xtick.color": MUTED, "ytick.color": MUTED,
    "xtick.major.size": 0, "ytick.major.size": 0, "xtick.minor.size": 0, "ytick.minor.size": 0,
    "xtick.major.pad": 5, "ytick.major.pad": 5, "xtick.labelsize": 8.5, "ytick.labelsize": 8.5,
    "grid.color": GRID, "grid.linewidth": 0.6, "axes.grid": False, "axes.axisbelow": True,
    "axes.spines.top": False, "axes.spines.right": False,
    "axes.spines.left": False, "axes.spines.bottom": False,
    "font.family": "sans-serif", "font.sans-serif": ["Inter", "Segoe UI", "Helvetica", "DejaVu Sans"],
    "font.size": 9, "axes.labelsize": 8.8,
    "legend.frameon": False, "legend.fontsize": 8.8, "legend.labelcolor": INK2,
    "figure.dpi": 150, "savefig.dpi": 200, "savefig.bbox": "tight",
    "lines.linewidth": 1.3, "lines.solid_capstyle": "round",
    "axes.prop_cycle": matplotlib.cycler(color=SERIES5),
})
for cm in (SEQ, DIV):
    if cm.name not in matplotlib.colormaps:
        matplotlib.colormaps.register(cm)
plt.rcParams["image.cmap"] = "rf_seq"


def fig(nrows=1, ncols=1, w=12.0, h=6.8, **kw):
    return plt.subplots(nrows, ncols, figsize=(w, h), **kw)


def _esc(t):
    """matplotlib parses $...$ as mathtext; dollar amounts must be escaped."""
    return str(t).replace("$", r"\$")


def _place_title(ax):
    arts = getattr(ax, "_rf_title", None)
    if not arts:
        return
    y = 4 + (18 if getattr(ax, "_rf_toplegend", False) else 0)
    for a in reversed(arts):
        a.xyann = (0, y)
        y += 14


def title(ax, main, sub=None):
    ax.set_title("")
    kw = dict(xy=(0, 1), xycoords="axes fraction", textcoords="offset points",
              ha="left", va="bottom", annotation_clip=False)
    arts = [ax.annotate(_esc(main), xytext=(0, 0), color=INK, fontsize=10.5, **kw)]
    if sub:
        arts.append(ax.annotate(_esc(sub), xytext=(0, 0), color=MUTED, fontsize=8.6, **kw))
    ax._rf_title = arts
    _place_title(ax)


def _handle_color(h):
    if hasattr(h, "patches"):
        h = h.patches[0]
    elif hasattr(h, "lines"):
        h = h.lines[0] if h.lines[0] is not None else h.lines[2][0]
    for get in ("get_color", "get_facecolor"):
        if hasattr(h, get):
            c = to_rgba_array(getattr(h, get)())
            if len(c):
                return (*c[0, :3], 1.0)
    return INK2


def legend(ax, items=None):
    """Square-swatch legend in one row above the plot. items: [(label, color)] or None."""
    if items is None:
        hs, ls = ax.get_legend_handles_labels()
        items = [(l, _handle_color(h)) for h, l in zip(hs, ls)]
    if not items:
        return None
    handles = [Patch(facecolor=c, edgecolor="none") for _, c in items]
    lg = ax.legend(handles, [_esc(l) for l, _ in items], loc="lower left",
                   bbox_to_anchor=(0, 1.0), ncol=len(items), borderaxespad=0.25, borderpad=0,
                   handlelength=0.9, handleheight=0.9, handletextpad=0.45, columnspacing=1.6)
    ax._rf_toplegend = True
    _place_title(ax)
    return lg


def caption(f, text):
    f._rf_caption = f.text(0.006, 0.998, _esc(text), color=MUTED, fontsize=8.4, ha="left", va="top")


def zeroline(ax, y=0.0, **kw):
    ax.axhline(y, color=kw.pop("color", AXIS), lw=kw.pop("lw", 0.9), zorder=1, **kw)


class _TagGap(mticker.Formatter):
    """Blanks y tick labels that a refline tag would sit on."""
    def __init__(self, ax, base):
        self.ax, self.base, self.ys = ax, base, []

    def set_axis(self, axis):
        super().set_axis(axis); self.base.set_axis(axis)

    def set_locs(self, locs):
        self.base.set_locs(locs)

    def get_offset(self):
        return self.base.get_offset()

    def __call__(self, v, pos=None):
        lo, hi = self.ax.get_ylim()
        if any(abs(v - y) < 0.04 * abs(hi - lo) for y in self.ys):
            return ""
        return self.base(v, pos)


def refline(ax, y, label=None, color=REF, tag=True):
    """Dashed horizontal reference with a boxed value tag on the y axis."""
    ax.axhline(y, color=color, lw=0.9, ls=(0, (4, 3)), zorder=1)
    if tag:
        fmt = ax.yaxis.get_major_formatter()
        if not isinstance(fmt, _TagGap):
            fmt = _TagGap(ax, fmt)
            ax.yaxis.set_major_formatter(fmt)
        fmt.ys.append(y)
        ax.yaxis.labelpad = max(ax.yaxis.labelpad, 34)   # keep the axis label clear of the tag
        ax.annotate(_esc(label if label is not None else f"{y:g}"), xy=(0, y),
                    xycoords=ax.get_yaxis_transform(), xytext=(-3, 0), textcoords="offset points",
                    ha="right", va="center", fontsize=8, color=INK, annotation_clip=False,
                    bbox=dict(boxstyle="rarrow,pad=0.28", fc=TAG_BG, ec="none"))


def vline(ax, x, color=LIMIT, **kw):
    ax.axvline(x, color=color, lw=kw.pop("lw", 1.2), zorder=kw.pop("zorder", 3), **kw)


def band(ax, x, lo, hi, inner=False, label=None):
    return ax.fill_between(x, lo, hi, color=BAND_INNER if inner else BAND_OUTER,
                           lw=0, zorder=0.6 if inner else 0.5, label=label)


def posfill(ax, x, y, color=BLUEFILL, base=0.0):
    y = np.asarray(y, float)
    return ax.fill_between(x, base, y, where=y > base, interpolate=True, color=color, lw=0, zorder=0.8)


def bars(ax, x, h, color=PRIMARY, width=0.8, bottom=0.0, fade=True, label=None, floor=0.10):
    """Bars that fade from `color` at the far end to white at the baseline."""
    cont = ax.bar(x, h, width=width, bottom=bottom, color=color, linewidth=0, label=label,
                  alpha=0.0 if fade else 1.0)
    if not fade:
        return cont
    r, g, b, _ = to_rgba(color)
    grad = np.zeros((128, 1, 4))
    grad[..., 0], grad[..., 1], grad[..., 2] = r, g, b
    grad[..., 3] = (floor + (0.95 - floor) * np.linspace(1.0, 0.0, 128) ** 1.6)[:, None]
    for p in cont.patches:
        x0, y0, w, hh = p.get_x(), p.get_y(), p.get_width(), p.get_height()
        if hh == 0 or not np.isfinite(hh):
            continue
        ext, org = ((x0, x0 + w, y0, y0 + hh), "upper") if hh > 0 else ((x0, x0 + w, y0 + hh, y0), "lower")
        ax.imshow(grad, extent=ext, origin=org, aspect="auto", interpolation="bilinear", zorder=2)
    ax.relim(); ax.autoscale_view()
    lo, hi = ax.get_ylim()
    if lo > hi:
        ax.set_ylim(hi, lo)
    return cont


def footer(f, *items):
    f._rf_footer = "      ".join(_esc(i) for i in items if i)


def save(f, name, note="", path=None):
    """Write the PNG. note = [source file, rows, split] for the footer status line."""
    if note:
        footer(f, *([note] if isinstance(note, str) else note))
    for ax in f.axes:
        _place_title(ax)
    h_pt = f.get_figheight() * 72
    top = 1 - (16 / h_pt if getattr(f, "_rf_caption", None) else 0)
    bot = 16 / h_pt if getattr(f, "_rf_footer", None) else 0
    try:
        f.tight_layout(h_pad=3.6, w_pad=2.4, rect=(0, bot, 1, top))
    except Exception:
        pass
    if getattr(f, "_rf_footer", None):
        f.text(0.006, 0.004, f._rf_footer, color=MUTED, fontsize=7.6, ha="left", va="bottom")
    p = Path(path) if path is not None else FIGURES / f"{name}.png"
    p.parent.mkdir(parents=True, exist_ok=True)
    f.savefig(p); plt.close(f)
    print(f"    [fig] {p}")
    return p
```
