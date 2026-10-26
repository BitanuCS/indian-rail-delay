# Project guide: Indian Rail Delay

Your personal map of the project: what every file does, how the pieces connect, and **exactly what to edit and re-run** if you change something.

---

## 1. What the project does

**Question:** a train has just reached a station and is running late. How late will it be at its **final destination**?

**Data:** September 2024, Indian Railways: 1.28 million stop records, 3,892 trains, 4,736 stations.

**Result:** the final model is off by **15.5 minutes on average** on dates it never saw (Sept 26–30).

| Model | Average error (MAE, min) | Within ±10 min |
|---|---|---|
| **Final model** | **15.51** | **75.7%** |
| Historical rule (this train's usual change from this station) | 19.10 | 65.1% |
| Persistence (the delay stays the same) | 25.02 | 52.7% |
| Always on time | 29.83 | 75.3% |

**Keep this table.** If you change something, re-run, and compare with these numbers.

---

## 2. Folder structure

```
Indian Rail Delay/
├── README.md                    front page for GitHub (summary + charts)
├── PROJECT_GUIDE.md             this file
├── requirements.txt             Python libraries the project needs
├── .gitignore                   files git does NOT upload
│
├── 01_data_checks.ipynb         (yours) 8 data quality checks
├── 02_eda.ipynb                 (yours) charts and exploration
├── 03_data_cleaning.ipynb       cleaning
├── 04_feature_engineering.ipynb problem framing, features, split
├── 05_baselines.ipynb           metrics + simple rules to beat
├── 06_model_training.ipynb      5 models, training logs, comparison
├── 07_evaluation.ipynb          final model, test score, error analysis
│
├── reports/                     ON GITHUB
│   ├── figures/                 all charts (used by README)
│   └── test_results.csv         final score table (written by 07)
│
├── data/processed/              NOT on GitHub (recreated by 03 and 04)
├── models/                      NOT on GitHub (recreated by 07)
├── Indian-Railway-Network-and-Delays/       raw data, NOT on GitHub
└── Indian-Railway-Network-and-Delays.zip    backup of raw data, NOT on GitHub
```

### The raw data (`Indian-Railway-Network-and-Delays/`)
| File | What it is | Used? |
|---|---|---|
| `train_routes_delays_Sep2024.csv` | One row per train, date, station: scheduled vs actual time, delay | ✅ main file |
| `train_routes_Sep2024.csv` | Timetable: every stop of every train, distance, scheduled times | ✅ |
| `stations_zones_mapping.json` | Station → railway zone (NR, CR, KRCL…) | ✅ |
| `train_delays_Sep2024.json` | Same delays as the CSV, raw format (`"30H 34M"`) | Only checked once (multi-day delays are real) |
| `IRN_edges.csv` | Network: station-to-station links, distance, number of trains | ❌ not used yet (a "next step") |

### `data/processed/`: handover files between notebooks
| File | Made by | Used by | Contents |
|---|---|---|---|
| `stops.parquet` | 03 | 04 | Cleaned data: one row per train, date, stop |
| `examples_dev.parquet` | 04 | 05, 06 | 23 features + target. History learned from **Sept 1–20** |
| `examples_final.parquet` | 04 | 07 | Same, but history learned from **Sept 1–25** |

`.parquet` = a table file like CSV, but smaller, faster, and it keeps column types (`train` stays text).

### `models/`
| File | Made by | Contents |
|---|---|---|
| `final_model.joblib` | 07 | The trained trees + feature list + baseline formula. Load with `joblib.load(...)` |

**Safe to delete** `data/` and `models/` at any time: re-running 03 → 07 recreates them.

---

## 3. How the notebooks connect

```
raw files ──► 03 ──► stops.parquet ──► 04 ──► examples_dev.parquet ───► 05, 06
                                          └──► examples_final.parquet ─► 07 ──► final_model.joblib
                                                                           └──► reports/test_results.csv
```

**Always run in order 03 → 04 → 05 → 06 → 07.** Each one needs the file saved by the one before. 01 and 02 are independent (they read the raw data directly).

| Notebook | Run time |
|---|---|
| 03 | ~10 s |
| 04 | ~20 s |
| 05 | ~5 s |
| 06 | **~2–3 min** (trains 5 models) |
| 07 | ~30 s |

---

## 4. Each notebook in detail

### 03 · Data cleaning
| Step | What happens |
|---|---|
| 1 | Load delays. `train` read as **text** (keeps leading zeros). Departure columns dropped (copy of arrival) |
| 2 | Load timetable. Rebuild **`sched_min`** = scheduled minutes since leaving the origin (uses `% 1440` per step, so midnight works) |
| 3 | Join delays + timetable (**62 rows dropped**, not in timetable) + zones |
| 4 | Create `trip_id` (= train + date), sort by stop order |
| 5 | **Fake zeros:** a 0 after a stop **≥ 60 min late** (with only zeros between) → set to missing. **47,603 rows (3.7%)** |
| 6 | Mark each trip's destination. **4,043 trips** have no reliable destination delay |
| 7 | Save `stops.parquet` |

New columns: `delay` (cleaned; the original stays in `arr_delay`), `sched_min`, `total_sched_min`, `zone`, `trip_id`, `is_suspect_zero`, `is_destination`.

### 04 · Feature engineering
| Step | What happens |
|---|---|
| 1 | **Target** = delay at the destination, copied to every row of the trip |
| 2 | **Features** (see list below) |
| 3 | Keep usable rows: not the destination, current delay known, target known → **1,099,443 examples** |
| 4 | **Split by date:** train Sept 1–20 (729k) · valid 21–25 (182k) · test 26–30 (188k) |
| 5 | **History features**, out-of-fold (see section 6). Builds two versions: `dev` and `final` |
| 6–7 | Feature table, missing values, target chart (66% of targets are exactly 0) |
| 8 | Save both parquet files |

**The 23 features:**
| Group | Features |
|---|---|
| Delay so far | `cur_delay`, `prev_delay`, `delay_change`, `origin_delay`, `max_delay_so_far` |
| Position | `stop_no`, `stops_left`, `km_done`, `km_left`, `share_done` |
| Timetable | `sched_min_left`, `planned_kmh_left`, `total_km`, `n_stops`, `hour`, `dow` |
| Train/region | `zone`, `train_type`, `is_special` |
| History | `hist_change`, `hist_train_dest`, `hist_station_gain`, `hist_n_trips` |

### 05 · Metrics and baselines
- Defines the metrics (section 7) with a 4-train worked example
- 3 baselines on **valid**: Zero 28.8 · Persistence 23.6 · **Historical 18.0** (the bar to beat)
- Error by stops left: predicting further ahead is harder

### 06 · Model training (all scored on **valid**, Sept 21–25)
| Model | Valid MAE | Notes |
|---|---|---|
| **Residual boosting (final)** | **14.72** | Learns the correction to the historical rule. Best at round 149 |
| Historical baseline | 18.02 | |
| Neural network | 18.42 | 20 epochs. Best RMSE/R², worse MAE |
| Boosting, Poisson loss | 18.65 | |
| Decision tree | 19.80 | depth 12, 512 leaves |
| Linear regression | 20.83 | |
| Boosting, absolute loss | 28.78 | **Collapsed** to predicting 0 (66% of targets are 0) |

### 07 · Final evaluation (on **test**, Sept 26–30, used only here)
- Retrains the final model on Sept 1–25 (150 rounds)
- Test results: section 1
- Error analysis: by stops left, by actual delay (**3+ hour delays underestimated by ~2 h**), by zone (SECR hardest), by train type (special trains hardest, MAE ≈ 40; Vande Bharat easiest, ≈ 5)
- Example journey: Azad Hind Express (12130), 27 Sept
- Permutation importance: `cur_delay` and `hist_change` matter most
- Saves `models/final_model.joblib` and `reports/test_results.csv`

---

## 5. How to change things ⚠️

Use **Cmd+F** in VS Code to find the text in the "Search for" column.

| What you want to change | Notebook | Search for | Then re-run |
|---|---|---|---|
| Fake-zero threshold (now 60) | 03 | `THRESHOLD = 60` | 03 → 07 |
| Train/valid/test dates | 04 | `"2024-09-20"` | 04 → 07 |
| Train-type rules | 04 | `TRAIN_TYPES = [` | 04 → 07 |
| Number of out-of-fold groups (now 5) | 04 | `n_folds=5` | 04 → 07 |
| Add or remove a feature | 04 **and** 06 **and** 07 | `NUMERIC_FEATURES = [` | 04 → 07 |
| Boosting settings (try out) | 06 | `gb_residual = HistGradientBoostingRegressor` | 06 |
| Boosting settings (final model) | **07** | `model = HistGradientBoostingRegressor` | 07 |
| Neural network shape / epochs | 06 | `hidden_layer_sizes` / `EPOCHS = 20` | 06 |
| Baseline formula | 05 **and** 06 **and** 07 | `hist_change` | 05 → 07 |

### ⚠️ Things that are written in more than one place
The notebooks are self-contained, so some code is **copied** between them. If you change one copy, change the others too:
1. **The feature list** (`NUMERIC_FEATURES`, `CATEGORICAL_FEATURES`): in **04, 06 and 07**
2. **The historical baseline** (`cur_delay + hist_change`, clipped at 0): in **05, 06 and 07**
3. **The final model settings** (`max_iter=150, max_leaf_nodes=127, …`): chosen in **06**, typed again in **07**
4. **The `evaluate()` metrics function**: in **05, 06 and 07**

### The golden rule
**Never use the test set to make a decision.** Choose settings by looking at **valid** (notebook 06). Only after that, run 07 once. If you keep changing things until the test score looks good, the test score stops being honest.

### Before you change anything
Write down the current numbers (section 1), make **one** change at a time, re-run, and compare.

---

## 6. Key ideas, in plain words

| Term | Meaning in this project |
|---|---|
| **Target** | What we predict: the delay at the destination |
| **Feature** | An input the model uses (e.g. the current delay) |
| **Example** | One row: one journey seen from one stop |
| **Fake zero** | A missing arrival time that the data recorded as "0 = on time" |
| **Schedule padding** | Extra time in the timetable near the end of a route, so late trains can still arrive "on time" |
| **Split by date** | Learn from early dates, test on later dates. A random split would leak stops of the same journey into the test |
| **Target leakage** | The model secretly using information it wouldn't have in real life. Looks great in testing, fails in reality |
| **History feature** | A statistic from *past* journeys, e.g. "this train usually gains 5 min from here" |
| **Out-of-fold** | For training rows, history comes from the **other** training days only, so a row never sees its own answer |
| **Baseline** | A simple rule with no ML. The model must beat it to be worth anything |
| **Residual model** | prediction = baseline + correction. The model only learns the **correction**. This gave the biggest improvement |
| **Loss function** | What the model tries to minimise while training (absolute error, squared error, Poisson) |
| **Boosting round** | One new small tree added to the model. Similar to an epoch |
| **Epoch** | (Neural network) one full pass over all training data |
| **Overfitting** | Training error keeps falling but validation error stops improving or rises. The model is memorising noise |
| **train / valid / test** | Learn / choose settings / final honest score |

---

## 7. Evaluation metrics

| Metric | Question | Good value |
|---|---|---|
| **MAE** | On average, how many minutes off? **Headline metric** | lower |
| **MedianAE** | How far off is a typical prediction? | lower |
| **RMSE** | Like MAE, but big misses count much more | lower |
| **R²** | Share of variation explained. 1 = perfect, 0 = no better than guessing the average, < 0 = worse | higher |
| **within_10min_%** | How often within ±10 min | higher |
| **within_30min_%** | How often within ±30 min | higher |
| **bias** | Average of (prediction − actual). Negative = predicts too early | close to 0 |

⚠️ Never judge on one metric: "always on time" has a good *within 10 min* (75%) but the worst MAE.

---

## 8. How to run

### Setup (once)
- In VS Code: `Cmd+Shift+P` → **Python: Select Interpreter** → **base** (`/opt/anaconda3/bin/python`)
- In each notebook: top right → **Select Kernel** → **base**
- In the terminal, use **`python`**, not `python3` (`python3` has no scikit-learn)
- `.venv` was deleted. If the terminal still shows `(.venv)`, type `deactivate` or open a new terminal

### Run
Open 03 → 04 → 05 → 06 → 07 and click **Run All** in each, in that order.

### Common errors
| Error | Fix |
|---|---|
| `No module named 'sklearn'` / `'joblib'` | Wrong Python. Choose the **base** kernel / interpreter |
| `FileNotFoundError: ...stops.parquet` or `examples_...parquet` | Run the earlier notebooks first (03, then 04) |
| `FileNotFoundError: ...train_routes_delays_Sep2024.csv` | Raw data folder missing. Unzip the zip into the project root |

---

## 9. Saving to GitHub

```bash
git status                     # see what changed
git add .
git commit -m "Describe what you changed"
git push
```
Repo: `BitanuCS/indian-rail-delay` (private).

Not uploaded (see `.gitignore`): raw data, zip, `data/`, `models/`, `.claude/`, `.DS_Store`.

**Current state:** notebooks 03–07, README, this guide and your Challenges 3–4 in `02_eda` are **not committed yet**.

---

## 10. Known limitations and open points

- **Big delays underestimated:** trains 3+ hours late are predicted ~2 hours too early on average
- **Only one month** of data: no seasonal effects
- **Fake zeros at the start of a trip** can't be detected (no earlier stop to compare with)
- **No congestion, weather or maintenance data**
- `IRN_edges.csv` is not used yet
- In notebook 02, some charts still have no title or `"..."` as a title, and Challenges 5–10 were not done

## 11. Possible next steps
1. Prediction intervals with `loss="quantile"` (e.g. "10–45 min late")
2. Classification: "will it be more than 15 minutes late?"
3. Network features from `IRN_edges.csv`
4. Turn the copied code (section 5) into one shared Python module: a good exercise in **refactoring**
5. A small `predict` function: give train number, station and current delay → get a predicted destination delay
