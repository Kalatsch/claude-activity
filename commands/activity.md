---
description: Generate and open a local heatmap of your Claude Code activity
allowed-tools: Bash(python3 *), Bash(open *), Bash(xdg-open *), Bash(test *), Bash(echo *), Bash(cat *)
argument-hint: "[--settings | --help | --no-open]"
---

# /activity

Generate (or update) the local Claude Code activity heatmap dashboard.

## Silence rule (READ FIRST)

Run everything below **silently**. The only chat output allowed is:

- The `AskUserQuestion` UI itself (when the wizard fires).
- The final one-line summary in Step 4.
- The help block in Step 0 (only when `--help` was passed).

**Do not** narrate steps, do not explain what you are about to do, do not
comment on which step you are on, do not announce that the wizard will run,
do not describe how many AskUserQuestion calls you are making, do not
acknowledge "config exists / config missing" — just do it. The user knows
what `/activity` does; they invoked it deliberately.

If something fails (Python missing, generate.py error), report the error in
one line — nothing else.

## Step 0 — Handle `--help` / `-h`

If the user passed `--help`, `-h`, or `help` as the argument, print the
block below verbatim **and stop**. No other text.

```
/activity                  Generate the heatmap and open it in the browser.
/activity --no-open        Generate but do not open the browser.
/activity --settings       Re-run the setup wizard (work days, hours, gap…).
                            Aliases: --setup, --config, --reconfigure
/activity --help           Show this help.
                            Aliases: -h, help

Config file:  ~/.claude-activity/config.json
                gap_minutes, work_intervals, work_days,
                first_day_of_week, cache_read_weight, output_dir
Output:       ~/.claude-activity/index.html (the dashboard)
History:      ~/.claude-activity/history.json (preserves data after Claude
                Code prunes old session JSONL files)

The plugin also runs generate.py automatically on Claude Code SessionStart
and SessionEnd, so `history.json` stays fresh without manual /activity calls.
```

## Step 1 — Locate `lib/generate.py`

`${CLAUDE_PLUGIN_ROOT}/lib/generate.py` when running inside Claude Code.
Otherwise fall back to `../lib/generate.py` relative to this command file.

## Step 2 — First-run setup

Run `test -f ~/.claude-activity/config.json` silently.

- If the file does **not** exist, OR the user passed any of `--settings`,
  `--setup`, `--config`, `--reconfigure` — run the wizard (below).
- Otherwise skip straight to Step 3 (no acknowledgement in chat).

### Setup wizard

Make **one** `AskUserQuestion` tool call with all four questions at once.
No preface, no narration, no "running the wizard now."

1. **Work days** — single-select:
   - `Mon–Fri` (recommended) → `[0,1,2,3,4]`
   - `Mon–Sat` → `[0,1,2,3,4,5]`
   - `All 7 days` → `[0,1,2,3,4,5,6]`
   - `Custom` → ask follow-up multiSelect of MO/TU/WE/TH/FR/SA/SU

2. **Work hours** — single-select. Each preset maps to `work_intervals`
   (array of `[start_inclusive, end_exclusive]` pairs):
   - `09:00–18:00` (default) → `[[9, 18]]`
   - `09:00–13:00, 14:00–18:00` (with 1h lunch) → `[[9, 13], [14, 18]]`
   - `10:00–19:00` → `[[10, 19]]`
   - `08:00–17:00` → `[[8, 17]]`
   - `Custom` → follow-up with **3 questions in one AskUserQuestion call**:
     - **Start hour** (0–23, e.g. 9)
     - **End hour** (0–23, exclusive — e.g. 18)
     - **Lunch break** — single-select: `None`, `12:00–13:00`,
       `13:00–14:00`, `14:00–15:00`, or any other hour-range the user
       wants to specify via the "Other" free-text field.
     Build `work_intervals` from the answers:
       - `None` → `[[start, end]]`
       - lunch `[L, L+k]` → `[[start, L], [L+k, end]]` (split). Drop the
         second interval if `end <= L+k`, or the first if `start >= L`.

3. **Gap threshold** — single-select:
   - `10 min` (recommended), `5 min` (strict), `15 min`, `30 min` (lenient)

4. **First day of week** — single-select:
   - `Monday (ISO)` → `0`, `Sunday (US)` → `6`

Write the answers to `~/.claude-activity/config.json`. `auto_open` is always
`true` in the file (use `/activity --no-open` for one-off silent runs).

```bash
python3 -c "import json,os,pathlib; \
  p=pathlib.Path(os.path.expanduser('~/.claude-activity/config.json')); \
  p.parent.mkdir(parents=True, exist_ok=True); \
  p.write_text(json.dumps($CONFIG_DICT, indent=2))"
```

Substitute `$CONFIG_DICT` with a Python dict literal built from the answers,
e.g.:

```python
{
  "gap_minutes": 10,
  "work_intervals": [[9, 18]],
  "work_days": [0, 1, 2, 3, 4],
  "first_day_of_week": 0,
  "auto_open": True,
  "cache_read_weight": 0.1,
  "output_dir": "~/.claude-activity"
}
```

## Step 3 — Generate

Silently run:

```bash
python3 <path-to-generate.py>
```

(Append ` --no-open` if the user passed `--no-open` on the command line.)
The script writes `~/.claude-activity/index.html` and opens it in the
browser when `auto_open` is true.

## Step 4 — Report

Output exactly one short line, e.g.:

```
85h 6m across 4 months · 1.25B tokens · ~/.claude-activity/index.html
```

If "merged with history" totals significantly exceed "this run" (Claude
Code pruned some logs), add a second line noting the history saved that
data. Otherwise no extra text.

## Notes (for the user — not for you to repeat in chat)

- Change settings later: `/activity --settings` (also `--setup`, `--config`,
  `--reconfigure`).
- Manual config edit: `~/.claude-activity/config.json`.
- Output dashboard: `~/.claude-activity/index.html`.
