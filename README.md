# Systematic Research Framework

A build specification for a research workspace that makes false positives hard
and closes dead strategies cleanly. Hand it to a coding agent, and the agent builds
an Obsidian vault plus a Python engine for testing trading and forecasting ideas:
intraday, long-horizon portfolio, prediction markets, or any domain you add.


| file | what it is |
|---|---|
| [`FRAMEWORK.md`](FRAMEWORK.md) | The spec the agent reads and builds from: ruleset, data layer, leakage guards, validation protocol, anti-overfitting statistics, verdicts, Obsidian conventions, templates, tests, plotting style. |
| [`Systematic-Research-Framework-negvol-2026.pdf`](Systematic-Research-Framework-negvol-2026.pdf) | The companion paper for humans: probability basics, the research workflow from idea to deployment, how to read results, and why each rule exists. Read it to understand what your agent is doing. |

## What you get once it is built

- **An Obsidian vault**: a hypothesis, experiment and finding note per test, linked
  together, with typed properties, nested tags and Dataview dashboards. It also holds a knowledge store
  (concepts, sources, venues, market mechanics, method lessons) that is kept separate
  from results.
- **An append-only, hash-chained ledger** that records every registration, every
  configuration tried (including ones seen only in notebooks), every data access and
  every verdict.
- **A data layer** with immutable raw files, hashes, cleaning logs, as-of joins on
  the time each value became known, and quality reports.
- **Leakage guards**: TEST and HOLDOUT data are locked until the protocol opens them,
  every feature must pass a truncation (causality) test, and cross-validation is
  purged and embargoed.
- **An anti-luck stack**: pre-registration, trial counting, effective number of
  trials, the Deflated Sharpe Ratio, PBO by CSCV, BH-FDR, power and minimum
  detectable effect, three placebos, positive and negative engine controls, cost
  stress, era sign agreement, and a sequential test for forward incubation.
- **Verdicts from confidence intervals against a pre-registered minimum useful
  effect**: PASS, INCONCLUSIVE, KILL-CLEAN, KILL-COST, KILL-ERA.
- **A PNG for every test**, in one light house style.

## Requirements

- Python 3.11 or newer, and git
- An AI coding agent that can read files and run commands (for example
  [Claude Code](https://claude.com/claude-code)). The spec is plain Markdown, so any
  capable agent works.
- [Obsidian](https://obsidian.md) to browse the vault (optional). Recommended
  community plugin: **Dataview**. **Templater** is optional.

## Install

1. Make an empty folder for your workspace and put `FRAMEWORK.md` in it:

   ```bash
   mkdir my-research && cd my-research
   curl -O https://raw.githubusercontent.com/qwkw/systematic-research-framework/main/FRAMEWORK.md
   git init
   ```

2. Start your agent in that folder (for example run `claude`) and paste this prompt:

   ```text
   Read FRAMEWORK.md in full before doing anything. You are building and then
   operating the research workspace it specifies.

   1. Ask me the kickoff interview in §0.3 in a single message, and wait for my answers.
   2. Build phases P0 to P9 in order (§0.4). After each phase, run the tests,
      commit, and tell me what passed. Do not start a phase until the previous one
      passes its definition of done.
   3. Follow the hard rules in §0.2 at all times, including after the build:
      never edit the ledger, never read TEST or HOLDOUT outside the protocol, book
      every configuration, never resolve an ambiguity in a result's favour, and
      produce and look at a PNG for every test.
   ```

3. When it finishes, open the folder as a vault in Obsidian (*Open folder as vault*),
   enable the Dataview community plugin, and start from `Home.md`.

## Using it

- Write an idea in today's journal note, or tell the agent: *"Here's my idea, try to kill it."*
- The agent runs the idea gates, fills in a hypothesis note, registers it, and runs
  DEV → controls → placebos → freeze → TEST (once) → verdict, with a PNG verdict sheet
  and a finding note.
- `Home.md` shows the open hypotheses, the verdicts, and the reopen queue.
- Rules change only by an amendment logged in `LEDGER.md`. Amendments apply
  going forward, never to past verdicts.

## Notes

- The framework does not contain or recommend any strategy, and it cannot make a
  bad idea work. It is built to tell you, as cheaply and honestly as possible,
  that an idea doesn't work.
- Cost, fee and rule sheets for venues must come from the venues' own pages, dated.
  The agent will refuse to use guessed values.
- Nothing here is financial advice.
