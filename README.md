# Can earlier patient transfers help predict new infection clusters?

**Complex Systems and Applications, final project portfolio (Option 1: Hospital infection control)**
Author: marielandreou ([GitHub](https://github.com/marielandreou))

This project models a regional hospital group as a **directed, weighted network** of wards linked by patient transfers. It asks whether the network adds predictive information, beyond two ward attributes, about which wards will develop a new simulated infection cluster. The required comparison gives a clear "yes" on the held-out test set. Four further checks (repeated cross-validation, feature ablation, two placebo controls, and error and threshold analysis) show that the gain is robust, that it depends on **edge direction**, and where the model still fails.

> The data are synthetic teaching data. Nothing here is evidence about real hospitals or suitable for clinical decisions.

---

## 1. Problem statement

At an observation point, 18 of 120 wards already have an active infection cluster. For the **102 wards without one**, can transfers recorded in the **preceding 14 days** help predict whether a **new** cluster is reported in the **following 14 days**, compared with using occupancy and device use alone?

## 2. Data

| File | Rows | Contents |
|---|---|---|
| `hospital_wards.csv` | 120 wards | `ward_id`; `occupancy_ratio` (occupied-bed fraction); `device_use_share` (share of patients with an invasive device); `active_cluster_at_start` (known at the observation point); `new_cluster_later` (target, blank for the 18 wards already affected) |
| `hospital_transfers.csv` | 420 directed links | `source_ward_id` → `destination_ward_id`, `transfer_count` (1–9) in the 14 days before the observation point |

Source: `https://raw.githubusercontent.com/mikeauth/leap/main/hospital` (course data). Copies are included beside the notebook for offline use. Of the 102 eligible wards, **39 (38%) later had a new cluster**.

**Timing rule:** every predictor uses only information available at the observation point. `new_cluster_later` is used only as the target, never to build a feature.

## 3. Method and workflow

1. **EDA:** Restrict to the 102 eligible wards. Compare occupancy and device use by later outcome. Both are higher on average in wards that later had a cluster, but the two groups overlap heavily.
2. **Network:** Build a `networkx.DiGraph` containing all 120 wards, with `transfer_count` as the edge weight. Compute two features for each ward:
   - `incoming_volume`: the total weighted in-degree, meaning all transfers received.
   - `from_active_volume`: transfers received from wards that had an active cluster at the start. This is the ward's recorded *exposure*.
3. **Fair comparison:** One stratified 70/30 split (`random_state=123`), giving 71 training and 31 test wards. Fit `StandardScaler` + `LogisticRegression` in a single pipeline, so scaling is learned from training rows only.
   - **Model A:** occupancy + device use.
   - **Model B:** Model A + both network features.
4. **Evaluation:** Accuracy, F1 for the later-cluster class, ROC AUC from predicted probabilities, and a confusion matrix for each model.
5. **Extension (Part 4):**
   - 5-fold stratified CV repeated 20 times (100 paired folds).
   - Ablation of each network feature.
   - Two placebo controls: transfers *sent to* active wards (direction reversed), and network features shuffled across wards.
   - Analysis of the missed clusters.
   - A decision threshold chosen on **training folds only**.

## 4. Results

### Required comparison (same 31 test wards)

| Model | Accuracy | F1 (later cluster) | ROC AUC | False alarms | Missed clusters |
|---|---|---|---|---|---|
| A: ward attributes | 0.581 | 0.435 | 0.693 | 6 of 19 | 7 of 12 |
| B: + network features | **0.839** | **0.737** | **0.904** | **0 of 19** | **5 of 12** |

All three metrics agree, so this is a clear improvement rather than a mixed result.

![Confusion matrices](figures/03_confusion_matrices.png)

### Robustness, ablation and placebo controls (100 repeated CV folds)

| Feature set | Mean ROC AUC | Gain vs A | Folds beating A |
|---|---|---|---|
| A: ward attributes only | 0.720 | – | – |
| A + incoming volume | 0.862 | +0.142 | 96% |
| A + volume from active wards | 0.864 | +0.144 | 100% |
| **B: A + both network features** | **0.895** | **+0.175** | **98%** |
| Placebo: A + volume *sent to* active wards | 0.739 | +0.019 | 59% |
| Placebo: A + shuffled network features | 0.708 | −0.013 | 34% |

![Repeated CV and coefficients](figures/04_robustness_ablation_placebo.png)

**Key observations**

- **The single-split result is not a fluke.** Model B beats A in 98 of 100 folds.
- **Direction matters.** The reversed-direction placebo performs at chance level relative to A, winning about as often as a coin flip.
- **The gain is not an artefact of adding columns.** Shuffled network features give no gain.
- **Exposure carries the most weight.** In Model B, `from_active_volume` has the largest coefficient: 1.43 log-odds per standard deviation, against 0.37 for `incoming_volume`.

### Where the network model fails

All five clusters Model B missed were in wards with little or no recorded exposure: median exposure 0, against 8 for the clusters it caught. One of them (W004) had been caught by Model A. A threshold chosen on training folds only (0.25) would cut misses on the test set from **5 to 2**, at the cost of **4 false alarms** instead of 0.

![Threshold trade-off](figures/05_threshold_tradeoff.png)

## 5. Interpretation and limitations

On this test set, adding earlier directed transfers clearly improved prediction of new clusters, and repeated cross-validation confirms the gain is stable.

The two network features measure different things:
- **Total incoming volume** measures how many patients a ward receives from anywhere.
- **Volume from initially affected wards** measures exposure to wards that already had a cluster, and it carries most of the signal.

**Limitations:**
- The data are synthetic. The construction note states that the outcome was generated so that both network features matter, so the models are recovering a signal that was deliberately placed in the data.
- A transfer link is not an infection route, and a better score does not show that transfers cause clusters.
- The sample is small (31 test wards), and the validation uses random splits rather than time-ordered ones.
- The model has a systematic blind spot: clusters that arise without recorded exposure.

**Ethics:** If such flags guided infection-control attention, the two kinds of error would harm people in different ways.
- A **false alarm** diverts screening, isolation beds and staff time, and can cause anxiety.
- A **missed cluster** can leave patients and staff unprotected. Because misses concentrate in wards outside the main transfer routes, those wards would be systematically under-protected.

Flags should be reviewed by the infection-prevention team together with ward clinicians, the miss/false-alarm trade-off should be set as an explicit policy, and errors should be audited across ward types.

## 6. Reflection

- **What worked well:** The pipeline made the comparison fair by construction. The placebo controls turned "the score went up" into evidence that the gain depends on *who sent* the patients.
- **What was difficult:** With 31 test wards, one ward changes F1 by about 0.06, so judging the size of the gain required repeated cross-validation. Timing also needed constant care: every feature, including the placebo, had to use only the initial status.
- **What I would improve with more time:**
  - Use time-ordered validation instead of random splits.
  - Try multi-step exposure measures, such as two-step paths or a PageRank-style score.
  - Calibrate the predicted probabilities before using them to rank wards.
  - Validate on independent data with clinicians before any real use.

## 7. Repository contents and how to run

```
├── README.md                                  this file
├── Hospital_Infection_Control_Completed.ipynb completed notebook, all cells run, outputs visible
├── Hospital_Infection_Control_Report.pdf      PDF export of the notebook (code, outputs, answers)
├── Hospital_Infection_Control_Completed.html  HTML export of the notebook
├── hospital_wards.csv, hospital_transfers.csv course data (offline fallback)
├── requirements.txt
└── figures/                                   key plots exported from the notebook
```

To run, install the packages with `pip install -r requirements.txt`, then open the notebook and choose **Kernel → Restart & Run All**. It loads the data from the course URL and falls back to the local CSVs if the URL is unavailable. The full run takes under 30 seconds.
