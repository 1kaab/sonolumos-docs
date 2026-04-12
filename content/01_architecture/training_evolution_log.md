# Training Evolution Log

## Purpose
Track training decisions and transitions between phases/populations.

## Entry 001
- Base sample: Fin,_Finback_Whale | 61064006.wav
- Seed: 1543664249
- Objective: structure (keep)
- Score: 0.6974
- Next: narrow params to `cap_alpha`, `cap_leak`, `smoothing`
- Seed policy: sweep 1–12

## Applied-to-draft log
Record when a training result is applied to draft for the next evolution step.

Table columns:
- time
- run_id
- objective
- dataset_filter / row_filter
- train_params
- seed policy + range
- score
- sample (species | path)
- applied params (key=value list)

Notes:
- Raw run data lives in `output/training_runs/<run_id>/summary.json` and `results.csv`.
- Applied settings are still reflected in `sonolumos_v3/data/settings.json`.

| time | run_id | objective | dataset_filter / row_filter | train_params | seed policy | score | sample | applied params |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |

## Evolution tree (generated)
This tree is built from `parent_run_id` in `summary.json`.

Generate/update:
```bash
python3 scripts/training_tree.py
```

```mermaid
graph TD
```

## Future UI integration
- Show the evolution tree in Training → Results.
- Click a node to load that run summary/results.
- Color nodes by objective or best score.
