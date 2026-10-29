# Release notes, build of 2026-10-29

These are the sections of docs/CHANGES_2026-10-27.md that describe this build (sections 8 to 10).

## 8. Rule provenance (2026-10-28)

`stage1/pcie_isolate/rules.py` now ends with `SOURCE`, a table with one entry per rule: *where* the rule is taken from
(document and section) and *why* it is checked (what failure it is evidence of).  The two texts are merged into
`RULES[id]["spec"]` and `RULES[id]["why"]` at import.  `stage1/tests/test_rule_sources.py` checks that every rule has
an entry, every entry names a rule, and every entry names a known document.  Section numbers follow PCIe Base 4.0,
NVMe Base 2.0, NVM Command Set 1.0, SMBus 3.1 and NVMe-MI 1.2; the section title is the stable part across revisions.
`docs/FAQ.html` gained FAQ 30 to 34 (decision card, Settings, synthetic/hook and verdict words, AI reviewer calls and
test generation, finding/defect/error type on an Errors.csv) and a rebuilt step-by-step "findings to defects" widget.


## 9. Isolation without distance assumptions (2026-10-28)

Requested rule: anything in the isolation step that is not grounded in a PCIe/NVMe/SMBus/analyzer specification is
removed.  `stage1/pcie_isolate/incidents.py`:
- grouping: findings join one defect only through SAME_KEY (RequesterID+Tag, PSN, SQID+CID, queue, LBA), REFERS (a
  rule related the two packets) or the same packet; identifier-less findings of one rule and qualifier form a
  counted SERIES.  GAP_PKTS (1000), MAX_REL_SPAN, the 200-packet context attach and H_LEAD_PKTS (50) are gone.
- statistical rules (S.*) never start a defect and are never the root; they are "observation" rows.
- root = earliest specification-level finding with no CAUSES edge; roles root / cascade / independent / series /
  observation / context.
- attribute(): every branch names the specification fact it rests on; the card badge is no longer used as a guess;
  "undetermined" is returned when the wire does not decide; new flow-control branches (PCIe 4.0 §2.6.1).
- evidence rows: exactly the packets the findings name and each finding's own span (bounded at 400 rows), no
  CONTEXT_ROWS padding; the host task is reported as command counts inside the span (no guessed label, TASK_ROWS gone).
- `atlas.py`: card score = signature similarity only (the 0.85/0.15 pattern-prior weighting is gone).
- `engine.py`: early-access findings (CFG.D0_EARLY, CFG.D3_ACCESS, CFG.FLR_EARLY, CFG.MEM_DISABLED_MMIO) carry the
  request's RequesterID+Tag so their completion links to them; a UR/CA completion to a controller-register read also
  emits REG.ALL_ONES (PCIe 4.0 §2.3.2: software reads all ones), keyed and referenced.
- `app/static/js/defect.js`: words for the new roles and for "undetermined".
Remaining engineering values are the engine's MODELED timing thresholds (listed in PROVENANCE_AUDIT.md) and the
statistical rules; they were not changed in this step.

## 10. NVMe commands page (2026-10-29)

New step 3 "NVMe commands" between "Analyse the trace" and "Defects found" (the later steps are renumbered 4, 5, 6).
The page leads with the errors: one row per command that failed, never completed or had a packet go wrong under it,
with chips that name every kind of error found and filter the rows; a folded table under it lists every kind.
Four columns per row (and per kind): the command (admin / I/O, opcode, count, completion status), the parameters it
carries (decoded from the submission entry as NVMe Base 2.0 chapter 5 and NVM Command Set 1.0 section 3.2 define
them, with the LBA data size from Identify Namespace), the measured latency of each step (doorbell to fetch, fetch to
first data, data phase, last data to completion, doorbell to completion) with a label whose basis is stated (host
driver limits 30 s / 60 s; slowest 1 % and 10 % of its own kind in this capture; long by nature), and the low-level
PCIe packets and errors under the command (findings that carry the command's identity or sit on its own packets,
each with the defect it belongs to).  Click a kind for its commands, click a command for its parameters, steps,
pieces (each data packet with its LBA) and errors.  CSV export of every command.
- `stage1/pcie_isolate/engine.py`: one JSON line per ended command (commands.jsonl: steps, packets, times, status,
  the first 32 data packets with address and time, PRP pages, trim ranges); findings raised on a command's data
  packet get the command's identity key; LBA data size from Identify Namespace LBADS (was a fixed 512).
- `stage1/pcie_isolate/commands.py` (new): parameter decoders, status-code names, latency steps and labels, bucket
  tables (edges as in the driver team's Trace Vision), per-command errors, command_summary.json, rows, detail, CSV.
- `stage1/pcie_isolate/cli.py`: subcommand `commands`; the `run` chain and the app runner call it after `isolate`.
- `app/api/runs_api.py`: GET /api/runs/<run>/commands, /commands/rows, /commands/<sqid>/<cid>, /commands.csv.
- `app/static/index.html`, `js/commands.js`, `js/main.js`, `js/run.js`: the page, navigation and step words.
- `samples/sample_nvme_commands_seed21.csv`: a compact hex capture with twelve NVMe-command episodes (generator
  seed 21) for the page and the tests; `stage1/tests/test_commands.py` (17 tests).
Basis of the research: NVMe Base 2.0 (CAP.TO, CRTO, RTD3R/RTD3E, power state ENLAT/EXLAT are the only time budgets
the specification sets; none per command), NVM Command Set 1.0 (DSM access latency is a hint), Linux nvme_core
timeouts; the driver team's Trace Vision recommendations (per-command tables, latency / interval / size buckets,
queue-depth tables, admin command DWORDs, CSV export) from the Confluence pages supplied as pictures.

Management view (same day): a block "Which NVMe commands are causing trouble" at the top of step 3, one line per
kind that went wrong: called / went wrong with the Wilson 95 % interval, failed status / never completed / packet
errors, who the specification points at (NVMe Base 2.0 section 4.2 status code definitions, table WHO_* in
commands.py), the parameter values the failures share (association only; shown when they carry at least half of the
problems, the comparison group has at least 10 commands and the rate is at least 1.5 times the others), the packets
behind them, usual time vs failed time vs the host's limit, defects. A narrative per kind: sentences built from the
facts; with a model, `command_narrative.py` asks the AI reviewer once per kind with the facts only and accepts the
answer only when every number and side is in the facts, no cause is asserted, and it is short (`check()`); the
page prints the source either way. `cli commands --backend/--model`; the runner passes the configured reviewer.
Tests: 24 in `stage1/tests/test_commands.py` (89 stage 1 tests in all).
