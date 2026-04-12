# Reaper Feedback Fitness Loop

## Summary
Hybrid selection uses Reaper for lightweight, post‑selection analysis and uses human ratings to learn a fitness score. This avoids full‑library spectral scans while still building a "why" for good combinations.

## Flow (high level)
```mermaid
graph TD
  Index[SQLite index (rms/peak/duration)] --> Query[Candidate query]
  Query --> Select[Run insert]
  Select --> Analyze[Reaper analysis on selected clips]
  Analyze --> Log[Run log + features]
  Log --> Rate[Human rating (1-5 / color)]
  Rate --> Model[Fit fitness model]
  Model --> Query
```

## Data logged (per run)
- Run metadata: timestamp, seed, filters, candidate count.
- Selected clips: path, offset, duration.
- Reaper features (per clip): loudness, dynamics, spectral proxies.
- Triad features: pairwise distances, role balance, diversity score.
- Human rating: ordinal (1..5 or 1..7 colors).

## Reaper analysis sources (post‑selection only)
- Peak cache: quick envelopes, crest factor, fade hints.
- SWS (if installed): LUFS/RMS/peak.
- Optional JSFX: centroid/rolloff/band energy proxies.

## Fitness model (simple + explainable)
Fitness can be a weighted sum of:
- Clip quality (avg loudness, dynamics).
- Triad balance (roles covered, no duplicates).
- Diversity (pairwise distances).
- User rating history (learned weights).

Outputs:
- Score per clip.
- Score per triad (what "works together").
- Explanation from top contributing terms.

## Why this works
- Keeps library indexing lightweight.
- Learns personal taste from explicit ratings.
- Uses Reaper's analysis only when needed.

## Integration points
- 44 Run Insert: writes run log (paths + offsets).
- Post‑run analyzer: computes features + triad scores.
- Trainer: fits weights and outputs a scoring profile used by queries.
