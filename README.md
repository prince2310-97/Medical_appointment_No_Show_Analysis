# Medical Appointment No-Show Analysis
**Operational Insights for Healthcare Scheduling Optimization**

---

## Executive Summary

This exploratory data analysis examines **110,527 medical appointments** from a Brazilian healthcare system to identify operational patterns associated with patient no-shows. The analysis tests age, SMS reminders, wait time, chronic conditions, socioeconomic status (scholarship), day-of-week, and neighbourhood — all validated with chi-square testing.

**Headline finding:** SMS reminders are associated with a *lower* no-show rate once no-show rates are stratified by wait-time category — but the raw, unconditional comparison shows the opposite direction, because same-day appointments (35% of the dataset) never receive an SMS and have a naturally near-zero no-show rate. This confound is documented in detail below; see "SMS Effectiveness — The Full Picture."

**Business Impact:** Data-driven, appropriately-hedged guidance to optimize reminder strategy, resource allocation, and scheduling — framed as operational hypotheses for a pilot, not proven causal effects (see Limitations).

---

## Business Context

### Healthcare Operations Challenge
Modern healthcare systems face significant operational inefficiencies caused by **patient no-shows**:
- **Unused capacity** – Idle clinician time and facility resources
- **Workflow disruption** – Scheduling delays for other patients
- **Revenue loss** – Uncompensated appointment slots
- **Resource misallocation** – Equipment and staff scheduled for no-show cases

This analysis addresses a fundamental operational question: **Which patient segments and scheduling patterns are most strongly associated with no-shows?**

### Why This Matters
Understanding no-show patterns enables healthcare operations teams to:
- Design targeted reminder protocols for high-risk appointments
- Optimize overbooking strategies
- Allocate outreach resources by neighbourhood and segment
- Improve patient communication and engagement

---

## Problem Statement

1. **Demographic No-Show Patterns** – Do certain age groups have systematically higher no-show rates?
2. **Scheduling Window Impact** – How does the time between scheduling and appointment date affect attendance?
3. **Reminder Effectiveness** – Is SMS receipt associated with better attendance, and does that hold once no-show rates are stratified by wait-time category?
4. **Chronic Disease & Socioeconomic Correlation** – Do chronic conditions or scholarship (subsidy) status relate to attendance?
5. **Scheduling Pattern Influence** – Do day-of-week or neighbourhood show meaningful variation in no-shows?

**Core Question:** What operational factors within our control show a measurable, robust association with appointment no-shows?

---

## Objective of Analysis

1. **Identify High-Risk Segments** – Determine which demographics and scheduling patterns correlate with higher no-show rates
2. **Test the SMS Association Rigorously** – Check whether the SMS/no-show relationship holds once wait time is accounted for, since the two are correlated by design in this dataset
3. **Generate Actionable Recommendations** – Provide data-driven, appropriately-hedged guidance for operations teams

### Analysis Scope
- **Exploratory approach** – Understanding patterns through visualization and summary statistics
- **Statistical validation** – Chi-square tests for every reported association
- **No predictive modeling** – Operational diagnostics, not a forecasting project

---

## Dataset Description

### Source
- **Dataset:** Kaggle Medical Appointment No-Show (Brazil, 2016)
- **Records:** 110,527 medical appointments (raw); **110,519 after cleaning**
- **Coverage:** Brazilian healthcare system appointments with appointment status outcomes
- **Availability:** Publicly available on Kaggle (educational use)

### Key Features

| Column | Description | Data Type |
|--------|-------------|-----------|
| **PatientId** | Unique patient identifier | Float |
| **AppointmentID** | Unique appointment identifier | Integer |
| **ScheduledDay** | Date appointment was booked | DateTime |
| **AppointmentDay** | Date of scheduled appointment | DateTime |
| **Age** | Patient age in years | Integer |
| **Neighbourhood** | Healthcare facility location | Categorical (81 values) |
| **Scholarship** | Presence of social scholarship/subsidy | Binary (0/1) |
| **Hipertension** | Hypertension (renamed `Hypertension` after cleaning) | Binary (0/1) |
| **Diabetes** | Diabetes diagnosis | Binary (0/1) |
| **Alcoholism** | Alcoholism diagnosis | Binary (0/1) |
| **Handcap** | Disability status, 0–4 (renamed `Handicap`) | Categorical |
| **SMS_received** | Patient received SMS reminder | Binary (0/1) |
| **No-show** | Patient did not attend (renamed `No_show`) | Binary (0/1) |

### Data Characteristics
- **No Missing Values** – 0 nulls across all columns
- **Target Variable Distribution** – 20.19% no-show rate
- **Categorical Richness** – 81 neighbourhoods; 5 handicap levels (0–4)
- **Temporal Spanning** – Appointments scheduled April–June 2016
- **34.9% of appointments are same-day** (scheduled and appointment date identical) — this turns out to be operationally important (see SMS section)

---

## Data Cleaning & Preparation

- Parsed `ScheduledDay`/`AppointmentDay` from ISO strings to dates
- Renamed `Hipertension`→`Hypertension`, `Handcap`→`Handicap`, `No-show`→`No_show`
- Filtered to `0 <= Age <= 100`: **110,519 records retained** (99.99%)
- Found 5 rows where `AppointmentDay` preceded `ScheduledDay`; rather than dropping them, the resulting negative wait time is clipped to 0
- Engineered features:
  ```python
  df['days_diff'] = (AppointmentDay - ScheduledDay).dt.days, clipped at 0
  df['Age_group'] = pd.cut(Age, bins=[0,18,35,60,101], labels=['0-17','18-34','35-59','60+'])
  df['wait_category'] = pd.cut(days_diff, bins=[0,7,14,365], labels=['0-7 days','8-14 days','15+ days'])
  df['Appointment_weekday'] = AppointmentDay.dt.day_name()
  df['has_chronic_condition'] = Hypertension==1 or Diabetes==1 or Alcoholism==1
  ```

---

## Key Findings

### 1. SMS Effectiveness — The Full Picture (most important finding)

**Unconditional comparison (misleading on its own):**

| SMS Received | No-show Rate |
|---|---|
| No | 16.70% |
| Yes | 27.58% |

Taken at face value, this looks like SMS *increases* no-shows. **It doesn't — this is a confound.** 34.9% of all appointments are booked same-day, and same-day appointments structurally never receive an SMS (there's no lead time to send one) *and* have a naturally low 4.66% no-show rate (the patient is essentially already there when they book). This drags the "No SMS" average down and makes the comparison misleading.

**Stratified by wait-time category (apples-to-apples):**

| Wait Category | No-show, No SMS | No-show, SMS Sent | SMS Effect |
|---|---|---|---|
| 0–7 days | 24.36% | 23.76% | -0.6pp |
| 8–14 days | 33.76% | 28.11% | **-5.6pp** |
| 15+ days | 37.10% | 29.95% | **-7.2pp** |

**Once wait time is held constant, SMS is associated with a meaningfully lower no-show rate — and the effect grows with wait time.** This is the correct, defensible reading of the SMS relationship in this dataset, and it reverses the naive/unconditional comparison. Chi-square on the unconditional SMS/no-show relationship: χ²=1767.11, p<0.0001 (significant, but see above for why the direction alone is not enough).

### 2. Age Group

| Age Group | No-show Rate |
|---|---|
| 0–17 | 21.90% |
| 18–34 | 23.98% (highest) |
| 35–59 | 19.26% |
| 60+ | 15.30% (lowest) |

Chi-square: χ²=599.93, p<0.0001 — significant. Younger adults (18–34) are the highest-risk age group; attendance improves steadily with age.

### 3. Wait Time

Chi-square: χ²=567.42, p<0.0001 — significant. Longer wait categories are associated with higher no-show, independent of the SMS analysis above.

### 4. Chronic Condition

| Has Chronic Condition (Hypertension/Diabetes/Alcoholism) | No-show Rate |
|---|---|
| No | 20.91% |
| Yes | 17.77% |

Chi-square: χ²=118.53, p<0.0001 — significant. Counter-intuitively, patients with a chronic condition show up *more* reliably, not less — plausibly because they're more engaged with ongoing care.

### 5. Scholarship (Socioeconomic Subsidy)

| Scholarship | No-show Rate |
|---|---|
| No | 19.81% |
| Yes | 23.74% |

Chi-square: χ²=93.65, p<0.0001 — significant. Patients on social subsidy no-show more often, consistent with a socioeconomic access barrier (transport, work schedule flexibility, etc.).

### 6. Day of Week

| Day | No-show Rate |
|---|---|
| Monday | 20.65% |
| Tuesday | 20.09% |
| Wednesday | 19.69% |
| Thursday | 19.35% (lowest) |
| Friday | 21.23% |
| Saturday | 23.08% (highest) |

Chi-square: χ²=27.62, p<0.0001 — statistically significant, but the smallest chi-square value of all tests run here. This is a good illustration of the difference between statistical and operational significance: with 110K+ records, even a modest, low-impact pattern like this one clears the p<0.05 bar.

### 7. Neighbourhood

Filtered to neighbourhoods with at least 100 appointments for reliability (81 total neighbourhoods):

**Highest no-show rate:** Santos Dumont (28.92%, n=1,276), Santa Cecília (27.46%, n=448), Santa Clara (26.48%, n=506)

**Lowest no-show rate:** Mário Cypreste (14.56%, n=371), Solon Borges (14.71%, n=469), De Lourdes (15.41%, n=305)

A ~14-point spread between the best and worst high-volume neighbourhoods — worth investigating for local operational or transport factors.

### 8. Gender

Chi-square: χ²=1.83, p=0.1758 — **not statistically significant.** No evidence to treat genders differently in reminder or scheduling strategy.

---

## Statistical Validation Summary

| Test | χ² | p-value | Result |
|------|-----|---------|--------|
| SMS Received vs. No-show (unconditional) | 1767.11 | <0.0001 | Significant — but see confound above |
| Age Group vs. No-show | 599.93 | <0.0001 | Significant |
| Wait Category vs. No-show | 567.42 | <0.0001 | Significant |
| Chronic Condition vs. No-show | 118.53 | <0.0001 | Significant |
| Scholarship vs. No-show | 93.65 | <0.0001 | Significant |
| Day of Week vs. No-show | 27.62 | <0.0001 | Significant (small effect size) |
| Gender vs. No-show | 1.83 | 0.1758 | **Not significant** |

**Interpretation rule:** p < 0.05 → association unlikely due to chance. With ~110K records, even small/trivial effects can register as "significant" — statistical significance is not the same as operational significance (the Day-of-Week result is the clearest example of this in this dataset). Note also that the χ² statistic itself is not a clean strength-of-association measure — it scales with sample size, so a larger χ² does not necessarily mean a more operationally important effect. Use the p-value for significance and the actual rate differences (%) for practical importance.

---

## Key Insights

- **SMS reminders work — but only once you control for wait time.** The naive comparison is backwards due to the same-day-appointment confound; within any given wait-time band, SMS recipients show up more reliably, and the benefit grows with wait time (up to 7.2 percentage points at 15+ days)
- **Young adults (18–34) are the highest-risk age group** (23.98% no-show); attendance improves steadily with age, bottoming out at 15.30% for 60+
- **Longer wait times are independently associated with higher no-shows**, on top of the SMS effect above
- **Chronic-condition patients attend more reliably** (17.77% vs. 20.91%) — likely reflects higher engagement with ongoing care, not a risk factor
- **Scholarship (subsidy) recipients no-show more** (23.74% vs. 19.81%) — a socioeconomic access signal worth a targeted intervention
- **Saturday has the highest no-show rate, Thursday the lowest** — real but the weakest of all the significant patterns found
- **Neighbourhood matters** — a ~14-point gap between the best and worst high-volume neighbourhoods
- **Gender shows no meaningful association** — don't build gender-specific protocols on this data

---

## Operational Recommendations

> Framed as pilot hypotheses, not guaranteed outcomes — this is a correlational analysis (see Limitations). No specific "expected % impact" figures are quoted, since none were derived from a controlled trial or model here.

1. **Expand SMS coverage for longer-wait appointments specifically** — the data shows the SMS benefit is largest at 15+ day waits (7.2pp) and smallest for near-term bookings (0.6pp at 0-7 days). Prioritize SMS budget accordingly rather than blanket coverage.
2. **Target reminder/engagement strategy at young adults (18–34)** — the single highest-risk age segment found.
3. **Reduce long booking windows where feasible** — wait time is an independent, significant driver of no-shows on top of the SMS effect.
4. **Investigate the top 3 highest-no-show neighbourhoods** (Santos Dumont, Santa Cecília, Santa Clara) for local transport/access barriers, using the lowest-no-show neighbourhoods as a benchmark.
5. **Consider a transportation or scheduling-flexibility pilot for scholarship (subsidy) patients** — the ~4pp gap is consistent with an access barrier rather than a motivation issue.
6. **Do not build gender-specific protocols** — not supported by this data.
7. **Treat the Day-of-Week pattern as low-priority** relative to the other findings — it's statistically real but the smallest effect measured here.

---

## Assumptions

1. **SMS Message Quality & Timing** – Assumption: SMS sent at reasonable lead time. Reality Check: verify actual send timing before assuming the wait-time-stratified effect above generalizes.
2. **Data Capture Accuracy** – Assumption: no-show status accurately recorded. Reality Check: verify late arrivals aren't misclassified.
3. **Temporal Stability** – Assumption: 2016 patterns still hold today. Reality Check: transportation, communication habits, and post-2016 events may have shifted behavior.
4. **SMS as Partial Marker** – Even after stratifying by wait-time category, SMS recipients may differ from non-recipients in other unobserved ways (e.g., contact information on file, prior engagement) — this stratification addresses the largest known confound, not necessarily all of them.

---

## Limitations

### Data & Scope Constraints
1. **Single Geographic Region** – Brazilian healthcare system (2016); findings may not generalize elsewhere or to later years.
2. **Correlational, Not Causal** – Even the wait-time-controlled SMS result is an association, not a proven causal effect from a controlled experiment.
3. **Missing Operational Variables** – No appointment duration, procedure type, or patient distance from facility.
4. **Temporal Snapshot** – Cross-sectional; cannot measure before/after impact of any intervention.
5. **Cannot Quantify Individual Risk** – Group-level insights only; cannot predict which specific patient will no-show.

### Analytical Approach Limitations
6. **Bivariate Analysis Only** – Each factor (age, SMS, wait time, chronic condition, scholarship, day, neighbourhood) is tested against no-show individually or in a single 2-way control (SMS × wait time); a multivariate model controlling for all factors simultaneously has not been built.
7. **Chi-Square Sample-Size Sensitivity** – With 110K+ records, even small effects register as statistically significant — the Day-of-Week result (smallest χ² of all significant tests) is the clearest illustration of this in this analysis.
8. **Neighbourhood Analysis Is Associational Only** – The 81-neighbourhood comparison doesn't control for the demographic or socioeconomic composition of each neighbourhood, so part of the spread may reflect age/scholarship mix rather than a purely geographic effect.

---

## Tools Used

| Tool | Version | Purpose |
|------|---------|---------|
| **Python** | 3.7+ | Data analysis programming language |
| **pandas** | 1.x | Data loading, cleaning, feature engineering |
| **NumPy** | 1.x | Numerical calculations |
| **matplotlib** | 3.x | Static visualizations and charts |
| **seaborn** | 0.11+ | Statistical visualization and aesthetics |
| **SciPy** | 1.x | Chi-square statistical testing |
| **Jupyter Notebook** | - | Interactive analysis environment |

---

## How to Run the Notebook

### Prerequisites
- Python 3.7 or later
- Jupyter Notebook or JupyterLab

### Setup Instructions

1. **Navigate to Project Directory**
   ```bash
   cd medical-appointment-no-show-analysis
   ```

2. **Install Required Packages**
   ```bash
   pip install pandas numpy matplotlib seaborn scipy jupyter
   ```
   Or from requirements.txt:
   ```bash
   pip install -r requirements.txt
   ```

3. **Verify Data Files** — ensure `Data.csv` is in the same directory as the notebook

4. **Launch Jupyter Notebook**
   ```bash
   jupyter notebook medical_appointment_no_show_eda.ipynb
   ```

5. **Run Cells Sequentially** — Cell → Run All, or step through one-by-one for inspection

### Troubleshooting

| Issue | Solution |
|-------|----------|
| **FileNotFoundError: Data.csv not found** | Verify Data.csv is in same folder as notebook |
| **ModuleNotFoundError: pandas** | Run `pip install pandas` in terminal |
| **Kernel not responding** | Restart kernel: Kernel → Restart Kernel |
| **Visualizations not showing** | Ensure `%matplotlib inline` is in first code cell |

---

### File Descriptions

| File | Description |
|------|-------------|
| **README.md** | Project overview, methodology, insights, and instructions |
| **requirements.txt** | Python dependencies |
| **Data.csv** | 110,527 healthcare appointments from Brazil (2016); publicly available from Kaggle |
| **medical_appointment_no_show_eda.ipynb** | Jupyter notebook: data cleaning, feature engineering, visualizations, statistical tests |

### Notes
- This project is **notebook-first**: all analysis is contained in the Jupyter notebook
- Reproducible: running the notebook top to bottom regenerates all outputs

---

## Author & Project Info

**Project Title:** Medical Appointment No-Show Analysis
**Type:** Exploratory Data Analysis (Operational Insights)
**Dataset:** Kaggle Medical Appointment No-Show (Brazil, 2016)
**Focus:** Healthcare operations optimization, not predictive modeling

**Suitable For:**
- Healthcare Operations teams
- Data Analyst portfolios
- Healthcare analytics interviews

---

## License & Attributions

**Data Source:** [Kaggle - Medical Appointment No-shows](https://www.kaggle.com/joniarroba/noshowappointments) (Public Dataset)

**Original Dataset Credit:** Kaggle Community
**Analysis & Report:** Original work for portfolio purposes
