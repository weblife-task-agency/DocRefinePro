# First run — clone to a branded PDF

Written for someone picking this up cold, on a Mac, with nobody to ask. Everything you need to do a real
rebrand is in the repo; you do not need Drive access, a ClickUp seat or anything from the previous owner.

Budget about 20 minutes, most of which is the test suite running.

## 1. What you need

- **Python 3.10 or 3.11.** The project was built and released on 3.10.11.
- **git.**
- **Ollama is optional.** It powers the classification step. Without it the analyze pass falls back to
  filename rules instead of reading the documents, which is fine for a first run — you will just see weaker
  classifications. Install it later from ollama.com and pull `llama3.2:3b` when you want the real thing.

**On macOS and Linux you DO need poppler and Tesseract installed.** The `poppler/` and `Tesseract-OCR/`
folders in this repo are **Windows binaries only** — 31 `.exe` files and nothing else. `processing.py`
appends `.exe` only on Windows and otherwise resolves the tool from your PATH, so on a Mac it quietly relies
on a system install:

```bash
brew install poppler tesseract
```

*(Corrected 2026-09-30 after Atul Joshi's first run. This section previously claimed both were bundled for
every platform — true only on Windows. His run worked because he already had both from Homebrew, which is
exactly how a wrong assumption survives a successful test.)*

## 2. Set up

```bash
git clone https://github.com/weblife-task-agency/DocRefinePro.git
cd DocRefinePro
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

On Windows that last path is `.venv\Scripts\pip` instead — the suite resolves either, so you do not have to
care beyond this step.

Check the app itself boots headlessly:

```bash
QT_QPA_PLATFORM=offscreen .venv/bin/python main.py --dry-run
```

Exit code 0 and no traceback is a pass. This is the GUI proving it can start — you will not be using it for
rebranding work, see [03-running-headless.md](03-running-headless.md).

## 3. Prove the install with the real test suite

The repo ships its own sample corpus at `samples/` — two PDFs (a 16-page portrait manual with a text layer,
and a 44x34in landscape technical drawing) plus a complete brand kit in both orientations. The suite runs
entirely against it.

```bash
.venv/bin/python verify/run_suite.py "$PWD" "$PWD/samples" /tmp/drp-verify
```

**On a clean clone, expect 537/537 across 22 scripts** without Ollama — about four and a half minutes.

**Fourteen checks do not run on a clean clone, and that is correct, not a fault.** They need the real Batch 4
corpus, which is not in this repo because it is gigabytes and lives on Drive: `verify_pagebox` 7,
`verify_nav` 4, and `verify_v141` 3 (S6-S8, gated behind `REAL.is_dir()`). **They skip silently rather than
announcing themselves** — which is the trap: on a machine that holds the corpus the same run reports
**551/551**, so a number quoted from such a machine is not reproducible anywhere else. Adding Ollama brings
`verify_phase2` back, for 563 on that machine.

Per-script counts from a full Windows run with the corpus present, so you can locate any gap you see:

| Script | Checks | | Script | Checks |
|---|---:|---|---|---:|
| verify_sleep | 38 | | verify_dedup | 16 |
| verify_analyze_route | 8 | | verify_attrib | 16 |
| verify_checkpoint | 25 | | verify_pagebox | 32 *(7 need corpus)* |
| verify_repair | 23 | | verify_nav | 11 *(4 need corpus)* |
| verify_engine | 21 | | verify_v139 | 27 |
| verify_boot | 15 | | verify_v140 | 45 |
| verify_regression | 18 | | verify_v141 | 26 *(3 need corpus)* |
| verify_phase0 | 12 | | verify_v147 | 124 |
| verify_phase1 | 14 | | verify_vision | 25 |
| verify_phase3 | 6 | | **Total, corpus present** | **551** |
| verify_collision | 4 | | **Total, clean clone** | **537** |
| verify_naming | 24 | | | |

The command:

```bash
.venv/bin/python verify/run_suite.py "$PWD" "$PWD/samples" /tmp/drp-verify \
  --only=verify_sleep,verify_analyze_route,verify_checkpoint,verify_repair,verify_engine,verify_boot,verify_regression,verify_phase0,verify_phase1,verify_phase3,verify_collision,verify_naming,verify_keepnames,verify_dedup,verify_attrib,verify_pagebox,verify_nav,verify_v139,verify_v140,verify_v141,verify_v147,verify_vision
```

That figure was confirmed on 2026-09-30 against this exact commit. If you get less, something about your
environment differs — fix it before touching a real batch.

## 4. Do an actual rebrand

Two stages with a human in the middle. That middle step is the whole design, not a formality.

```python
from pathlib import Path
from docrefine.worker import Worker

REPO = Path.cwd()
SRC  = REPO / "samples"            # the two sample PDFs
KIT  = REPO / "samples"            # kit art lives alongside them here
OUT  = Path("/tmp/drp-firstrun")

w = Worker(callback=lambda e: print(e))       # print the events so you can watch it

# Stage 1 — classify, and write a review sheet
w.run_rebrand_analyze(str(SRC))

# Now go and read the sheet it just wrote. It lands in
#   ~/Documents/DocRefinePro_Data/Rebrand Reviews/<parent>__<folder>_rebrand_plan.xlsx
# Each row is a decision: rebrand or leave, plus product / asset type / manufacturer /
# title. Change anything that looks wrong. This is where the real work happens.

# Stage 2 — apply the approved sheet
w.run_rebrand_apply(str(SRC), str(KIT), out_dir=str(OUT))
```

`run_rebrand_apply` also takes `complete_set`, `show_attribution`, `keep_original_names` and `stamp_opts` —
all default off. The playbook explains what each one was for and which delivery turned it on.

### What to look at in the output

Open the branded PDF and check, in this order:

1. **The content is still there.** Not just that a file exists — that the original pages render and the text
   is still selectable. A silently blanked file is the single worst defect this project has had.
2. **The cover title** reads as a short uppercase phrase, not a part number.
3. **Page count** is source + 2 (cover and back cover).
4. **The leave-as-is file is byte-identical** to its source. The landscape drawing should be untouched.

That habit — inspecting the actual deliverable rather than trusting a green suite — is how every real defect
in this project was found. The tests were passing the whole time.

## 5. When you move to a real batch

- **`DRP_DOCS`** tells `verify/batch_paths.py` where batch corpora live. It defaults to your `~/Documents`.
  The per-batch helpers (`apply_batch4.py`, `analyze_vision.py`, `check_imperial.py` and about 17 others)
  still carry absolute paths from the previous owner's Windows machine — they are kept as a **record of what
  was run**, not as scripts you can execute unchanged. `batch_paths.py`, `run_suite.py` and `trial_run.py`
  are the portable ones.
- **Copy `verify/trial_run.py`** as the starting point for a new batch rather than writing from scratch.
- **Author a `brand.json` for every new kit.** Kits arrive from Drive with artwork and no `brand.json`, and a
  kit without one does not fail — it silently falls back to defaults, stamping *Budget Mailboxes* on
  everything regardless of whose job it is. Start from `docrefine/assets/brand.example.json`, save it beside
  the artwork under exactly that filename, and reuse the approved names in `brandkits/`.

## If something does not work

Check [knowledge/known-issues.md](knowledge/known-issues.md) and
[knowledge/clickup-api-quirks.md](knowledge/clickup-api-quirks.md) first — they exist precisely because
several things in this project look broken and are not. Then search the playbook; it is long, but it is
mostly a record of things that went wrong, so the odds are good that whatever surprised you is in there.
