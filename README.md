# Real Estate Investment Opportunity Analyst — Autorater Evaluation & Skill Hill-Climbing Manual

This repository contains the **Real Estate Investment Opportunity Analysis Agent (`re_analyst`)** built with the Google Agent Development Kit (ADK) and the **Autorater Evaluation Harness (`eval_harness`)** powered by the Vertex AI Gen AI Evaluation Service.

---

## Table of Contents

1. [Architecture & Repository Structure](#1-architecture--repository-structure)
2. [Task 1: Environment & Project Setup](#2-task-1-environment--project-setup)
3. [Task 2: Generate Baseline Agent Evaluation Traces](#3-task-2-generate-baseline-agent-evaluation-traces)
4. [Task 3 & 4: Run Multi-Pass Autorater Evaluation](#4-task-3--4-run-multi-pass-autorater-evaluation)
5. [Task 5: Analyze Autorater Intra-Rater Reliability](#5-task-5-analyze-autorater-intra-rater-reliability)
6. [Task 6: Hill-Climb Agent Skills to Raise Autorater Scores](#6-task-6-hill-climb-agent-skills-to-raise-autorater-scores)
7. [GitHub Push Instructions](#7-github-push-instructions)

---

## 1. Architecture & Repository Structure

```text
autoraters/
├── eval_harness/
│   ├── create_re_agent_evalset.py    # Executes re_analyst over target lots to produce ADK evalset traces
│   ├── run_geap_eval.py              # Dispatches k-pass adaptive rubric evaluations to Vertex AI
│   ├── single_criterion_analysis.py  # Deep-dive Lot ID (N) vs. Pass (k) score matrix & reliability metrics
│   ├── all_criteria_analysis.py      # Summary table of reliability metrics across all 19 rubric criteria
│   ├── explain_criteria_results.py   # Displays LLM judge pass rationales filtered by score threshold
│   ├── run_hillclimb.py              # Automated trace generation + single-pass evaluation loop
│   ├── metrics_calc.py               # Pairwise agreement, Krippendorff's alpha, and matrix marginals
│   ├── rubrics.json                  # 19 Likert-scale (1-5) evaluation criteria definitions
│   └── hillclimb_lots.txt            # Target lot IDs for hill-climbing optimization
├── re_analyst/
│   ├── agent.py                      # Root ADK Agent (`re_opportunity_analyst`) and SkillToolset definition
│   ├── tools/
│   │   ├── lot_tool.py               # `get_lot_data(plot_id)` tool backed by tools_data/lots_database.json
│   │   └── tax_tool.py               # `get_tax_assessment_data(parcel_apn)` tool backed by tax_database.json
│   └── skills/
│       ├── analyze-opportunity/      # Core property & adjacent parcel investigation skill
│       ├── analyze-flood-risk/       # Subject & neighboring property flood zone / hydrological skill
│       └── analyze-tax-implications/ # Acquisition cost-per-acre & monthly tax holding cost skill
├── tools_data/
│   ├── lots_database.json            # Mock county parcel & GIS records
│   └── tax_database.json             # Mock county tax assessor records
├── best_criteria_submission.txt      # Selected high-reliability criteria from Task 5
└── requirements.txt                  # Python dependencies
```

---

## 2. Task 1: Environment & Project Setup

### Step 1.1: Authenticate and Configure Google Cloud
```bash
gcloud auth application-default login
gcloud config set project <YOUR_PROJECT_ID>
```

### Step 1.2: Install Python 3.13 & Dependencies
```bash
python3 -m pip install -r requirements.txt
pip install "google-adk[eval]"
```

### Step 1.3: Configure `.env`
Create a `.env` file in the `autoraters/` root directory:
```ini
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=<YOUR_PROJECT_ID>
GOOGLE_CLOUD_LOCATION=global
MODEL=gemini-3.5-flash
```
*(Tip: If experiencing model capacity constraints, `MODEL=gemini-3.1-flash-lite` can also be used.)*

---

## 3. Task 2: Generate Baseline Agent Evaluation Traces

Run the agent across the 10 target lots in `eval_harness/hillclimb_lots.txt` to generate an ADK evaluation trace dataset:

```bash
cd ~/autoraters
python3 -m eval_harness.create_re_agent_evalset \
  --lots-file eval_harness/hillclimb_lots.txt \
  --out ./re_analyst/hillclimb_lots.evalset.json \
  --agent-dir ./re_analyst
```

To interactively inspect the agent traces in the ADK Web UI:
```bash
adk web --allow_origins "regex:https://.*\.cloudshell\.dev"
```

---

## 4. Task 3 & 4: Run Multi-Pass Autorater Evaluation

### Step 4.1: Dry-Run Validation
Verify the evaluation payload and rubric formatting without calling the Vertex AI Evaluation Service:
```bash
python3 -m eval_harness.run_geap_eval \
  --traces re_analyst/hillclimb_lots.evalset.json \
  --rubric-file eval_harness/rubrics.json \
  --out runs/baseline/geap_calls_and_responses.json \
  --experiment-name rubric-agreement-check \
  --dry-run \
  --k 3
```

### Step 4.2: Execute Multi-Pass Evaluation (`k=3`)
```bash
python3 -m eval_harness.run_geap_eval \
  --traces re_analyst/hillclimb_lots.evalset.json \
  --rubric-file eval_harness/rubrics.json \
  --out runs/baseline/geap_calls_and_responses.json \
  --experiment-name rubric-agreement-check \
  --k 3
```

---

## 5. Task 5: Analyze Autorater Intra-Rater Reliability

Using the large-scale dataset (`runs/geap_calls_and_responses_for_analysis.json`, containing 32 lots graded on 19 criteria across 15 passes):

### Step 5.1: Inspect a Single Criterion Matrix
```bash
python3 -m eval_harness.single_criterion_analysis \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json \
  --criterion environmental-esa
```

### Step 5.2: Compare All 19 Criteria Simultaneously
```bash
python3 -m eval_harness.all_criteria_analysis \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json
```

### Step 5.3: Interpret Metrics & Avoid the Homogeneity Trap
- **Distractor Metrics**: `mean`, `min rating`, `max rating`, `max range pass`, and `sd pass means` do not measure rater consistency on a given report.
- **Homogeneity Trap**: Criteria such as `general-tone`, `logical-flow`, `professionalism`, and `readability` give a score of `5` to every report across every pass (`sd report means = 0`). While `krippendorff-aleph-alpha` reports `alpha = 1` when variance is zero, these criteria fail to distinguish report quality. Similarly, `neighboring-flood-risk` scored `1` on 31/32 baseline reports because the unoptimized agent omitted neighbor flood checks.
- **Best High-Reliability Criteria** (High Krippendorff's Alpha + High `sd report means`):
  1. `monthly-tax-holding` (`alpha = 0.997`, `pairwise %_agree = 99.17%`, `sd report means = 0.8872`)
  2. `flood-zone-flagging` (`alpha = 0.897`, `pairwise %_agree = 94.41%`, `sd report means = 0.3357`)
  3. `environmental-esa` (`alpha = 0.897`, `pairwise %_agree = 79.10%`, `sd report means = 0.7042`)
  *(Other strong candidates: `risk-assessment` [`alpha = 0.803`] and `utility-topography` [`alpha = 0.751`]).*

### Step 5.4: Save and Upload Submission
```bash
echo "monthly-tax-holding,flood-zone-flagging,environmental-esa" > best_criteria_submission.txt
gcloud storage cp best_criteria_submission.txt gs://<YOUR_BUCKET>/best_criteria_submission.txt
```

---

## 6. Task 6: Hill-Climb Agent Skills to Raise Autorater Scores

### Step 6.1: Run Initial Hill-Climb Baseline (`hillclimb_1`)
```bash
python3 -m eval_harness.run_hillclimb
```
- **Baseline Results (`hillclimb_1`)**:
  - `cost-per-acre`: **1.5000**
  - `neighboring-flood-risk`: **1.4000**

### Step 6.2: Diagnose Deficiencies Using Judge Rationales
```bash
python3 -m eval_harness.explain_criteria_results \
  --run_geap_eval_out runs/hillclimb_1/geap_calls_and_responses_hillclimb_1.json \
  --criterion cost-per-acre \
  --max_score_to_show 4.0

python3 -m eval_harness.explain_criteria_results \
  --run_geap_eval_out runs/hillclimb_1/geap_calls_and_responses_hillclimb_1.json \
  --criterion neighboring-flood-risk \
  --max_score_to_show 4.0
```
- **Root Causes Identified**:
  1. **`cost-per-acre`**: The agent reported `last_sale_price` and `acreage` separately without explicitly writing out the derivation formula (`Last Sale Price / Acreage`), computing the exact dollar-per-acre figure, or highlighting/benchmarking it as a core valuation metric.
  2. **`neighboring-flood-risk`**: The agent did not call `get_lot_data` for the lot IDs in `adjacent_parcels` and therefore never reported neighboring properties' specific FEMA flood zone designations or analyzed cross-boundary drainage, runoff, and hydrological impacts.

### Step 6.3: Enhance Agent Skills in `re_analyst/skills/`
Updated the three skill definitions:
- `re_analyst/skills/analyze-opportunity/SKILL.md`: Added mandatory steps to call `get_lot_data` for all `adjacent_parcels`, compute and highlight exact `Acquisition Cost-Per-Acre = Last Sale Price / Acreage`, benchmark against assessed value per acre and neighbor cost-per-acre, and synthesize a comparative hydrological table.
- `re_analyst/skills/analyze-flood-risk/SKILL.md`: Added `get_lot_data` to `adk_additional_tools` and instructed the agent to report every neighbor's `plot_id`, FEMA `flood_zone`, `topography`, `soil_classification`, and `easement_type`, plus analyze overland runoff, floodplain continuity, tailwater/backwater risk, and zero-rise mitigation.
- `re_analyst/skills/analyze-tax-implications/SKILL.md`: Added mandatory derivation and benchmarking of `Acquisition Cost-Per-Acre` alongside `Monthly Tax Holding Cost = Annual Tax Amount / 12`.

### Step 6.4: Re-Run Hill-Climb Evaluation (`hillclimb_2`)
```bash
python3 -m eval_harness.run_hillclimb
```
- **Final Results (`hillclimb_2`)**:
  - `cost-per-acre`: **5.0000** (Target: > 4.0)
  - `neighboring-flood-risk`: **5.0000** (Target: > 4.0)

---

## 7. GitHub Push Instructions

Authenticate with GitHub and push this repository:

```bash
# Option A: Using GitHub CLI (creates repo and pushes automatically)
gh auth login
gh repo create autoraters --public --source=. --remote=origin --push

# Option B: Using an existing GitHub repository URL and Personal Access Token (PAT)
git remote add origin https://github.com/<YOUR_GITHUB_USERNAME>/<YOUR_REPO_NAME>.git
git branch -M main
git push -u origin main
```
