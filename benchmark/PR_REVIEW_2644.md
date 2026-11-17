# PR #2644 review fixes

- PR: <https://github.com/intel/neural-compressor/pull/2644>
- Reviewed head: `8ae6450df0` (includes the pre-commit.ci auto-fix commit)
- Scope: 17 inline findings from the Copilot review

All 23 findings (17 in the first review round, 6 in the second) were checked against the code and are valid. The table below
maps each finding to the fix and the verification performed. No benchmark
scripts or state schema were intentionally removed.

## Summary

| # | Severity | File | Finding | Fix |
| ---: | --- | --- | --- | --- |
| 1 | High | `agent_benchmarks/lib/run_state.py` | Failed-run reports leak into aggregates | Aggregate metrics and per-instance counts only from `completed` runs |
| 2 | High | `agent_benchmarks/run_swe_verified.sh` | Retry controller overwrites child failure state | Track `STATE_STARTED`; only finalize state started by this process |
| 3 | High | `evalscope/evalscope_data.py` | First metric row selected instead of primary | Honor `primary_metric_identity` with first-row fallback |
| 4 | High | `evalscope/evalscope_data.py` | Nonzero evaluator exit still marks run completed | Require exit code 0 for `completed`; retain scores as `partial` |
| 5 | High | `evalscope/evalscope_data.py` | Partial runs included in mean/std | Aggregate only `completed` runs; keep incomplete rows for diagnosis |
| 6 | High | `evalscope/run_evalscope_endpoint.sh` | API key stored in `meta.command` | Redact `--api-key`/`--api-key=...` before recording the runner command |
| 7 | High | `humaneval_plus/run_humaneval_plus.sh` | Summarizer exit code not propagated | Capture summarizer status and exit with it; summarizer exits nonzero for partial/failed |
| 8 | High | `ifbench/run_ifbench_suite.sh` | API key stored in per-run state command | Record a redacted copy of the evaluator arguments |
| 9 | Medium | `agent_benchmarks/run_swe_verified.sh` | Harness preflight runs with `--skip-eval` | Check `swebench.harness` only when evaluation is enabled |
| 10 | Medium | `agent_benchmarks/run_swebenchpro.sh` | Docker SDK preflight runs with `--skip-eval` | Keep `pandas` unconditional; check Docker SDK only for evaluation |
| 11 | Medium | `evalscope/run_evalscope_endpoint.sh` | Signal trap does not stop evaluator | Track evaluator PID, terminate/wait in cleanup, exit 130 on INT/TERM |
| 12 | Medium | `humaneval_plus/run_humaneval_plus.sh` | Model discovery is unauthenticated | Send `Authorization: Bearer` on discovery requests |
| 13 | Medium | `humaneval_plus/run_humaneval_plus.sh` | EvalPlus artifact name normalization differs | Match `model.strip("./").replace("/", "--")` |
| 14 | Medium | `humaneval_plus/run_humaneval_plus_independent.sh` | Independent discovery is unauthenticated | Send the bearer token in the discovery wrapper |
| 15 | Medium | `ifbench/run_ifbench_suite.sh` | Relative `--root`/`--workdir` break after `cd` | Resolve both to absolute paths before deriving suite/clone paths |
| 16 | Medium | `ifbench/run_ifbench_suite.sh` | IFBench discovery is unauthenticated | Send the configured API key with model discovery |
| 17 | Medium | `ifbench/run_ifbench_suite.sh` | Repeated rounds reuse seeds when `--repeats > 1` | Advance the base seed by `(round - 1) * repeats` |

## Details

### 1. Exclude failed-run reports from aggregates

- Comment: `discussion_r4597483448`, `agent_benchmarks/lib/run_state.py:253`
- Analysis: valid. A failed run can leave a partially populated `report.json`
  after its first evaluation chunk. The old aggregate loop read every report,
  so a failed run's subset accuracy and instance outcomes were mixed with
  completed full-benchmark runs.
- Fix: `aggregate_records()` now reads a report for diagnosis, but only adds
  metrics and per-instance counts when `run.status == "completed"`.
  `runs_recorded` in `cmd_aggregate()` also counts only completed runs with a
  score. The per-run table still shows failed rows and their available scores.
- Verification: synthetic suite with one completed run (80%, 10 instances) and
  one failed run with a 100% partial report. Aggregate mean stayed 80%,
  `runs_recorded=1`, `failed_runs=1`, and only the completed run's instances
  appeared in the instance list.

### 2. Preserve child failure state in the retry controller

- Comment: `discussion_r4597483528`, `agent_benchmarks/run_swe_verified.sh:117`
- Analysis: valid. The outer `--retry-attempts > 1` process sets `STATE_FILE`
  before its retry loop, but it does not start a new state itself. If a retry
  child fails and the loop exits 0, the EXIT cleanup finalized the child's
  state as `completed`/0.
- Fix: added `STATE_STARTED=false`. It is set to `true` only immediately after
  this process successfully runs `run_state.py start`. The cleanup path
  finalizes `STATE_FILE` only when `STATE_STARTED == true`.
- Verification: code inspection plus `bash -n`; the retry loop now has no path
  that writes a state it did not create.

### 3. Select the report's designated primary metric

- Comment: `discussion_r4597483590`, `evalscope/evalscope_data.py:66`
- Analysis: valid. EvalScope reports can contain several metric rows. The old
  `report_score()` returned the first row containing a known score key, so a
  secondary metric could be published as the task score.
- Fix: `report_score()` now selects the row whose `identity` matches
  `primary_metric_identity` and falls back to the first row only when no match
  exists. This mirrors `eval_summary.py`.
- Verification: synthetic payload with `acc1=0.1` and `acc2=0.9` and
  `primary_metric_identity=acc2` returned `0.9`.

### 4. Require a successful evaluator exit before completing a run

- Comment: `discussion_r4597483661`, `evalscope/evalscope_data.py:198`
- Analysis: valid. `cmd_suite_run_finish()` previously marked a run completed
  whenever all task reports were present, even if EvalScope exited nonzero.
  The suite could then report success despite a failed evaluator.
- Fix: a nonzero evaluator exit produces `partial` when scores exist and
  `failed` otherwise; only exit code 0 plus all task scores produces
  `completed`. Available scores remain in the run for diagnosis.
- Verification: synthetic run with three scores and `--exit-code 1` was stored
  as `partial` with accuracy retained; a zero-exit run was stored as
  `completed`.

### 5. Aggregate scores only from completed runs

- Comment: `discussion_r4597483704`, `evalscope/evalscope_data.py:308`
- Analysis: valid. Partial runs can cover only a subset of tasks, so including
  them in the mean/std makes their scores incomparable with complete runs.
- Fix: `cmd_suite_aggregate()` appends overall and per-task values only for
  `status == "completed"` records. Partial/failed rows remain in `runs` and in
  the markdown table, but not in `metrics`.
- Verification: one completed run (20/30/40%) and one partial run (90/90/90%)
  produced a mean of 30% with `n=1`; the partial row remained visible in the
  per-run table.

### 6. Redact the API key from the recorded runner command

- Comment: `discussion_r4597483759`, `evalscope/run_evalscope_endpoint.sh:10`
- Analysis: valid. `RUNNER_COMMAND` was built from raw `ORIGINAL_ARGS`, so
  `--api-key SECRET` was written to `state.json` as `meta.command`. Per-run
  commands already used `***`, but the suite command did not.
- Fix: the runner now builds a `recorded_args` copy that replaces the value of
  `--api-key` and `--api-key=...` with `***`. `RUNNER_COMMAND` uses that copy;
  the real key is still used only for execution.
- Verification: static inspection; `--api-key` and `--api-key=...` are both
  covered. The recorded per-run command continues to use `recorded_key`.

### 7. Propagate the summarizer's failure exit code

- Comment: `discussion_r4597483807`, `humaneval_plus/run_humaneval_plus.sh:330`
- Analysis: valid. `humaneval_data.py summarize` recorded `partial`/`failed`
  and `exit_code=1` in state but exited 0. The independent runner therefore
  treated an incomplete child with a score as completed.
- Fix: `cmd_summarize()` raises `SystemExit(state["exit_code"])` after saving
  the summary. `run_humaneval_plus.sh` captures that status, sets
  `FINALIZED=true`, and exits with the summarizer's code. The independent
  runner now records the child as failed when the summarizer is incomplete.
- Verification: synthetic one-task result with `--expected 2` produced
  `status=partial`, `exit_code=1`, and the CLI exited 1.

### 8. Redact API keys before recording arguments in state

- Comment: `discussion_r4597483857`, `ifbench/run_ifbench_suite.sh:274`
- Analysis: valid. The evaluator `args` array contains `--api-key ${API_KEY}`;
  serializing it for `suite-run-start` wrote the secret into `state.json`.
- Fix: the runner constructs a `recorded_args` copy with the API key value
  replaced by `***` and records that copy. The real `args` array is unchanged
  for execution.
- Verification: static inspection of the loop; `--api-key` is the only
  credential-bearing argument in the evaluator command.

### 9. Skip the harness preflight when evaluation is disabled

- Comment: `discussion_r4597483912`, `agent_benchmarks/run_swe_verified.sh:354`
- Analysis: valid. `--skip-eval` is documented as generation-only, but the new
  preflight imported `swebench.harness.run_evaluation` unconditionally and
  could fail before generation.
- Fix: the harness import is now guarded by `SKIP_EVAL == false`. Docker and
  mini-extra remain unconditional because generation needs them.
- Verification: `bash -n`; the later existing guard is now consistent with the
  preflight guard.

### 10. Check for the Docker SDK only when evaluation is enabled

- Comment: `discussion_r4597483967`, `agent_benchmarks/run_swebenchpro.sh:309`
- Analysis: valid. The Pro preflight required the Python Docker SDK even for
  `--skip-eval`, although generation only needs `datasets`, `yaml`, `pandas`
  and the Docker CLI.
- Fix: the unconditional import check is now
  `datasets, yaml, pandas`; the `docker` SDK check is guarded by
  `SKIP_EVAL == false`. `require_command docker` remains unconditional because
  generation starts Docker containers.
- Verification: `bash -n`; the check set matches the code paths.

### 11. Terminate the evaluator and exit on interruption

- Comment: `discussion_r4597484021`, `evalscope/run_evalscope_endpoint.sh:233`
- Analysis: valid and reproduced during the 7-round validation. The old
  `trap cleanup EXIT INT TERM` only wrote state; it did not exit the runner or
  stop the foreground `evalscope eval` process, so a later round could start
  and overwrite the interrupted status.
- Fix: the runner tracks `EVAL_PID`, starts EvalScope in the background, and
  waits for it. `cleanup()` sends TERM, waits up to five seconds, sends KILL if
  needed, then writes an `interrupted` state. The INT/TERM trap sets
  `INTERRUPTED=1` and exits 130, which triggers cleanup.
- Verification: a fake evaluator that sleeps was started through the runner;
  sending TERM to the runner produced `status=interrupted`, `exit_code=130`,
  and the evaluator process was gone.

### 12. Authenticate the model discovery request

- Comment: `discussion_r4597484055`, `humaneval_plus/run_humaneval_plus.sh:183`
- Analysis: valid. EvalPlus received `OPENAI_API_KEY`, but the runner's
  `vllm_curl` discovery call did not send it. An API-key-protected vLLM server
  returned 401 before EvalPlus started.
- Fix: `AUTH_ARGS` is derived from `OPENAI_API_KEY` (empty for `EMPTY`) and
  passed to every `vllm_curl` request, including `/v1/models`, health and
  metrics.
- Verification: `bash -n`; the discovery call now uses the same key as
  EvalPlus.

### 13. Match EvalPlus model ID normalization

- Comment: `discussion_r4597484104`, `humaneval_plus/run_humaneval_plus.sh:231`
- Analysis: valid. The old `SERVED_NAME#./` only removed a leading `./`.
  `/models/Qwen` became `--models--Qwen`, while EvalPlus 0.3.1 writes
  `models--Qwen` because it uses `model.strip("./").replace("/", "--")`.
- Fix: the runner strips leading/trailing `.` and `/` with `sed` before
  replacing `/` with `--`, matching EvalPlus.
- Verification: `/models/Qwen` now produces `models--Qwen`.

### 14. Authenticate discovery in the independent runner

- Comment: `discussion_r4597484149`,
  `humaneval_plus/run_humaneval_plus_independent.sh:114`
- Analysis: valid. The independent wrapper had its own unauthenticated
  `vllm_curl` discovery call, even though the child EvalPlus run accepted
  `OPENAI_API_KEY`.
- Fix: the same `AUTH_ARGS` bearer-token handling was added to the independent
  runner's `vllm_curl`.
- Verification: `bash -n`; both HumanEval drivers now share the same discovery
  behavior.

### 15. Resolve root and workdir paths before changing directories

- Comment: `discussion_r4597484189`, `ifbench/run_ifbench_suite.sh:111`
- Analysis: valid. The evaluator changes into `CLONE_DIR`; a relative
  `--root` would place results under the clone, and a relative `--workdir`
  would break `$VENV_DIR/bin/python`.
- Fix: `ROOT` and `WORKDIR` are converted to absolute paths with
  `os.path.abspath` before deriving `CLONE_DIR`, `VENV_DIR`, and `SUITE_DIR`.
- Verification: from `/tmp/ifb_rel`, `--root relroot --workdir relwork`
  produced absolute suite/clone/venv paths under `/tmp/ifb_rel`.

### 16. Send the API key with IFBench discovery requests

- Comment: `discussion_r4597484234`, `ifbench/run_ifbench_suite.sh:115`
- Analysis: valid. `IFBENCH_API_KEY`/`VLLM_API_KEY` was used for generation,
  but `vllm_curl` did not send it to `/v1/models`, causing 401 on protected
  endpoints.
- Fix: `AUTH_ARGS` is set from `API_KEY` and passed to `vllm_curl`, covering
  model discovery, health and metrics.
- Verification: `bash -n`; the discovery call and evaluator now use the same
  key.

### 17. Use distinct seeds across repeated rounds

- Comment: `discussion_r4597484263`, `ifbench/run_ifbench_suite.sh:262`
- Analysis: valid. `evaluate_model.py` uses `seed + repeat_index`. The suite
  advanced the seed by one per round, so with `--repeats 2` round 1 used
  42/43 and round 2 used 43/44, reusing samples.
- Fix: the round seed is now
  `SEED + (round - 1) * REPEATS`, so each round occupies a disjoint seed
  block. The `--seed` help text was updated.
- Verification: static check of the arithmetic; with `--seed 42 --repeats 2`,
  rounds start at 42 and 44.

## Second review round

Six additional findings were raised after the first fixes. All six are valid
and were fixed.

| # | Severity | File | Finding | Fix |
| ---: | --- | --- | --- | --- |
| 18 | Medium | `agent_benchmarks/run_swebenchpro.sh` | Relative `SWEBENCH_PRO_RESULTS_ROOT` breaks paths after `cd` | Resolve the results root to an absolute path before deriving artifacts |
| 19 | Medium | `agent_benchmarks/run_swebenchpro.sh`, `run_swe_verified.sh` | Suite discovery runs before vLLM is ready | Wait for `/health` before discovering the suite model |
| 20 | Medium | `evalscope/run_evalscope_endpoint.sh` | Nested/dotted `reasoning.effort` ignored | Parse `reasoning.effort`, nested `reasoning.effort`, and `default_reasoning_effort` |
| 21 | Medium | `humaneval_plus/humaneval_data.py` | Partial rounds included in accuracy mean/std | Compute accuracy metrics from completed rounds only |
| 22 | Medium | `humaneval_plus/run_humaneval_plus.sh` | EvalPlus descendants survive cancellation | Launch EvalPlus in its own session and TERM/KILL the process group |
| 23 | Medium | `humaneval_plus/run_humaneval_plus.sh` | INT/TERM exits 1 instead of 130 | Exit 130 so the state helper records `interrupted` |

### 18. Resolve the Pro results root before deriving artifact paths

- Comment: `discussion_r4600950040`, `agent_benchmarks/run_swebenchpro.sh:276`
- Analysis: valid. `mini-extra` runs after `cd "${AGENT_DIR}"`, so a relative
  `SWEBENCH_PRO_RESULTS_ROOT` made the runtime config and output paths resolve
  relative to the agent directory instead of the caller's directory. The
  evaluator also changes directories.
- Fix: `RESULTS_ROOT` is resolved with `os.path.abspath` immediately after the
  run tag is sanitized and before `readonly RESULTS_ROOT`. `OUT_DIR`,
  `RUNTIME_CONFIG`, `REPORT_FILE`, and the suite child
  `SWEBENCH_PRO_RESULTS_ROOT` all derive from the absolute root.
- Verification: `bash -n`; the path derivation is now absolute by construction.

### 19. Wait for vLLM readiness before suite model discovery

- Comment: `discussion_r4600950112`, `agent_benchmarks/run_swebenchpro.sh:324`
- Analysis: valid. Both suite functions called `discover_vllm_model` before the
  readiness wait. Starting a suite while vLLM was loading weights failed
  immediately instead of honoring `VLLM_WAIT_TIMEOUT`.
- Fix: `wait_for_vllm "${VLLM_WAIT_TIMEOUT:-300}"` now runs before
  `discover_vllm_model` in both `run_suite()` implementations.
- Verification: `bash -n`; the single-run path already used this ordering.

### 20. Honor nested and dotted `reasoning.effort` metadata

- Comment: `discussion_r4600950205`, `evalscope/run_evalscope_endpoint.sh:175`
- Analysis: valid. The recursive metadata search only matched keys containing
  `reasoning_effort`. It ignored `reasoning: {"effort": "low"}` and the dotted
  key `"reasoning.effort": "medium"`, then silently used `xhigh`.
- Fix: the parser now recognizes `reasoning_effort`,
  `default_reasoning_effort`, `reasoning.effort`, a nested `reasoning.effort`,
  and a string-valued `reasoning` key before falling back to model-name
  inference.
- Verification: a fake `/v1/models` server returned
  `reasoning: {"effort": "low"}` and `"reasoning.effort": "medium"`;
  `--dry-run` printed `reasoning_effort=low` and `reasoning_effort=medium`.

### 21. Compute HumanEval accuracy metrics from completed rounds only

- Comment: `discussion_r4600950281`, `humaneval_plus/humaneval_data.py:189`
- Analysis: valid. A partial sample round can cover fewer tasks than a
  completed round; averaging their accuracies compares different denominators.
  The reported mean/std could be skewed by partial rounds.
- Fix: `accuracy_values` and `base_values` now come from
  `status == "completed"` runs. `recorded_values` still counts every run with a
  score, so partial rows remain visible and `runs_recorded` remains useful for
  diagnosis.
- Verification: synthetic result with one completed run at 100% and one
  partial run at 0% produced `accuracy.mean=1.0`, `runs_recorded=2`, and
  overall `status=partial`.

### 22. Terminate EvalPlus workers and descendants on cancellation

- Comment: `discussion_r4600950373`, `humaneval_plus/run_humaneval_plus.sh:257`
- Analysis: valid. `pkill -P` only signals direct children. EvalPlus workers
  spawn generated-code subprocesses; killing a worker can also remove the
  process that enforces its child timeout.
- Fix: EvalPlus is launched with `setsid`, so it becomes a session/process
  group leader. Cleanup sends TERM to the whole group, waits up to five
  seconds, sends KILL if needed, and reaps the leader.
- Verification: a fake EvalPlus spawned `sleep 300`; sending TERM to the
  runner left both the evaluator and its child stopped.

### 23. Use recognized exit codes for INT and TERM cancellations

- Comment: `discussion_r4600950450`, `humaneval_plus/run_humaneval_plus.sh:272`
- Analysis: valid. The signal handler exited 1, while
  `humaneval_data.py:337` treats only 130/143 as interruptions, so an actual
  cancellation was recorded as `failed`.
- Fix: `termination_requested()` now exits 130, matching the state helper's
  interruption convention.
- Verification: the process-group signal test produced
  `status=interrupted`, `exit_code=130`, and `message=interrupted`.

## Additional CI item: DCO

The DCO check failed because the branch commits did not contain
`Signed-off-by` trailers. The branch was rebased with `git rebase --signoff`;
every commit now carries the author's sign-off
(`lkk12014402 <kaokao.lv@intel.com>`). The pre-commit.ci formatting commit was
folded into the review-fix commit so the history contains only authored
commits.

## Verification performed

- `bash -n` on all modified shell runners.
- `py_compile` on all modified Python helpers.
- `git diff --check`.
- Synthetic `run_state.py` aggregate test (completed + failed partial report).
- Synthetic EvalScope primary-metric, nonzero-exit and partial-aggregate tests.
- Synthetic HumanEval partial-summary exit-code test.
- EvalScope signal test with a sleeping fake evaluator.
- IFBench relative-path dry-run.
- HumanEval model-ID normalization check.
- 7-round smoke validation before the review fixes:
  SWE Verified Mini, SWE-bench Pro 120-subset, and HumanEval+ completed with
  `rc=0`; EvalScope and IFBench were stopped when the review fixes began and
  were rerun against the fixed scripts.
- Second-round checks: fake model metadata for nested/dotted
  `reasoning.effort`, synthetic HumanEval completed/partial metrics, and a
  process-group cancellation test with a fake EvalPlus child.
