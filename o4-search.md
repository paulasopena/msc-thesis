# O4-WCET Pipeline Search Implementation

After the traditional `O1/O2/O3` experiment and the one-pass survey, we added a third experiment: a small search procedure that tries to synthesize a WCET-oriented optimization pipeline. Informally, this is our experimental `O4-WCET`.

The research question is:

```text
Can a short pass sequence, selected using WCET feedback, produce lower LLVMTA WCET bounds than LLVM O1/O2/O3?
```

This experiment is implemented in:

```text
testcases/batch_testing/scripts/search_wcet_pipelines.py
```

### Pass Universe

The broad pass universe comes from:

```text
testcases/batch_testing/scripts/all_passes.csv
```

That file is a CSV-style list of LLVM new-pass-manager passes and their required wrapping kind:

| Kind | Meaning | Example wrapping |
|---|---|---|
| `module` | module pass | `pass-name` |
| `cgscc` | call-graph SCC pass | `cgscc(pass-name)` |
| `function` | function pass | `function(pass-name)` |
| `loop` | loop pass under loop MSSA | `function(loop-mssa(pass-name))` |
| `loopnest` | loop-nest pass | `function(loopnest(pass-name))` |
| `baseline` | marker row for the baseline | no extra pass |

The current file contains 151 usable entries after ignoring the repeated `pass,pass_kind` header row:

| Kind | Count |
|---|---:|
| `function` | 81 |
| `module` | 46 |
| `loop` | 14 |
| `cgscc` | 5 |
| `loopnest` | 4 |
| `baseline` | 1 |

The one-pass WCET and ACET surveys can evaluate this whole list. The O4-WCET search does not try all 151 passes at once by default, because the combination space would explode. Instead, it uses this file to map pass names to valid pass kinds, then chooses a smaller per-benchmark pass pool.

By default, the O4-WCET search seeds that smaller pass pool from the previous one-pass WCET results:

```text
testcases/batch_testing/results/survey30_all_passes_wcet/summary.csv
```

If that seed CSV is available, passes that previously reduced WCET are preferred. If the seed CSV is missing, the script falls back to a curated WCET-oriented list:

```text
scc-oz-module-inliner
jump-threading
loop-rotate
loop-simplifycfg
break-crit-edges
reassociate
loop-reduce
early-cse<memssa>
newgvn
licm
```

The deep search used this command:

```bash
python3 testcases/batch_testing/scripts/search_wcet_pipelines.py \
  --benchmark-file benchmark_configs/survey30.csv \
  --top-passes 10 \
  --max-depth 3 \
  --beam-width 6 \
  -o results/o4_wcet_search_survey30_deep
```

### Search Method

For each benchmark, the script performs this loop:

1. Build linked raw `-O0` LLVM IR as `unoptimized.ll`.
2. Build the same analyzable baseline used by the other experiments.
3. Run LLVM `default<O1>`, `default<O2>`, and `default<O3>` for direct comparison.
4. Select the top candidate passes for that benchmark.
5. Evaluate pass sequences up to `--max-depth`.
6. After each depth, keep only the best bounded candidates according to `--beam-width`.
7. Write every evaluated candidate to `summary.csv`.

The generated candidate pipeline has this shape:

```text
baseline + pass_1 + pass_2 + ... + pass_n + cleanup
```

where the cleanup pipeline is:

```text
function(instcombine, dce), globaldce
```

This means O4-WCET is not a fixed LLVM optimization level. It is a measured search over short pass sequences, using LLVMTA WCET as the objective.

### O4-WCET Output

The main output is:

```text
testcases/batch_testing/results/o4_wcet_search_survey30_deep/summary.csv
```

Important columns:

| Column | Meaning |
|---|---|
| `benchmark` | benchmark name from `survey30.csv` |
| `pass` | generated row label, such as `o4w_jump_threading__loop_rotate` |
| `pipeline` | human-readable pass sequence, such as `jump-threading,loop-rotate` |
| `depth` | number of searched passes in the sequence |
| `status` | LLVMTA result status, usually `ok`, `unbounded`, or `opt-failed` |
| `ub` | LLVMTA upper bound, used as WCET |
| `delta_vs_baseline` | `ub - baseline_ub`; negative means WCET improved vs baseline |
| `delta_vs_o3` | `ub - O3_ub`; negative means the candidate beat O3 |
| `improved_vs_baseline` | yes/no/unknown flag derived from `delta_vs_baseline` |
| `beats_o3` | yes/no/unknown flag derived from `delta_vs_o3` |
| `result_dir` | folder containing LLVMTA artifacts for this candidate |
| `opt_cmd` | exact `opt -passes=...` command used to build the candidate IR |

Each candidate result directory contains the same kind of LLVMTA artifacts as the earlier surveys, including `optimized.ll`, `TotalBound.xml`, command logs, and analysis outputs when produced.

### Deep Search Result Summary

For the deep search over 30 benchmarks:

```text
30 benchmarks
3360 searched candidate rows
3344 ok candidate rows
2818 candidate rows improved WCET vs baseline
263 candidate rows beat O3
246 candidate rows beat the best bounded traditional level among O1/O2/O3
30/30 best searched pipelines improved WCET vs baseline
10/30 best searched pipelines beat O3
10/30 best searched pipelines beat the best bounded traditional level among O1/O2/O3
```

Best searched pipelines that beat O3 in this run:

| Benchmark | Best O4-WCET pipeline | O4 UB | Delta vs O3 | Best traditional level | Delta vs best traditional |
|---|---|---:|---:|---|---:|
| `malardalen_countnegative_smoke` | `jump-threading` | 5975 | -1238 | O1 | -1238 |
| `polybench_atax_smoke` | `loop-rotate,scc-oz-module-inliner` | 7217 | -84 | O2 | -84 |
| `polybench_bicg_smoke` | `loop-rotate,scc-oz-module-inliner,loop-reduce` | 9723 | -168 | O2 | -168 |
| `polybench_jacobi2d_smoke` | `loop-rotate,scc-oz-module-inliner,loop-reduce` | 24473 | -53 | O1 | -53 |
| `polybench_seidel2d_smoke` | `nary-reassociate,scc-oz-module-inliner,loop-rotate` | 24086 | -1698 | O1 | -1698 |
| `synthetic_branchy_table_smoke` | `jump-threading,loop-rotate` | 9890 | -274 | O3 | -274 |
| `synthetic_conflict` | `jump-threading,loop-rotate` | 5724 | -189 | O1 | -189 |
| `synthetic_same_line` | `jump-threading,loop-rotate` | 3242 | -189 | O1 | -189 |
| `tacle_fft_small_smoke` | `scc-oz-module-inliner,reassociate` | 439 | -3428 | O2 | -3428 |
| `tacle_matrix1_smoke` | `loop-reduce,scc-oz-module-inliner,loop-rotate` | 4755 | -390 | O1 | -354 |

This is the strongest evidence so far for Hypothesis 2: pass selection and ordering can produce WCET bounds that are better than LLVM's default optimization levels on a meaningful subset of the benchmark set.

### Explorer Support

The local explorer has an **O4 WCET** tab. It loads:

```text
testcases/batch_testing/results/o4_wcet_search_survey30_deep/summary.csv
```

through the server endpoint:

```text
/api/o4-wcet
```

The O4 tab shows:

- summary cards for candidate counts and wins over O3
- best O4-WCET pipeline per benchmark
- baseline, O1, O2, O3, best traditional, and best O4 side by side
- exact pass sequence responsible for the best result
- detailed candidate rows for inspecting alternative sequences

Run the explorer with:

```bash
python3 tools/llvmta_explorer/server.py --host 127.0.0.1 --port 8790
```

Open:

```text
http://127.0.0.1:8790
```
