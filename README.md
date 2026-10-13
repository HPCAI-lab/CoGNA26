# CoGNA26: Arithmetic Sensitivity of Kernel Change Detection

Artifact for the paper *Beyond Numerical Error: Arithmetic Sensitivity of Kernel Change Detection in Scientific Data Streams* (CoGNA 2026).

The artifact runs the KCUSUM change detector on the 1H9T molecular-dynamics trajectory in FP64, FP32, FP16, BF16, Posit16, and Posit32. It regenerates every table and figure in the paper.

## Contents

```
src/
  kcusum_posit_full.cpp   KCUSUM in posit<16,2> and posit<32,2> with native Universal types
  kcusum_ablation.cpp     Six-format harness with single-stage ablation (S1 to S4)
scripts/
  xyz_to_meanz.py         Reduce the .xyz trajectory to the per-frame mean-z stream
  run_1h9t.py             1H9T full-format runs, stage ablation, and (delta, h) grid
  center_stream.py        Center the stream on the reference mean (Table 9, second row)
  synth.py                Generate and run the 560 synthetic streams
  analyze_synth.py        Synthetic-stream results and ablation (Table 7)
  figs.py                 Figures 1 to 3
  verify_sweep.py         Recompute every threshold-sweep crossing and compare
  rounding_test.py        Check double-then-round against exact rounding
results/
  published/              Trajectories and threshold sweep reported in the paper
  1h9t/                   Mean-z stream, ablation and grid results, deduplicated sweep
logs/, validation/        Universal build and posit validation logs
paper/                    Manuscript source and figures
```

## Requirements

- A C++20 compiler (tested with GCC 13.3)
- [Stillwater Universal](https://github.com/stillwater-sc/universal) 4.8.0, header-only
- Python 3 with NumPy, pandas, and Matplotlib

## Data

The 1H9T trajectory is not redistributed here. Supply the NWChem trajectory as an `.xyz` file with 8,993 atoms per frame. The script uses the 151 complete frames and drops the incomplete final frame.

## Reproduction

Run every command from the repository root.

**1. Build.**

```bash
git clone https://github.com/stillwater-sc/universal
g++ -O2 -std=c++20 -I universal/include/sw src/kcusum_posit_full.cpp -o kcusum_posit
g++ -O2 -std=c++20 -I universal/include/sw src/kcusum_ablation.cpp -o kcusum_ablation
```

**2. Build the input stream.** This writes 151 frames with mean z between 187.66 and 188.14.

```bash
python3 scripts/xyz_to_meanz.py 1h9t_traj.xyz mean_z.csv
```

**3. Posit trajectories.** This writes `posit16_full.csv` and `posit32_full.csv`.

```bash
./kcusum_posit mean_z.csv
```

**4. 1H9T runs, ablation, and grid (Tables 5, 8, 9).** The `--check` option first compares all six trajectories with `results/published`.

```bash
python3 scripts/run_1h9t.py mean_z.csv --check results/published
python3 scripts/center_stream.py mean_z.csv mean_z_centered.csv
python3 scripts/run_1h9t.py mean_z_centered.csv --out centered
```

**5. Synthetic streams (Table 7).**

```bash
python3 scripts/synth.py
python3 scripts/analyze_synth.py
```

**6. Figures 1 to 3.** These are written to `figures/`.

```bash
python3 scripts/figs.py results/published
```

**7. Checks.**

```bash
python3 scripts/verify_sweep.py results/published h9t_stream.txt
python3 scripts/rounding_test.py
```

## Expected results

| Check | Expected output |
|---|---|
| `run_1h9t.py --check` | FP16, BF16, Posit16, Posit32 differ from the published trajectories by 0. FP32 by at most 1e-15, FP64 by at most 7e-12. |
| `verify_sweep.py` | 0 mismatches against `threshold_sweep_wide.csv`. On the 62 distinct thresholds FP16 and Posit16 differ from FP64 at 38 (30 displaced, 8 detection disagreements). |
| `rounding_test.py` | 0 mismatches for +, -, *, / in FP32, FP16, BF16 |
| Grid agreement, raw stream | FP32 20, FP16 7, BF16 5, Posit16 7, Posit32 20 of 20 |
| Grid agreement, centered | FP32 20, FP16 20, BF16 19, Posit16 20, Posit32 20 of 20 |

`threshold_sweep_wide.csv` lists h = 1.0 twice because of floating-point accumulation. The paper counts the 62 distinct thresholds, which are listed in `results/1h9t/threshold_sweep_distinct.csv`.

## Notes

- The exponential in the Gaussian kernel is computed in FP64 and rounded to the target format in every format, matching the Universal posit `exp()`.
- All posit results are software emulation. They do not measure posit hardware speed or energy.
