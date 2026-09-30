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

You do **not** need poppler or Tesseract — both are bundled, and the code picks the right binary for your
platform (`processing.py` appends `.exe` only on Windows).

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

**Expect 563/563 across 23 scripts.** Two of those scripts drive a real Ollama model; without Ollama
installed, run everything else and expect **551/551 across 22 scripts**, which takes about four and a half
minutes:

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
