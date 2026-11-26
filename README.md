# Position-Free Time-Window Certificates for Dynamic Programming Search

Code, scripts and results for the paper *Position-Free Time-Window Certificates for Dynamic Programming Search*
(anonymous submission).

 
## Layout

| path | content |
|---|---|
| `src/main.rs` | TSPTW model in RPID with certificate extraction, static enumeration, matching, bound gate, cap and all ablation modes |
| `src/bin/wttw.rs` | single-machine model `1 \| r_j, dl_j \| Σ w_j T_j` with the same certificate machinery |
| `src/clap_lite.rs` | command-line options (`lu_rpid --help`) |
| `vendor/` | RPID 0.4.0 (parallel solvers removed, search unchanged) and the other crates, for offline builds |
| `tools/` | experiment runners, baselines, analysis, proof-log checker, instance preparation |
| `scripts/reproduce.sh` | every experiment of the paper, one block per table or figure |
| `scripts/collect_results.sh` | copies an existing experiment directory into `results/` under the names used here |
| `scripts/make_figs.py` | Figures 1 and 2 |
| `tests/` | small TSPTW and WTTW instances with brute-force optima |
| `results/` | raw results (one JSON line per run) and summaries behind the numbers in the paper |

## Build and test

Requires Rust ≥ 1.90 (edition 2024; the paper's runs used Rust 1.99.1 on Ubuntu 22.04.1) and Python 3 with `numpy`
. All Rust dependencies are vendored.

 
 

## Running the solver

```bash
target/release/lu_rpid INSTANCE --variant learn --time-limit 300               # Online
target/release/lu_rpid INSTANCE --variant learn --static 3 --no-learn --time-limit 300   # Static3
target/release/lu_rpid --help
```

The input format is the one of the Rifki–Solnon instances: `n`, an `n × n` travel-time matrix, then `n` lines
`ready due` (node 0 is the depot). The last line of the output is

```
RESULT {"variant":..., "cost":..., "optimal":..., "infeasible":..., "best_bound":..., "expanded":..., "time":...}
```

followed by `STATS` (certificate counters, including `static_certs`, `static_used` and `t_static`) and `TRACE`
(round and depth at which certificates were learned and used). Static enumeration runs at the first expansion,
inside the timed search, so its cost is part of every reported time.

### Variants in the result files

| name in `results/*.jsonl` | name in the paper | options |
|---|---|---|
| `base`, `base_v9` (WTTW) | Base | `--variant base` |
| `learn`, `learn_v4`, `learn_v5`, `learn_v9` (WTTW) | Online | `--variant learn` (`K=10`, cap 2000, anchor index, bound gate); the tags only mark the run |
| `learn_s2` … `learn_s5` | Static_2 … Static_5 | `--static K --no-learn` |
| `learn_s3l`, `learn_s4l` | Static_k together with Online | `--static K` |
| `cp`, `cp_simpl` | CP, CP_simp | their binary with propagation, without and with simplification, without state cache (see Baselines) |
| `cp_cache`, `cp_simpl_cache` | CP with its cache (setup and WTTW paragraphs) | as above with the state cache |
| `rpid_simpl` | their binary with simplification, without propagation (setup) | |
| `rpid_ref` | their binary without propagation (check of Base) | |
| `learn_locset` | location-bound summary | `--cert-mode locset` |
| `learn_local` | location- and time-bound nogood | `--cert-mode local` |
| `learn_dead` | dead-state memo | `--cert-mode dead` |
| `learn_clear` | no retention across rounds | `--clear-each-round` |
| `learn_locb`, `learn_locb_nocap` | one entry per (subset, location) (text) | `--cert-mode locbound [--cap 0]` |
| `*_astar` | A* as host | `--astar` |
| `learn_k3/k6/k14`, `learn_cap500/cap8000/nocap` | parameter study | `--kmax K`, `--cap C` |
| `learn_nogate_nocap`, `learn_caponly`, `learn_gateonly` | Table 4 | `--no-gate --cap 0`, `--no-gate`, `--cap 0` |
| `learn_locstats2` | cross-location share (counts only) | `--loc-stats` |
 

### Stores of the ablation variants

| variant | entry | query condition | capacity and eviction |
|---|---|---|---|
| Online (`lu`) | `(U, L_U)`, one per subset | `U ⊆ R` and `π_x > L_U(x)` for all `x ∈ U` | 2000 entries; when full, keep the most used half |
| Static_k | `(U, L_U)` for all `2 ≤ \|U\| ≤ k` | as Online | no cap, never evicted |
| location-bound summary (`locset`) | as Online, plus the set of locations at which `U` was extracted | as Online, and the current location is in that set | as Online (counted per subset) |
| one entry per subset and location (`locbound`) | `(U, L_U, location)` | as Online, at that location only | as Online, counted per pair |
| location/time-bound nogood (`local`) | minimized `U` and learning time `t0`, per location | `U ⊆ R` and `t ≥ t0` at that location | per location, 2000 entries, drop oldest half |
| dead-state memo (`dead`) | whole unvisited set and `t0`, per location | `R ⊇ stored set` and `t ≥ t0` at that location | as `local` |

 
 

## Baselines

DIDP with constraint propagation (Marijnissen et al., ICAPS 2026) is built from their Zenodo code archive
(doi:10.5281/zenodo.19051392) with `tools/build_cp.sh ZENODO_DIR BUILD_DIR`, which produces
`RPID-models/target/release/tsptw_rpid` (`CP`) and `wttw_rpid` (`WB`).

* `CP` and `CP_simp` run `--constraint-propagation --no-caching`, with `--simplification-level none` (CP) or the
  default simplification (CP_simp). This is the configuration of the authors' reported experiments: the README of
  their results archive builds all CP results from directories named `*_rpid_cp_no_caching`.
* Block `cpcache` runs the same binaries with their state cache (the default of the binary), which also supplies the
  dual bound derived from propagation, with `--limit-in-mb 12000` so that the cache shrinks before the 16 GB limit.
* `rpid_ref` (no propagation) checks that our Base matches the official model; `rpid_simpl` adds their simplification.

 
 

## License

MIT for our code (see `LICENSE`); the crates in `vendor/` keep their own licenses.
