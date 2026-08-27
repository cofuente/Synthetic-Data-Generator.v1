# 🎯 Project Goal

Build a production-grade synthetic data pipeline that generates, validates, and analyzes field-task / robot-capability pairs using LLMs. Your system will act as an intelligent agricultural robotics procurement analyst that can identify capability mismatches, detect overclaiming, and provide actionable feedback.

**Core Challenge**: Create a system that not only generates realistic data but also understands what makes a robot capability dossier "good" or "bad" for a specific field task, then prove it works through rigorous evaluation.

***

## 🧠 The Problem Context

A grower co-op planning next season publishes field task requisitions: *selective strawberry harvest across 12 ha of polytunnel, June–September, must tolerate occlusion, minimum conditional autonomy.* Robotics vendors and system integrators respond with capability dossiers for their VLA-equipped platforms.

Those dossiers are sales documents. Vendors routinely claim capabilities their policies do not have — "handles any crop, any field condition, zero supervision required." A procurement team receives dozens per requisition. Most are poorly matched: wrong capabilities, mismatched autonomy levels, or filled with agtech marketing jargon and fabricated deployment history.

Your task: Build an AI system that can:

* **Generate** realistic task-dossier pairs with varying quality levels
* **Validate** that data follows strict structural rules
* **Analyze** why a dossier fails to match a task (capability gap? autonomy mismatch? overclaiming?)
* **Visualize** patterns in failures across different scenarios
* **Correct** invalid data through iterative LLM feedback
* **Serve** this intelligence via a REST API for real-time analysis

***

## 🔀 Domain Crosswalk

This project is a domain adaptation of a resume/job-description analysis pipeline. Every technique, metric, threshold, and success criterion is carried over unchanged; only the domain, example data, and identifiers differ. This table is the authoritative mapping — use it whenever you need to check that an adapted requirement still corresponds to its original.

| Original identifier | This project | Kind |
| --- | --- | --- |
| `Resume` | `RobotCapabilityProfile` | Pydantic model |
| `JobDescription` | `FieldTaskSpec` | Pydantic model |
| `ResumePair` | `DeploymentPair` | Pydantic model |
| `resumes_{timestamp}.jsonl` | `profiles_{timestamp}.jsonl` | output file |
| `jobs_{timestamp}.jsonl` | `tasks_{timestamp}.jsonl` | output file |
| `pairs_{timestamp}.jsonl` | `pairs_{timestamp}.jsonl` | output file |
| `POST /review-resume` | `POST /review-deployment` | endpoint |
| `GET /health` | `GET /health` | endpoint |
| `GET /analysis/failure-rates` | `GET /analysis/failure-rates` | endpoint |
| skills | capabilities | concept |
| skills overlap | capability overlap | failure metric |
| seniority level | autonomy level | ordinal scale |
| experience years | field seasons | scalar quantity |
| `is_niche_role` | `is_niche_task` | boolean flag |
| hallucinated skills | overclaimed capabilities | failure metric |
| career-changer template | cross-domain transfer template | prompt template |
| GPA | `benchmark_gpa` | bounded float 0.0–4.0 |

All other names, thresholds, file formats, and section structures are unchanged from the original.

***

## 🔁 System Architecture Overview

Your pipeline should follow this high-level flow:

**0. Grounding**: Build the vocabulary snapshot once (see next section) → commit it → all later stages read only the snapshot

**1. Generation**: Generate field task specs (with niche task detection) → Generate capability profiles with controlled fit levels per task → Create task-profile pairs with metadata

**2. Validation**: Schema validation (Pydantic models) → Error extraction and categorization → Save valid/invalid records separately

**3. Analysis**: Calculate failure metrics (Jaccard, season gaps, etc.) → Optional LLM-as-Judge for subtle quality issues → Generate correlation matrices and heatmaps

**4. Correction (Optional)**: Feed validation errors back to LLM → Re-validate corrected outputs → Track correction success rates

**5. API Exposure**: POST /review-deployment (analyze profile against task) → GET /health (health check) → GET /analysis/failure-rates (aggregate statistics)

***

## 🌱 The Vocabulary Snapshot

Generated data is only as good as the vocabulary it draws from. If you invent the capability names yourself, your normalization logic will be solving a mess you created — which proves nothing. Instead, seed the vocabulary from a real, public, downloadable index of agricultural ML datasets, then let the LLM generate freely from that seed.

### Scope: vocabulary only

**The snapshot seeds vocabulary. Nothing else.** Specifically:

* ✅ Use it to draw crop names, field operations, and capability names
* ✅ Use it to derive the `is_niche_task` flag from measured crop rarity
* ✅ Use it to seed realistic task-instruction phrasing
* ❌ **Do not** use dataset episode counts, robot types, or any other metadata as a ground-truth oracle for hallucination detection

That last exclusion is deliberate. Hallucination detection in this project is **rule-based**, exactly as in the original — pattern matching over proficiency distributions, absolutist phrasing, and timeline arithmetic. Using real dataset metadata as an oracle would replace the technique you are supposed to be building.

### Sources

Two public HuggingFace collections. Both were fetched and verified on 2026-08-08; re-fetch before relying on the figures below.

**1. `Project-AgML/*` — 274 agricultural datasets.** Fetch the index:

```
https://huggingface.co/api/datasets?author=Project-AgML&limit=1000
```

Dataset names follow a `{crop}_{operation}_{qualifier}` convention, yielding three vocabularies:

* **Crops** (~95 distinct): apple, tomato, grape, rice, soybean, wheat, strawberry, sugarbeet, cotton, maize… down to agarwood, betel, taro, jujube, custard apple, moringa, centella asiatica
* **Operations**: detection, segmentation, classification, counting, maturity, ripeness, grading, quality, disease, weed, pruning, damage
* **Qualifiers**: spain, usa, brazil, germany, greece, minnesota, californiaday, californianight, australia, denmark, bangladesh, vietnam, iraq, uganda, drone, uav, thermal, synthetic

**2. Agricultural LeRobot manipulation datasets.** Fetch `meta/info.json` and `meta/tasks.parquet` from each:

| Dataset ID | `robot_type` | Episodes | Frames | Tasks |
| --- | --- | --- | --- | --- |
| `Faless/harvest_apples_with_agilex_piper_sim_ee_paraphrases20` | `piper_full` | 600 | 1,020,000 | 20 |
| `tomato-store/bi_arm_harvest` | `bi_so_follower` | 100 | 115,069 | 1 |
| `HojinJung/IsaacSim_apple_harvesting_20260707` | `rb10_1300e` | 127 | 49,933 | 1 |

Note that in LeRobot v3.0 the task string is the **index** of `tasks.parquet`, not a column — read with `pd.read_parquet(path).reset_index()`. Verified task strings include:

```
'Pick each red apple and place it into the green basket.'
'Harvest the red apples one by one and deposit them in the green basket.'
'Sequentially pick the red apples and put each one into the green basket.'
'harvest tomatoes.'
'harvest the target apple'
```

The first dataset carries 20 paraphrases of a single instruction. Use them as a reference for what *natural* task language sounds like — it is the baseline your awkward-language detector measures marketing copy against.

### Build script and caching

Write `scripts/build_vocabulary.py` that fetches both sources and writes `data/vocabulary_snapshot.json`. **Commit that snapshot.** The pipeline must read only the committed file, never the network. This keeps runs reproducible, keeps your success metrics from shifting between runs, and lets the whole pipeline run offline. Refreshing the snapshot is an explicit, logged action — record it in your iteration log like any other configuration change.

### Deriving `is_niche_task`

Count how many datasets in the AgML index mention each crop. The distribution is a clean long tail — measured on the 2026-08-08 snapshot:

| Crop | Datasets |
| --- | --- |
| weed | 18 |
| grape | 12 |
| banana | 12 |
| rice | 9 |
| tomato, date, soybean, apple | 7 |
| mango, tea, bean | 6 |
| cowpea, pomegranate, maize, guava, strawberry | 5 |
| … | … |
| agarwood, jujube, taro, moringa, custard apple, sapota, cocoa | 1 |

Set the threshold at **≤ 2 datasets → niche**. On this snapshot that classifies roughly 60 of ~95 crops as niche while the standard crops account for the large majority of datasets — a realistic head-and-tail split. Record the threshold in your iteration log and tune it if your niche/standard task counts come out too lopsided to compare.

***

## 📊 Success Metrics

Your system will be evaluated on these quantitative benchmarks:

### 1. **Data Generation Quality**

* Generate **50+ field task specs** across diverse crops and production systems
* Generate **5-10 capability profiles per task** with controlled fit levels:
* Excellent fit (80%+ capability overlap)
* Good fit (60-80%)
* Partial fit (40-60%)
* Poor fit (20-40%)
* Complete mismatch (\<20%)

### 2. **Schema Validation Performance**

* **Target: >90% validation success rate** for generated data
* Detailed error categorization for failures:
* Missing required fields
* Type mismatches
* Format violations (email, dates, phone)
* Logical inconsistencies (end\_date before start\_date)

### 3. **Failure Detection Accuracy**

Your labeling system must calculate these metrics for every task-profile pair:

| ​ | Metric                  | Calculation Method                                                              | Threshold       |
| - | ----------------------- | ------------------------------------------------------------------------------- | --------------- |
| ​ | **Capability Overlap**  | Jaccard similarity:                                                             | A ∩ B           |
| ​ | **Experience Mismatch** | Field-season gap or \<50% of required                                           | Binary flag     |
| ​ | **Autonomy Mismatch**   | Level difference (Teleop=0, Assisted=1, Supervised=2, Conditional=3, Full=4)    | >1 level = flag |
| ​ | **Missing Core Capabilities** | Absence of top-3 required capabilities                                    | Binary flag     |
| ​ | **Overclaimed Capabilities** | Unrealistic claims (20+ "expert" capabilities, etc.)                       | Binary flag     |
| ​ | **Awkward Language**    | Excessive agtech buzzwords, AI patterns                                         | Binary flag     |

### 4. **Correction Loop Effectiveness**

* **Target: >50% correction success rate** for invalid records
* Maximum 3 retry attempts per record
* Track: attempts per success, failure reasons

### 5. **API Performance**

* Response time: **\<2 seconds** (without LLM judge)
* Response time: **\<10 seconds** (with LLM judge enabled)
* All endpoints return valid JSON with proper error handling

***

## 🛠 Technical Requirements

### Required Technology Stack

* **Python 3.10+** - Core language
* **Pydantic** - Schema validation with detailed error reporting
* **Instructor** - Structured LLM outputs
* **LLM Provider** - Groq, OpenAI, or OpenRouter for generation
* **Pandas** - Data manipulation and analysis
* **Matplotlib/Seaborn** - Visualization generation
* **FastAPI** - REST API framework

### Optional Enhancements

* **Braintrust** - Evaluation tracking and logging
* **Logfire** - Observability and tracing
* **Pre-commit hooks** - Code quality (Black, Ruff, MyPy)

The vocabulary snapshot is built with `requests` (or `huggingface_hub`) plus `pandas`, which you already have. This is a one-time build step, not a runtime dependency of the pipeline.

***

## Data Schema Requirements

### RobotCapabilityProfile Schema Must Include:

* **Vendor Info**: platform\_name, vendor\_contact\_email, support\_line, vendor\_hq\_location (+ optional repo\_url, model\_card\_url)
* **Provenance**: base\_policy\_architecture, originating\_lab, base\_policy\_release\_date (+ optional benchmark\_gpa, pretraining\_corpora)
* **Deployments**: site\_operator, deployment\_role, dates, tasks\_performed, field\_metrics
* **Capabilities**: name, proficiency\_level (Beginner/Intermediate/Advanced/Expert), optional seasons
* **Metadata**: trace\_id, generated\_at, prompt\_template, fit\_level, writing\_style

### FieldTaskSpec Schema Must Include:

* **Operation**: name, production\_system, farm\_size\_ha, field\_location
* **Requirements**: required\_capabilities\[], preferred\_capabilities\[], minimum\_provenance, required\_field\_seasons, required\_autonomy\_level
* **Metadata**: trace\_id, generated\_at, is\_niche\_task (boolean flag)

### Validation Rules:

* Vendor contact email must be valid format
* Support line must be ≥10 characters
* Dates must be ISO format
* `benchmark_gpa` must be 0.0-4.0
* Field seasons must be 0-30
* end\_date must be after start\_date (if present)

**On `benchmark_gpa`**: this is a composite grade across an evaluation suite, defined as the weighted mean of per-task-family letter grades (A=4.0 … F=0.0). The 0.0–4.0 scale is retained deliberately so the bounded-float validation rule is identical to the original. It is a project-defined construct, not a published metric.

***

## 🧭 Project-Defined Scales

Two ordinal scales in this project are **defined by this project and by nothing else**. They are not drawn from, aligned with, or intended to represent any published standard, regulation, or classification scheme. Do not cite an external authority for them, and do not let generated text imply one.

### Autonomy Level

| Level | Name | Meaning |
| --- | --- | --- |
| 0 | Teleoperated | Human drives every motion |
| 1 | Assisted | Human in the loop per action; robot assists |
| 2 | Supervised | Robot executes the task; human supervises continuously and intervenes often |
| 3 | Conditional | Robot runs the block unattended; human on call for exceptions |
| 4 | Full | Robot runs the season-long operation unattended |

Mismatch flag if `|profile_level - task_level| > 1`.

### Proficiency Level

Beginner / Intermediate / Advanced / Expert — carried over unchanged from the original. Applies to capabilities.

***

## 🧪 Key Implementation Challenges

### Challenge 1: Multi-Template Generation

Don't generate monotonous data. Implement **5+ prompt templates** with distinct characteristics:

* **Vendor formal** — RFP-response register, corporate and hedged
* **Pilot-deck casual** — startup field-notes tone, first person, informal
* **Engineering datasheet** — technical and detail-heavy; sensors, actuators, latency, payload
* **Benchmark-driven** — achievement-focused; throughput, damage rate, uptime, pick success
* **Cross-domain transfer** — a warehouse or indoor-manipulation platform pivoting into agriculture, arguing transferable capabilities

**Why it matters**: Real vendor dossiers have diverse registers. Your failure detection must work across all of them. The cross-domain transfer template is the hardest case by design — it argues for fit that the capability sets do not directly support.

### Challenge 2: Controlled Fit Level Generation

Generating a "poor fit" dossier is harder than it sounds. You must:

* Intentionally create capability gaps
* Misalign autonomy levels
* Introduce subtle mismatches (not obvious failures)

Domain-specific subtlety worth using: a platform with excellent capability overlap but for the wrong **production system** — a field-grown row-crop platform bid against a polytunnel task. Or right crop, right operation, wrong season window.

**Why it matters**: Your labeling system needs challenging test cases to prove it works.

### Challenge 3: Capability Normalization

`strawberry_detection_2022`, `strawberry_detection_2023`, and `Strawberry Detection` should all match. Implement normalization:

* Lowercase conversion
* Version and year removal
* Suffix and prefix stripping (geography, modality, dataset acronyms)

The AgML index supplies real naming noise — every example below is an actual dataset name from the snapshot:

```
strawberry_detection_2022  /  strawberry_detection_2023        ← year suffixes
riseholme_strawberry_classification_2021                        ← institution prefix + year
betel_leaf_disease_classification  /  ..._2                     ← numeric disambiguators
MH_Weed16_weed_detection                                        ← acronym prefix + version
GHAI_/ ghai_broccoli_detection  vs  GEMINI_cowpea_flower_detection  ← case inconsistency
wGrapeUNIPD-DL_white_grape_bunch_detection                      ← mixed-case hyphenated prefix
weed_crop_detection / crop_weeds_greece / carrot_weeds_germany  ← singular-plural drift
potato_leaf_blight  /  potato_leaf_blight_classification        ← near-duplicates
apple_detection_spain / apple_detection_usa / apple_detection_drone_brazil  ← geography + modality
vegann_mulitcrop_presence_segmentation                          ← typo in the wild ("mulitcrop")
growliflower_caluiflower_segmentation                           ← typo in the wild ("caluiflower")
```

**Why it matters**: Without normalization, Jaccard similarity will be artificially low. The last two lines matter especially — real label sets contain typos, and a normalizer that assumes clean input will silently under-count matches.

### Challenge 4: Overclaim (Hallucination) Detection

Rule-based detection is tricky. Consider these patterns:

* Entry-level platform (\<2 field seasons) claiming "Expert" in 10+ capabilities
* Profile listing 30+ capabilities with most marked "Expert"
* Phrases like "handles all crops", "any field condition", "zero supervision required", "100% pick accuracy"
* Inconsistent timelines (overlapping deployments at two sites in the same season, impossible progressions)

Two rules the agricultural domain donates, both pure arithmetic over fields you already have:

* **Deployment predating provenance** — a deployment window that starts before `base_policy_release_date`. A platform cannot have delivered 2023 field results on a policy released in 2025.
* **Out-of-window harvest** — a harvest deployment claimed outside the crop's plausible season. Requires only a small project-defined crop→season table; keep it in the snapshot file alongside the vocabulary.

**Why it matters**: LLMs hallucinate, and so do vendors. Your system must catch it. Keep this rule-based — do not reach for the dataset metadata as an oracle.

### Challenge 5: Awkward Language Detection

Identify AI-generated or buzzword-heavy text using pattern matching:

* Repeated agtech jargon: "revolutionary", "AI-powered", "farm of the future", "digital transformation", "end-to-end autonomy", "precision at scale", "game-changing", "seamlessly integrates", "unlock the potential of your fields"
* Repetitive patterns: same word 3+ times in close proximity
* Excessive buzzword density: >5 buzzwords in summary/description

Calibrate against the verified LeRobot task strings, which are plain and concrete ("Pick each red apple and place it into the green basket."). That register is your negative control — a description that drifts far from it is the signal.

**Why it matters**: Distinguishes substantive capability claims from low-quality AI-generated marketing copy.

***

## 📦 Deliverables

Your completed system must produce:

### 1. **Generated Data** (JSONL format)

* `profiles_{timestamp}.jsonl` - All generated capability profiles
* `tasks_{timestamp}.jsonl` - All generated field task specs
* `pairs_{timestamp}.jsonl` - Task-profile pairs with metadata

### 2. **Validation Results** (JSON/CSV format)

* `validated_data_{timestamp}.json` - Successfully validated records
* `invalid_{timestamp}.jsonl` - Failed records with error details
* `schema_failure_modes_{timestamp}.json` - Error analysis

### 3. **Failure Analysis** (JSONL format)

* `failure_labels_{timestamp}.jsonl` - All calculated metrics per pair
* Statistics: overall failure rates, correlations, distributions

### 4. **Visualizations** (PNG format)

Generate at least these heatmaps/charts:

* **Failure mode correlation matrix** - Which failures co-occur?
* **Failure rates by fit level** - Do "poor fit" profiles fail more?
* **Failure rates by template** - Which registers cause issues?
* **Niche vs standard tasks** - Do niche crops have different patterns?
* **Schema validation heatmap** - Which fields fail most often?

### 5. **REST API**

Functional FastAPI service with:

* `POST /review-deployment` - Real-time profile analysis
* `GET /health` - Health check
* `GET /analysis/failure-rates` - Aggregate statistics
* Automatic OpenAPI documentation at `/docs`

### 6. **Pipeline Summary** (JSON format)

* `pipeline_summary_{timestamp}.json` - Complete run statistics:
* Total records generated
* Validation success rate
* Failure mode distribution
* Correction success rate (if enabled)
* Processing time per stage
* Vocabulary snapshot version/date used

***

## 🎨 Visualization Requirements

All visualizations must use **Matplotlib**, **Seaborn**, or **Plotly**. Save each as a PNG file in a `visualizations/` directory.

### Required Charts

* **Failure Mode Correlation Matrix** (heatmap): Which failure modes co-occur across task-profile pairs?
* **Failure Rates by Fit Level** (grouped bar chart): Do "poor fit" profiles fail more than "excellent fit" ones?
* **Failure Rates by Template** (grouped bar chart): Which registers (vendor formal, pilot casual, datasheet, benchmark, cross-domain) cause the most issues?
* **Niche vs Standard Tasks** (side-by-side bar chart): Do niche crops have different failure patterns?
* **Schema Validation Heatmap** (heatmap): Which fields fail validation most often, by error category?
* **Overclaiming by Autonomy Level** (stacked bar chart): Do low-autonomy platforms overclaim more than high-autonomy ones?

### Quality Standards

* All charts must have descriptive titles, axis labels, and legends
* Use appropriate color schemes (diverging for correlations, sequential for rates)
* Include grid lines for readability
* Add annotations for key thresholds and targets

***

## 🔄 Iteration Logs

Every configuration or threshold change must be documented. Use this format:

```
## Iteration Log Entry

| Field | Value |
| --- | --- |
| Date | YYYY-MM-DD |
| Component | (e.g., Generator, Validator, Labeler, Correction Loop, API, Vocabulary) |
| Change | What was modified |
| Reason | Why the change was made |
| Before Metric | Value before the change |
| After Metric | Value after the change |
| Delta | Improvement or regression |
| Keep/Revert | Decision and rationale |
```

Example iteration entries:

| Date       | Component       | Change                                                     | Before               | After                | Delta | Decision |
| ---------- | --------------- | ---------------------------------------------------------- | -------------------- | -------------------- | ----- | -------- |
| 2026-01-15 | Generator       | Added explicit ISO date format instruction to prompt       | Validation: 82%      | Validation: 91%      | +9%   | Keep     |
| 2026-01-16 | Labeler         | Added year-suffix stripping to capability normalization    | Jaccard avg: 0.28    | Jaccard avg: 0.41    | +0.13 | Keep     |
| 2026-01-17 | Correction Loop | Included Pydantic error messages in correction prompt      | Correction rate: 38% | Correction rate: 62% | +24%  | Keep     |
| 2026-01-18 | Overclaim       | Lowered expert-capability threshold from 15 to 10          | Detection: 45%       | Detection: 72%       | +27%  | Keep     |
| 2026-01-19 | Vocabulary      | Raised niche crop threshold from ≤1 to ≤2 datasets         | Niche tasks: 8%      | Niche tasks: 24%     | +16%  | Keep     |

***

## 🔄 Correction Loop Strategy

When validation fails, your system should:

* **Extract Error Context**: Field path, error type, invalid value, expected format
* **Construct Correction Prompt**: `The following data failed validation with these errors: [error details] Original data: [invalid data] Please generate a corrected version that fixes these issues.`
* **Re-validate**: Parse corrected output and validate again
* **Retry Logic**: Up to 3 attempts, then mark as permanently failed
* **Track Statistics**: Success rate, average attempts, common failure reasons

**Success Criteria**: >50% of invalid records successfully corrected within 3 attempts.

***

## 🧠 LLM-as-Judge (Advanced Feature)

For subtle quality issues that rule-based systems miss, implement an LLM judge that evaluates:

### Evaluation Criteria:

* **Hallucinations**: Unverifiable capability claims, timeline inconsistencies
* **Awkward Language**: Excessive jargon, unnatural phrasing, AI patterns
* **Fit Assessment**: Holistic capability and deployment-history alignment
* **Red Flags**: Gaps between deployments, autonomy regressions, unexplained site churn

### Output Schema:

```
{
  "has_hallucinations": boolean,
  "hallucination_details": "string (explanation)",
  "has_awkward_language": boolean,
  "awkward_language_details": "string (explanation)",
  "overall_quality_score": 0.0-1.0,
  "fit_assessment": "narrative assessment",
  "recommendations": ["actionable suggestions"],
  "red_flags": ["concerns identified"]
}
```

**Trade-off**: LLM judge is slower (\~5-10s per pair) but catches nuanced issues. Make it optional.

***

## 🎯 Evaluation Approach

Follow these steps in order. Record every result in your iteration log.

### Step 1: Validate Data Generation Volume and Diversity

Generate at least 50 field task specs across diverse crops and production systems. For each task, generate 5-10 capability profiles with controlled fit levels. Verify coverage of all 5 fit levels and all 5 prompt templates.

Example output:

| Fit Level        | Count | % of Total | Avg Capability Overlap |
| ---------------- | ----- | ---------- | ---------------------- |
| Excellent (80%+) | 55    | 22%        | 0.87                   |
| Good (60-80%)    | 52    | 21%        | 0.71                   |
| Partial (40-60%) | 50    | 20%        | 0.49                   |
| Poor (20-40%)    | 48    | 19%        | 0.31                   |
| Mismatch (\<20%) | 45    | 18%        | 0.12                   |

**If any fit level has \< 15% of total pairs**, adjust the generation distribution weights. **If total pairs \< 250**, increase the number of tasks or profiles per task.

### Step 2: Check Schema Validation Rate

Run all generated records through Pydantic validation. Target > 90% pass rate. Categorize failures by error type.

Example output:

| Error Category          | Count | % of Failures | Most Common Field            |
| ----------------------- | ----- | ------------- | ---------------------------- |
| Missing required fields | 12    | 40%           | deployments.field\_metrics   |
| Type mismatches         | 8     | 27%           | capabilities.proficiency\_level |
| Format violations       | 6     | 20%           | vendor.vendor\_contact\_email |
| Logical inconsistencies | 4     | 13%           | deployments.end\_date        |

**If validation rate \< 90%**, inspect the top error category and add explicit formatting instructions to the generation prompt for that field. **If a single field accounts for > 50% of failures**, add a Pydantic field validator with a clear error message.

### Step 3: Verify Failure Labeling Accuracy

Manually spot-check 10 task-profile pairs. Verify Jaccard similarity, experience mismatch, autonomy mismatch, missing core capabilities, overclaiming, and awkward language flags are calculated correctly.

Example output:

| Pair ID   | Jaccard | Exp Mismatch | Autonomy Mismatch | Missing Core | Overclaim | Awkward Lang | Manual Agrees? |
| --------- | ------- | ------------ | ----------------- | ------------ | --------- | ------------ | -------------- |
| pair\_001 | 0.33    | Yes          | No                | Yes          | No        | No           | Yes            |
| pair\_002 | 0.85    | No           | No                | No           | No        | No           | Yes            |
| pair\_003 | 0.12    | Yes          | Yes               | Yes          | No        | Yes          | Yes            |

**If manual agreement \< 80%**, review the normalization logic for capability matching. **If overclaim detection misses obvious cases**, add more pattern rules (e.g., entry-level with 10+ expert capabilities).

### Step 4: Test Correction Loop Effectiveness

Run the correction loop on all invalid records. Target > 50% correction success within 3 attempts.

Example output:

| Attempt   | Records In | Corrected | Still Invalid | Success Rate |
| --------- | ---------- | --------- | ------------- | ------------ |
| 1         | 30         | 18        | 12            | 60%          |
| 2         | 12         | 5         | 7             | 42%          |
| 3         | 7          | 2         | 5             | 29%          |
| **Total** | **30**     | **25**    | **5**         | **83%**      |

**If overall correction rate \< 50%**, improve the correction prompt by including the specific Pydantic error messages and the expected format. **If most failures persist across all 3 attempts**, check whether the error type is something the LLM can reasonably fix (e.g., logical date ordering vs. missing domain knowledge).

### Step 5: Validate API Performance

Test each endpoint with representative payloads. Measure response times and verify error handling for edge cases.

Example output:

| Endpoint                    | Payload                     | Response Time | Status | Edge Case Handled? |
| --------------------------- | --------------------------- | ------------- | ------ | ------------------ |
| POST /review-deployment     | Full pair (no judge)        | 1.2s          | 200    | N/A                |
| POST /review-deployment     | Full pair (with judge)      | 7.8s          | 200    | N/A                |
| POST /review-deployment     | Empty capabilities list     | 0.8s          | 200    | Yes                |
| POST /review-deployment     | Missing fields              | 0.3s          | 422    | Yes                |
| GET /health                 | N/A                         | 0.1s          | 200    | N/A                |
| GET /analysis/failure-rates | N/A                         | 0.5s          | 200    | N/A                |

**If response time > 2s without judge**, profile the analysis pipeline and cache repeated computations. **If edge cases return 500 errors**, add input validation middleware with informative error messages.

### Step 6: Self-Evaluation Questions

After completing steps 1-5, answer these questions honestly:

* Can you explain why a "poor fit" profile was labeled as such?
* Do your visualizations reveal non-obvious patterns (e.g., template bias, niche crop challenges)?
* Does the correction loop actually improve data quality, or does it just mask errors?
* Can the API handle edge cases (empty capabilities, missing fields, malformed JSON)?
* Are failure modes distributed as expected across fit levels?
* Does the cross-domain transfer template fail differently from the others, and can you say why?

***

## 💡 First Principles

**Why generate synthetic task-profile pairs instead of using real data?** Real procurement dossiers are commercially sensitive and expensive to collect, and there is no public corpus of them. Synthetic generation lets you produce hundreds of diverse examples at low cost while controlling for specific quality dimensions (fit level, register, capability distribution). The tradeoff is that synthetic data can drift from reality, which is why validation and failure detection are essential — and why the vocabulary is anchored to a real dataset index rather than invented wholesale.

**Why anchor the vocabulary to a real index but still generate synthetically?** Because the two failure modes are different. Inventing the vocabulary yourself makes Challenge 3 circular: you create the naming mess, then solve it, and learn nothing about whether your normalizer survives contact with real label noise. Anchoring the vocabulary while generating the records keeps the exercise synthetic — you retain full control over fit levels and quality dimensions — but forces the normalizer to handle year suffixes, acronym prefixes, and typos that actually exist.

**Why use Pydantic for validation instead of manual checks?** Manual validation is error-prone and hard to maintain. Pydantic enforces structural rules at the schema level, catches type mismatches automatically, and produces detailed error messages that can be fed back into the correction loop. It turns validation from a manual review step into an automated, repeatable process.

**Why measure 6 failure modes separately instead of a single quality score?** A single score hides the structure of the problem. If 80% of your failures come from overclaimed capabilities, that requires a different fix than if failures are spread evenly across all modes. Separate metrics let you target corrections precisely and measure whether each fix actually worked.

**Why include a correction loop?** Generation is imperfect. Rather than discarding every invalid record, feeding validation errors back to the LLM gives it a chance to fix specific issues. This mirrors real-world data pipelines where automated repair is cheaper than regeneration. Tracking correction success rates tells you whether the loop is actually helping or just masking problems.

**Why expose the system as an API?** A pipeline that only runs as a batch script is useful for analysis but not for real-time applications. Wrapping the analysis logic in a REST API makes it usable by other systems (e.g., a procurement tool that screens dossiers on submission). It also forces you to handle edge cases, error responses, and performance constraints that batch processing can ignore.

***

## 💡 Bonus Challenges (Optional)

If you want to go beyond the baseline:

### 1. **Multi-Hop Questions for Evaluation**

Generate test questions that require understanding multiple profile sections:

* "Does this platform's provenance and deployment history align with the task's required autonomy level?"
* "Are the claimed capabilities consistent with the deployment roles and field metrics reported?"

### 2. **Feedback Classification**

Add thumbs up/down feedback mechanism to API responses, log to Braintrust for continuous improvement.

### 3. **Advanced RAG Integration**

Store capability profiles in a vector database, implement semantic search for "find similar platforms".

### 4. **Prompt Template Optimization**

A/B test different prompt templates, measure which produces highest validation rates.

### 5. **Synthetic Data Augmentation**

Generate "corrected" versions of failed profiles, compare failure rates before/after.

### 6. **Vocabulary Sensitivity Analysis**

Re-run the pipeline against a deliberately *unnormalized* vocabulary and measure how much average Jaccard drops. This quantifies exactly what Challenge 3 buys you.

***

## 🚀 Getting Started Hints

### Recommended Development Order:

* **Build the vocabulary snapshot** - Fetch once, commit, never touch the network again
* **Define schemas** - Pydantic models with all validation rules
* **Build generators** - Get LLM generation working with one template first
* **Implement validation** - Ensure you can catch and categorize errors
* **Add failure labeling** - Start with Jaccard similarity, then add other metrics
* **Create visualizations** - Prove your labeling system works
* **Build API** - Expose functionality for real-time use
* **Add correction loop** - Improve data quality iteratively
* **Integrate observability** - Add Braintrust/Logfire if desired

### Common Pitfalls to Avoid:

* **Don't hardcode prompts** - Use templates with variable injection
* **Don't skip normalization** - Capability matching will fail without it
* **Don't fetch the vocabulary at run time** - Snapshot it, or your metrics move under you
* **Don't use dataset metadata as a hallucination oracle** - Overclaim detection is rule-based by design
* **Don't attribute the autonomy scale to a standard** - It is project-defined; say so wherever it appears
* **Don't ignore edge cases** - Handle missing fields, empty lists, null values
* **Don't generate all data at once** - Use batch processing with progress tracking
* **Don't forget trace IDs** - Essential for debugging and linking records

### Storage Strategy:

* Use **JSONL** for generated data (streaming-friendly, line-by-line processing)
* Use **JSON** for summaries, analysis results, and the vocabulary snapshot
* Use **CSV** for tabular exports (easy to load in pandas/Excel)
* Use **PNG** for visualizations (widely compatible)

***

## 📚 Key Concepts to Understand

### Jaccard Similarity

```
Given two sets A and B:
Jaccard(A, B) = |A ∩ B| / |A ∪ B|

Example:
Profile capabilities: {strawberry_detection, ripeness_classification,
                       selective_pick, basket_deposit}
Task requirements:    {strawberry_detection, ripeness_classification,
                       stem_cut_grasp, occluded_reach}

Intersection: {strawberry_detection, ripeness_classification} = 2 items
Union: {strawberry_detection, ripeness_classification, selective_pick,
        basket_deposit, stem_cut_grasp, occluded_reach} = 6 items
Jaccard = 2/6 = 0.33 (poor overlap)
```

### Autonomy Level Mapping

```
Teleoperated:  0
Assisted:      1
Supervised:    2
Conditional:   3
Full:          4

Mismatch if |profile_level - task_level| > 1

Project-defined scale. Not derived from any published standard.
```

### Field Experience Calculation

```
For each deployment:
  if end_date exists:
    duration = end_date - start_date
  else:
    duration = today - start_date  # Ongoing deployment

total_field_experience = sum(all durations)
```

Report in seasons. If a deployment window spans a single growing season, count it
as one season regardless of calendar length — a four-month harvest deployment and
a nine-month protected-cropping deployment are both one season.

### Capability Normalization

```
Raw:        "MH_Weed16_weed_detection"
lowercase:  "mh_weed16_weed_detection"
strip prefix acronym:   "weed16_weed_detection"
strip version digits:   "weed_weed_detection"
collapse repeats:       "weed_detection"

Raw:        "strawberry_detection_2023"
strip year:             "strawberry_detection"

Raw:        "apple_detection_drone_brazil"
strip geography:        "apple_detection_drone"
strip modality:         "apple_detection"
```

***

## ✅ Final Success Criteria

Before submitting, verify that your implementation meets all of the following:

### Data Generation

* [ ] 50+ field task specs generated across diverse crops and production systems
* [ ] 5-10 capability profiles per task with controlled fit levels (250+ total pairs)
* [ ] All 5 fit levels represented (excellent, good, partial, poor, mismatch)
* [ ] All 5 prompt templates used (vendor formal, pilot casual, datasheet, benchmark-driven, cross-domain transfer)
* [ ] Niche task detection flag set correctly
* [ ] All records have trace IDs and timestamps

### Schema Validation

* [ ] Pydantic models defined for RobotCapabilityProfile, FieldTaskSpec, and DeploymentPair
* [ ] Validation rules enforced (email format, date ordering, benchmark\_gpa range, etc.)
* [ ] Validation success rate > 90%
* [ ] Error categorization by type (missing fields, type mismatches, format violations, logical inconsistencies)
* [ ] Valid and invalid records saved separately with proper filenames

### Failure Labeling

* [ ] All 6 failure metrics calculated for every pair (capability overlap, experience mismatch, autonomy mismatch, missing core capabilities, overclaiming, awkward language)
* [ ] Capability normalization implemented (lowercase, version/year removal, prefix and suffix stripping)
* [ ] Jaccard similarity correctly calculated and spot-checked on 10+ pairs
* [ ] Overclaim detection covers entry-level overclaiming, excessive expert ratings, impossible timelines
* [ ] Awkward language detection catches buzzword density and repetitive patterns

### Correction Loop

* [ ] Correction prompt includes specific Pydantic error messages
* [ ] Maximum 3 retry attempts per record
* [ ] Correction success rate > 50%
* [ ] Statistics tracked (attempts per success, failure reasons)

### LLM-as-Judge

* [ ] Evaluates hallucinations, awkward language, fit assessment, red flags
* [ ] Structured output with scores and explanations
* [ ] Optional (can be enabled/disabled per request)

### API

* [ ] POST /review-deployment responds in \< 2s (without judge), \< 10s (with judge)
* [ ] GET /health returns health status
* [ ] GET /analysis/failure-rates returns aggregate statistics
* [ ] All endpoints return valid JSON with proper error handling
* [ ] Edge cases handled (empty capabilities, missing fields, malformed input)

### Visualizations

* [ ] Failure mode correlation matrix (heatmap)
* [ ] Failure rates by fit level (grouped bar chart)
* [ ] Failure rates by template (grouped bar chart)
* [ ] Niche vs standard tasks (side-by-side bar chart)
* [ ] Schema validation heatmap
* [ ] Overclaiming by autonomy level (stacked bar chart)
* [ ] All charts saved as PNG with Matplotlib, Seaborn, or Plotly

### Iteration Logs and Traceability

* [ ] Every threshold or weight change documented with reason, before/after metrics, and delta
* [ ] At least 3 iteration log entries showing experimentation
* [ ] Final configuration decisions traceable to specific iteration log entries

### Testing and Documentation

* [ ] Pipeline runs end-to-end without crashes
* [ ] Output files saved with timestamps (profiles, tasks, pairs, labels, summaries)
* [ ] Pipeline summary JSON includes total records, validation rate, failure distribution, correction rate, timing
* [ ] You can explain why any given profile was labeled with specific failure modes

**Remember**: This isn't about following a step-by-step tutorial. It's about understanding the problem, making architectural decisions, and proving your solution works through rigorous evaluation. Good luck!
