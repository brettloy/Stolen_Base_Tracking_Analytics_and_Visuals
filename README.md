# Stolen Base Tracking Analytics and Visuals

SMT Data Challenge 2026: an R pipeline that detects stolen base attempts from MiLB player-tracking data, measures runner speed, jump, catcher pop time, and throw velocity, and models steal outcomes to flag standout plays.

**Team 305:** Brett Loy and Brady Schenck

## Overview

Stolen bases are decided in fractions of a second, but the box score only records the result. This project uses SMT optical tracking data from Minor League Baseball to measure what actually happens on a steal of second base and to turn those measurements into broadcast-style graphics.

The pipeline does three things:

1. **Detects steal attempts** of second base directly from player and ball tracking, rather than relying on play-by-play labels.
2. **Measures both sides of the play:** runner top speed and jump, and catcher pop time (split into exchange and flight time) and throw velocity.
3. **Models Safe%** with logistic regression and surfaces two kinds of fan-facing moments:
   - **Leaderboards:** top-10 plays in each of the four metrics, with the current play highlighted.
   - **Stolen Base Spotlight:** plays where the outcome went against the model, labeled **GOOD SLIDE!** (runner beat a low Safe%) or **GOOD TAG!** (defense beat a high Safe%).

## Repository structure

```
.
├── pipeline/                         Main analysis, run in order
│   ├── SMT_Data_Starter.R            Data loading (Arrow) and helpers from SMT
│   ├── 01_all_steal_outcomes.Rmd     Detect attempts and label outcomes
│   ├── 02_catcher_info.Rmd           Pop time, exchange time, throw velocity
│   ├── 03_runner_info.Rmd            Runner top speed and jump
│   ├── 04_leaderboards.Rmd           Join metrics, build top-10 leaderboards
│   ├── 05_stolen_base_modeling.Rmd   Compare models, fit final Safe% model
│   └── 06_graphic_maker.Rmd          Scoreboard and Spotlight graphics
└── exploratory/                      Earlier iterations kept for reference
    ├── Runner_Modeling.Rmd           Runner-only models (logit, tree, RF, XGBoost)
    └── runner_leaderboards.Rmd       First version of runner metrics and boards
```

## Pipeline

| Step | File | Reads | Writes |
|------|------|-------|--------|
| 1 | `01_all_steal_outcomes.Rmd` | Raw SMT data via `SMT_Data_Starter.R` | `steal_attempts.rds`, `runner_tracks.rds` |
| 2 | `02_catcher_info.Rmd` | `steal_attempts` (in session), ball events and positions | `catcher_info` |
| 3 | `03_runner_info.Rmd` | `steal_attempts.rds`, `runner_tracks.rds` | `runner_info.rds` |
| 4 | `04_leaderboards.Rmd` | `runner_info.rds`, `catcher_info.rds` | `all_leaderboard_triggers.rds`, `steal_modeling_data.rds` |
| 5 | `05_stolen_base_modeling.Rmd` | `steal_modeling_data.rds` | `safe_probability_table.rds`, `model_moments.rds` |
| 6 | `06_graphic_maker.Rmd` | `all_leaderboard_triggers.rds`, `model_moments.rds` | Scoreboard PNGs |

### 1. Steal detection and outcome labeling

Candidate plays come from `lineups`: a runner on first, second base empty, and no pickoff. Runner on third is allowed but flagged. Balls in play and pickoff throws are removed using `ball_events`. A candidate is confirmed as a real attempt using the runner's progress toward second base and **break timing**, the delay between pitch release and the runner committing to a full sprint.

Outcomes are labeled by checking the base state on the next play, grouped within the half inning so labels never cross innings. Walks, hit by pitches, wild pitches, and passed balls are separated from true steals. Caught stealings that end an inning are flagged for review, and ambiguous plays were checked by hand with `animate_play()` before the labels were locked.

### 2. Catcher metrics

Pop time runs from the catcher receiving the pitch to the middle infielder receiving the throw at second. It is split into **exchange time** (catch to release) and **flight time** (release to receipt). Throw velocity is computed from frame-to-frame ball positions during the throw, using the 95th percentile speed to limit the effect of single-frame tracking noise.

### 3. Runner metrics

Runner speed is computed frame by frame. **Top speed** is the highest one-second rolling average, which is steadier than a single peak reading. **Jump** is the runner's speed about 0.4 seconds after pitch release minus their speed at release.

### 4 to 6. Leaderboards, model, and graphics

Runner and catcher metrics are joined into one row per attempt. Logistic regression, a decision tree, and a random forest are compared on a held-out test set using a threshold chosen for balanced accuracy. Logistic regression was selected as the final Safe% model, refit on all labeled plays, and used to generate the Spotlight moments. The graphic maker takes a play and decides whether to show a leaderboard or a Spotlight board.

## Getting started

### Requirements

R 4.x with:

```r
install.packages(c("arrow", "tidyverse", "sportyR", "gganimate", "gifski",
                   "pROC", "rpart", "rpart.plot", "randomForest", "xgboost"))
```

### Data

The tracking data is provided to SMT Data Challenge participants and is **not included** in this repository. Download and unzip it, then set the path at the top of `pipeline/SMT_Data_Starter.R`:

```r
data_directory <- "/path/to/SMT-Data-Challenge-2026"
```

The folder should contain `ball-events/`, `ball-positions/`, `player-positions/`, `lineups.csv`, and `game-info.csv`. The `player-positions` dataset is several gigabytes, so the pipeline filters with Arrow before calling `collect()`.

### Running

Open the `.Rmd` files in `pipeline/` in RStudio and run them in numbered order. Intermediate `.rds` files are written to the `pipeline/` folder and read by later steps.

Note: step 2 uses objects left in the R session by step 1 (`steal_attempts`, `SECOND_BASE`, and the Arrow datasets), so run steps 1 and 2 in the same session. Step 4 reads `catcher_info.rds`, so save it at the end of step 2:

```r
saveRDS(catcher_info, "catcher_info.rds")
```

## Data notes

- Coordinates are in feet. `x = 0` is the line from home plate to second base, and `y = 0` is the back of home plate. Second base sits at (0, 90√2) and first base at (45√2, 45√2).
- `player_id` 11 is the runner on first, 2 is the catcher, and 4 and 6 are the middle infielders who take the throw.
- Timestamps are in milliseconds and each play begins 3 seconds before the pitch is released.

## Limitations

- The labeled sample is small, so model accuracy is modest and should be read as a proof of concept.
- Several caught stealing labels are inferred from base state and inning context rather than observed directly, which adds noise to the negative class.
- The model only uses four tracking features. Pitcher time to plate, lead distance, and slide type are not yet included.

## Acknowledgments

Data and starter code provided by [SMT](https://www.smt.com/) for the SMT Data Challenge 2026.
