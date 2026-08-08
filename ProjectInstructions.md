# Project Instructions — Synthetic Data Pipeline for Agricultural VLA Robotics

> **For anyone picking this up cold.** This document is self-contained — everything needed to
> build the project is here. **Read §0 before writing any domain prose.** Several
> plausible-sounding claims about agricultural robotics standards are factually wrong and have
> already been caught once; §0 records the corrections and exists to stop them coming back.

---

## 0. Guardrails — read before writing any domain prose

These are not style preferences. Each one is a specific factual error that an adversarial review
caught in an earlier draft, verified against primary sources.

| ❌ Do not | ✅ Instead |
| --- | --- |
| Cite ISO 18497 as defining autonomy *levels* | ISO 18497-1 defines a **non-ordinal vocabulary**. Clause 3 states, of each term: *"Partially automated is not a level or a category of a machine."* ISO 18497-4's Introduction records that SAE J3016's six levels were *"deliberately not chosen as a basis for the ISO 18497 series."* |
| Claim any standard backs the 5-level autonomy ladder | **The ladder in §5 is a project invention.** Label it as such every time it appears. There is no ISO 18497-5, no ASABE autonomy-levels standard, and ISO 17757 is binary. Industry (VDMA/AEF) harmonizes on **Agri-ODD**, a scenario framework — not a ladder. |
| Cite a standard for `field_readiness_grade` | ISO 18497-4 contains zero occurrences of *validation body*, *certif\**, *grade*, or *readiness*. The field exists **solely** to preserve a bounded-float validator. Defend it on that basis. |
| Call failure mode #5 "overclaiming" | "Overclaiming" is not a term of art — zero arXiv full-text hits. The documented phenomenon is an **evaluation-validity gap**: LIBERO scores cluster at 90–95% while real-world performance spans the full range. Describe the gap; do not impute intent. |
| List `RT-2-X` as a usable `base_policy` | Its weights were **never released**. Use it only inside mismatch/hallucination cases — claiming to have fine-tuned an unreleased model is itself a detectable red flag. |
| Write `YOLOv11` as canonical | Ultralytics dropped the "v": **`YOLO11`**. Keep `YOLOv11` in the normalization alias table as the *common error* — that is exactly what makes the exercise non-trivial. |
| Reference `ROS 2 Humble` as current | **`ROS 2 Jazzy`** (LTS to May 2029). Humble hits EOL May 2027. |
| Include aquaculture as a sub-sector | Excluded. It is marine robotics — different locomotion medium, sonar rather than RGB-D, no GNSS underwater. |
| Invent niche tasks | Use only the verified list in §6. Truffle detection (sensing, not robotics) and micro-green thinning (not a real operation) were removed for cause. |

**The honest thesis, which belongs in your First Principles section:** a full-text grep of the
field's current survey returns **zero** occurrences of "agricultur", "farm", "orchard", or "crop".
Every available VLA fact — OpenVLA, π0, Octo, Open X-Embodiment, DROID, LIBERO, CALVIN — comes from
**indoor, tabletop, short-horizon manipulation research**. Agriculture is outdoor, long-horizon,
safety-critical, GNSS-dependent, weather-coupled, and acts on deformable biological targets with
seasonal non-stationarity. That gap is **the justification for this project**, not a weakness to
conceal. Presenting tabletop benchmark numbers as though they characterized field-robot capability
is the single thing that would discredit the whole deliverable to a domain reader.

---

## 🎯 Project Goal

Build a production-grade synthetic data pipeline that generates, validates, and analyzes
**robot-deployment-dossier ↔ field-task-order** pairs using LLMs. Your system acts as an
intelligent deployment advisor that identifies capability mismatches, detects quality issues, and
provides actionable feedback.

**Core Challenge**: Create a system that not only generates realistic data but also understands
what makes a robot deployment proposal "good" or "bad" for a specific agricultural task, then
prove it works through rigorous evaluation.

---

## 🧠 The Problem Context

Growers and farm co-operatives evaluating field automation receive dozens of vendor proposals for
each operation they want to automate. Most are poorly matched: the platform cannot physically
perform the task, the claimed autonomy exceeds what the system has demonstrated, or the reported
capability derives from simulation benchmarks with poor correlation to field performance.

This is a real and documented problem. VLA policies routinely score 90–95% on standard
manipulation benchmarks while spanning the entire spectrum of real-world success. A dossier can be
entirely truthful about its benchmark numbers and still be useless as a predictor of field
behaviour.

Your task: build an AI system that can

* **Generate** realistic dossier–task pairs with varying quality levels
* **Validate** that data follows strict structural rules
* **Analyze** why a dossier fails to match a task (capability gap? autonomy mismatch? unsupported
  claims? physically impossible platform/task combination?)
* **Visualize** patterns in failures across different scenarios
* **Correct** invalid data through iterative LLM feedback
* **Serve** this intelligence via a REST API for real-time analysis

---

## 🔁 System Architecture Overview

**1. Generation**: Generate field task orders (with niche-task detection) → Generate deployment
dossiers with controlled fit levels per task → Create dossier–task pairs with metadata

**2. Validation**: Schema validation (Pydantic models) → Error extraction and categorization →
Save valid/invalid records separately

**3. Analysis**: Calculate failure metrics (Jaccard, season gaps, platform contradictions) →
Optional LLM-as-Judge for subtle quality issues → Generate correlation matrices and heatmaps

**4. Correction (Optional)**: Feed validation errors back to LLM → Re-validate corrected outputs →
Track correction success rates

**5. API Exposure**: `POST /review-deployment` → `GET /health` → `GET /analysis/failure-rates`

---

## 📊 Success Metrics

### 1. Data Generation Quality

* Generate **50+ field task orders** across diverse agricultural sub-sectors
* Generate **5–10 deployment dossiers per task order** with controlled fit levels:
  * Excellent fit (80%+ capability overlap)
  * Good fit (60–80%)
  * Partial fit (40–60%)
  * Poor fit (20–40%)
  * Complete mismatch (<20%)

### 2. Schema Validation Performance

* **Target: >90% validation success rate**
* Detailed error categorization: missing required fields · type mismatches · format violations
  (email, dates, phone) · logical inconsistencies (`end_date` before `start_date`)

### 3. Failure Detection Accuracy

Calculate all six for every pair:

| Metric | Calculation Method | Threshold |
| --- | --- | --- |
| **Capability Overlap** | Jaccard similarity: \|A ∩ B\| / \|A ∪ B\| | continuous |
| **Field-Season Mismatch** | Years gap, or <50% of required | Binary flag |
| **Autonomy Mismatch** | Level difference (Teleoperated=0 … Full Field Autonomy=4) | >1 level = flag |
| **Missing Core Capabilities** | Absence of top-3 required capabilities | Binary flag |
| **Unsupported Capability Claims** | Evaluation-validity gap, impossible timelines, platform contradictions | Binary flag |
| **Agtech Buzzword Density** | Excessive marketing jargon, AI patterns | Binary flag |

### 4. Correction Loop Effectiveness

* **Target: >50% correction success rate** · Maximum 3 retry attempts · Track attempts per
  success and failure reasons

### 5. API Performance

* **<2 seconds** without LLM judge · **<10 seconds** with judge · valid JSON with proper error
  handling on all endpoints

---

## 🛠 Technical Requirements

### Required Technology Stack

* **Python 3.10+** · **Pydantic** (schema validation with detailed error reporting) ·
  **Instructor** (structured LLM outputs) · **LLM Provider** (Groq, OpenAI, or OpenRouter) ·
  **Pandas** · **Matplotlib/Seaborn** · **FastAPI**

### Optional Enhancements

* **Braintrust** (evaluation tracking) · **Logfire** (observability) · **Pre-commit hooks**
  (Black, Ruff, MyPy)

### Environment note

A conda environment named **`sdg`** already exists on this machine
(`/home/cofuente/anaconda3/envs/sdg`, Python 3.12.13) carrying pydantic 2.13.4, instructor 1.15.1,
groq, openai, pandas 3.0.2, matplotlib, seaborn, pytest, black and ruff. Only `fastapi` and
`uvicorn` are missing. **Reuse it — do not create a new environment, and never install Python
packages against the host interpreter.**

```bash
conda activate sdg
```

```bash
pip install fastapi uvicorn
```

⚠️ pandas is at **3.0.x**. Write aggregation code against the 3.x API, not 2.x idioms.

---

## Data Schema Requirements

### `RobotDeploymentDossier` must include

* **Operator Contact**: contact_name, email, phone, site_location (+ optional fleet_portal_url,
  telemetry_endpoint)
* **Policy Provenance**: base_policy, training_provider, policy_release_date (+ optional
  field_readiness_grade, training_corpora[])
* **Deployment History**: site_operator, deployment_role, start_date, end_date, tasks_performed[],
  outcomes[]
* **Capabilities**: name, proficiency_level (Demonstrated/Benchmarked/Field-Validated/
  Production-Certified), optional validated_hours
* **Platform**: platform_locomotion, supported_interaction_modes[], validated_environments[]
* **Metadata**: trace_id, generated_at, prompt_template, fit_level, writing_style

### `FieldTaskOrder` must include

* **Operation**: grower_name, sub_sector, operation_size, site_location
* **Facets**: operating_environment, canopy_geometry, locomotion, interaction_mode, odd_volatility
* **Requirements**: required_capabilities[], preferred_capabilities[], min_policy_provenance,
  required_field_seasons, required_autonomy_level
* **Metadata**: trace_id, generated_at, is_niche_task (boolean flag)

### Validation Rules

* Email must be valid format
* Phone must be ≥10 characters
* Dates must be ISO format
* **`field_readiness_grade` must be 0.0–4.0**
* **`field_seasons` must be 0–30**
* `end_date` must be after `start_date` (if present)

> **On `field_readiness_grade`:** this is a project-defined field on a 0.0–4.0 scale. It exists to
> exercise bounded-float validation. **No standard defines it**, and no citation should be attached
> to it. See §0.

### Valid `base_policy` values

`OpenVLA-7B` · `OpenVLA-OFT` · `π0` · `Octo-Base` · `Octo-Small`

Octo is a 2024-era baseline — realistic as a *legacy* choice, not a current frontier one.
`RT-2-X` is real but its weights were never released; use it only to seed hallucination cases.

**Do not extend this list without verifying against a primary source.** In particular, do not add
`π0.5`: its weight-release status could not be established, and a `base_policy` value is a claim
that a practitioner could obtain and fine-tune the model. Every name above was confirmed to have
released weights.

---

## 5. The Autonomy Ladder

```
0  Teleoperated            — continuous human control
1  Operator-Assisted       — human in the loop, machine assists
2  Supervised Autonomy     — machine acts, human monitors and intervenes
3  Conditional Autonomy    — unsupervised within a defined operating zone
4  Full Field Autonomy     — unsupervised across zones and conditions
```

**Mismatch if `|dossier_level − task_level| > 1`.**

> ⚠️ **This ladder is a project-defined synthetic taxonomy with no standards provenance.** State
> this wherever the ladder appears. ISO 18497-1 provides a non-ordinal vocabulary and explicitly
> disclaims levels; ISO 18497-4 explicitly declined to adopt SAE J3016's ladder; the industry
> harmonizes on Agri-ODD, a scenario framework rather than a scale. A five-level ordinal scale is
> required here to exercise ordinal-distance logic, so we define one and label it honestly rather
> than borrowing authority no standard grants.

**Field seasons** are the experience analogue: sum of deployment durations, with an open-ended
deployment measured to today. Range 0–30.

---

## 6. Taxonomy

### Sub-sectors (the diversity axis — 9 values, ≥6 task orders each = 54)

`row_crops` · `orchard` · `vineyard_viticulture` · `greenhouse_cea` · `livestock_dairy` ·
`nursery_horticulture` · `silviculture` · `post_harvest_packhouse` · `specialty_high_value`

*Aquaculture is deliberately excluded — see §0.*

### Faceted axes (orthogonal; these drive generation and enable contradiction detection)

```
operating_environment : open_field | protected_cropping | indoor_facility | livestock_housing
canopy_geometry       : planar_trellis | volumetric_canopy | ground_plane | none
locomotion            : wheeled_ugv | tracked_ugv | uav | gantry | fixed_arm | rail
interaction_mode      : sense_only | non_contact_actuation | contact_rigid | contact_compliant
odd_volatility        : low | medium | high
```

Stratify generation across facet combinations so coverage is structural, not merely nominal.

### Niche tasks (`is_niche_task = true`) — verified list only

| Task | Status |
| --- | --- |
| Hop-trellis string-tying | Published; 97% success, 11.2 s/cycle |
| Saffron stigma harvesting | Commercial (Dyno Robotics × Blue Red Gold) |
| Date-palm drone pollination | Published (*Scientific Reports*) |
| Vanilla hand-pollination | Prototype-stage |
| Automated hive management | Commercial (Beewise BeeHome) |
| Robotic blossom / green-fruit thinning | Published, tree fruit |

Standard (non-niche) tasks: row-crop weeding, broadacre spraying, orchard mowing, greenhouse tomato
harvest, robotic milking, packhouse grading.

---

## 🧪 Key Implementation Challenges

### Challenge 1: Multi-Template Generation

Implement **5 prompt templates** with distinct characteristics:

| Template | Character |
| --- | --- |
| `vendor_datasheet` | Formal, regulatory-submission register; spec-table dense |
| `agtech_pitch` | Casual startup tone; benefit-led, light on numbers |
| `model_card` | Technical; architecture, action space, eval protocol, ablations |
| `field_trial_metrics` | Achievement-focused; ha/hr, success rate, intervention rate |
| `cross_embodiment` | A tabletop/warehouse-trained policy pitched for field work, emphasizing transferable skills |

**Why it matters**: real proposals have diverse registers. Your failure detection must work across
all of them. `cross_embodiment` is the most valuable of the five — cross-embodiment transfer is a
live research concern, and these dossiers should be genuinely *hard* to judge, not obviously bad.

### Challenge 2: Controlled Fit Level Generation

Generating a "poor fit" dossier is harder than it sounds:

* Intentionally create capability gaps
* Misalign autonomy levels
* Introduce subtle mismatches (not obvious failures)

### Challenge 3: Capability Normalization

`RTK-GPS`, `RTK GNSS`, and `rtk positioning` should all match. Implement:

* Lowercase conversion
* Version-number removal — `ROS 2 Jazzy` → `ros`, `YOLO11` → `yolo`, `SAM 2` → `sam`,
  `OpenVLA-7B` → `openvla`
* Suffix stripping — `_control`, `-perception`, ` module`, ` stack`, ` pipeline`, ` node`
* **An alias table for canonical ↔ wrong-variant pairs** — `yolov11 → yolo11`,
  `rtk-gps → rtk gnss`, `ros2 → ros 2`, `segment-anything → sam`

**Why it matters**: without normalization, Jaccard similarity is artificially low. This domain is
version-dense *and* carries genuinely wrong variants in circulation, which is what makes the alias
table earn its place.

### Challenge 4: Unsupported-Claim Detection

Rule-based detection. Cover these patterns:

* Dossier with <2 field seasons claiming `Production-Certified` on 10+ capabilities
* Dossier listing 30+ capabilities with most marked `Production-Certified`
* Phrases: `fully autonomous in all crops`, `zero human intervention`, `100% detection accuracy`,
  `works in any weather`, `no calibration required`
* **Timeline impossibility**: overlapping deployments at geographically incompatible sites;
  `validated_hours` exceeding the wall-clock span of the deployment window
* **Platform/task contradiction** — computable from the schema alone, no external data:
  * `uav` claiming `contact_compliant` manipulation
  * `gantry` claiming `open_field` operation at scale
  * `indoor_facility` deployment claiming `high` ODD volatility
  * `fixed_arm` claiming multi-hectare coverage
  * `sense_only` platform claiming harvesting or weeding outcomes
* **Evaluation-validity gap**: `training_corpora` is simulation-only while capabilities claim
  `Field-Validated`; or `base_policy` names a model with no released weights

Frame this in prose as an **evaluation-validity gap**, not vendor dishonesty. See §0.

### Challenge 5: Agtech Buzzword Detection

* Repeated marketing jargon: `revolutionize`, `game-changing`, `AI-powered`, `end-to-end`,
  `seamlessly`, `next-generation`, `holistic`, `synergy`, `digital transformation`,
  `unlock value`, `future-proof`
* Repetitive patterns: same token 3+ times in close proximity
* Excessive density: >5 buzzwords in a summary/description

---

## 📦 Deliverables

### 1. Generated Data (JSONL)

`dossiers_{timestamp}.jsonl` · `task_orders_{timestamp}.jsonl` · `pairs_{timestamp}.jsonl`

### 2. Validation Results (JSON/CSV)

`validated_data_{timestamp}.json` · `invalid_{timestamp}.jsonl` ·
`schema_failure_modes_{timestamp}.json`

### 3. Failure Analysis (JSONL)

`failure_labels_{timestamp}.jsonl` — all calculated metrics per pair, plus overall failure rates,
correlations, distributions

### 4. Visualizations (PNG, in `visualizations/`)

### 5. REST API

`POST /review-deployment` · `GET /health` · `GET /analysis/failure-rates` · OpenAPI docs at `/docs`

### 6. Pipeline Summary (JSON)

`pipeline_summary_{timestamp}.json` — total records, validation success rate, failure-mode
distribution, correction success rate, processing time per stage

---

## 🎨 Visualization Requirements

Matplotlib, Seaborn, or Plotly. Save each as PNG in `visualizations/`.

* **Failure Mode Correlation Matrix** (heatmap) — which failure modes co-occur?
* **Failure Rates by Fit Level** (grouped bar) — do "poor fit" dossiers fail more?
* **Failure Rates by Template** (grouped bar) — which registers cause the most issues?
* **Niche vs Standard Tasks** (side-by-side bar) — do niche tasks have different patterns?
* **Schema Validation Heatmap** — which fields fail most often, by error category?
* **Unsupported Claims by Autonomy Level** (stacked bar) — do lower-autonomy dossiers overclaim more?

**Quality standards**: descriptive titles, axis labels, legends · diverging colormaps for
correlations, sequential for rates · grid lines · annotations for key thresholds and targets.

---

## 🔄 Iteration Logs

Every configuration or threshold change must be documented:

| Field | Value |
| --- | --- |
| Date | YYYY-MM-DD |
| Component | Generator / Validator / Labeler / Correction Loop / API |
| Change | What was modified |
| Reason | Why |
| Before Metric | Value before |
| After Metric | Value after |
| Delta | Improvement or regression |
| Keep/Revert | Decision and rationale |

Example entries:

| Date | Component | Change | Before | After | Delta | Decision |
| --- | --- | --- | --- | --- | --- | --- |
| 2026-08-10 | Generator | Added explicit ISO-date instruction to prompt | Validation: 82% | Validation: 91% | +9% | Keep |
| 2026-08-11 | Labeler | Added alias table to capability normalization | Jaccard avg: 0.28 | Jaccard avg: 0.44 | +0.16 | Keep |
| 2026-08-12 | Correction Loop | Included Pydantic error messages in correction prompt | Correction: 38% | Correction: 62% | +24% | Keep |
| 2026-08-13 | Unsupported Claims | Lowered Production-Certified threshold from 15 to 10 | Detection: 45% | Detection: 72% | +27% | Keep |

**At least 3 entries required**, and final configuration decisions must be traceable to specific
entries.

---

## 🔄 Correction Loop Strategy

* **Extract Error Context**: field path, error type, invalid value, expected format
* **Construct Correction Prompt**: `The following data failed validation with these errors:
  [error details] Original data: [invalid data] Please generate a corrected version that fixes
  these issues.`
* **Re-validate**: parse corrected output and validate again
* **Retry Logic**: up to 3 attempts, then mark permanently failed
* **Track Statistics**: success rate, average attempts, common failure reasons

**Success Criteria**: >50% of invalid records corrected within 3 attempts.

---

## 🧠 LLM-as-Judge (Advanced Feature)

Evaluates: **unsupported claims** (unverifiable assertions, timeline inconsistencies) ·
**awkward language** (excessive jargon, AI patterns) · **fit assessment** (holistic capability and
platform alignment) · **red flags** (deployment gaps, inconsistent capability progression,
sim-only evidence presented as field evidence).

```json
{
  "has_unsupported_claims": true,
  "unsupported_claim_details": "string (explanation)",
  "has_awkward_language": false,
  "awkward_language_details": "string (explanation)",
  "overall_quality_score": 0.0,
  "fit_assessment": "narrative assessment",
  "recommendations": ["actionable suggestions"],
  "red_flags": ["concerns identified"]
}
```

**Trade-off**: ~5–10 s per pair. Make it optional (enable/disable per request).

---

## 🎯 Evaluation Approach

Record every result in your iteration log.

### Step 1: Validate Generation Volume and Diversity

50+ task orders across 9 sub-sectors; 5–10 dossiers each; verify all 5 fit levels and all 5
templates.

| Fit Level | Count | % of Total | Avg Capability Overlap |
| --- | --- | --- | --- |
| Excellent (80%+) | 55 | 22% | 0.87 |
| Good (60–80%) | 52 | 21% | 0.71 |
| Partial (40–60%) | 50 | 20% | 0.49 |
| Poor (20–40%) | 48 | 19% | 0.31 |
| Mismatch (<20%) | 45 | 18% | 0.12 |

**If any fit level has <15% of total pairs**, adjust generation distribution weights.
**If total pairs <250**, increase task orders or dossiers per order.

### Step 2: Check Schema Validation Rate

Target >90%. Categorize failures.

| Error Category | Count | % of Failures | Most Common Field |
| --- | --- | --- | --- |
| Missing required fields | 12 | 40% | deployment_history.outcomes |
| Type mismatches | 8 | 27% | capabilities.proficiency_level |
| Format violations | 6 | 20% | operator_contact.email |
| Logical inconsistencies | 4 | 13% | deployment_history.end_date |

**If <90%**, add explicit formatting instructions for the top error category. **If a single field
is >50% of failures**, add a Pydantic field validator with a clear message.

### Step 3: Verify Failure Labeling Accuracy

Manually spot-check 10+ pairs across all six metrics.

| Pair ID | Jaccard | Season Mismatch | Autonomy Mismatch | Missing Core | Unsupported | Buzzwords | Manual Agrees? |
| --- | --- | --- | --- | --- | --- | --- | --- |
| pair_001 | 0.33 | Yes | No | Yes | No | No | Yes |
| pair_002 | 0.85 | No | No | No | No | No | Yes |
| pair_003 | 0.12 | Yes | Yes | Yes | Yes | Yes | Yes |

**If manual agreement <80%**, review normalization logic. **If unsupported-claim detection misses
obvious cases**, add pattern rules — start with the platform contradictions, which are the most
reliable.

### Step 4: Test Correction Loop

Target >50% within 3 attempts.

| Attempt | Records In | Corrected | Still Invalid | Success Rate |
| --- | --- | --- | --- | --- |
| 1 | 30 | 18 | 12 | 60% |
| 2 | 12 | 5 | 7 | 42% |
| 3 | 7 | 2 | 5 | 29% |
| **Total** | **30** | **25** | **5** | **83%** |

**If <50%**, include specific Pydantic error messages and expected format in the prompt. **If
failures persist across all 3**, check whether the error is something an LLM can reasonably fix.

### Step 5: Validate API Performance

| Endpoint | Payload | Response Time | Status | Edge Case Handled? |
| --- | --- | --- | --- | --- |
| POST /review-deployment | Full pair (no judge) | 1.2s | 200 | N/A |
| POST /review-deployment | Full pair (with judge) | 7.8s | 200 | N/A |
| POST /review-deployment | Empty capabilities list | 0.8s | 200 | Yes |
| POST /review-deployment | Missing fields | 0.3s | 422 | Yes |
| GET /health | N/A | 0.1s | 200 | N/A |
| GET /analysis/failure-rates | N/A | 0.5s | 200 | N/A |

**If >2s without judge**, profile and cache repeated computations. **If edge cases 500**, add
input validation with informative errors.

### Step 6: Self-Evaluation Questions

* Can you explain why a "poor fit" dossier was labeled as such?
* Do your visualizations reveal non-obvious patterns (template bias, niche-task difficulty)?
* Does the correction loop improve data quality, or just mask errors?
* Can the API handle edge cases (empty capabilities, missing fields, malformed JSON)?
* Are failure modes distributed as expected across fit levels?

---

## 💡 First Principles

**Why synthesize this data at all?** Because it does not exist. Every VLA fact available — OpenVLA,
π0, Octo, Open X-Embodiment, DROID, LIBERO, CALVIN — comes from indoor tabletop manipulation
research. There is essentially no VLA corpus for agriculture, and real field-deployment records are
commercially confidential. Synthetic generation produces hundreds of diverse examples at low cost
while controlling for fit level, register, and capability distribution. The tradeoff is drift from
reality, which is why validation and failure detection are essential.

**Why Pydantic rather than manual checks?** Manual validation is error-prone and hard to maintain.
Pydantic enforces rules at the schema level, catches type mismatches automatically, and produces
detailed error messages that feed directly into the correction loop.

**Why six failure modes rather than one quality score?** A single score hides the structure of the
problem. If 80% of failures are platform contradictions, that needs a different fix than if they
are spread evenly. Separate metrics let you target corrections and measure whether each fix worked.

**Why a correction loop?** Generation is imperfect. Feeding validation errors back gives the LLM a
chance to fix specific issues, mirroring real pipelines where automated repair is cheaper than
regeneration. Tracking success rates tells you whether it helps or merely masks.

**Why expose an API?** A batch script is useful for analysis but not for real-time applications.
Wrapping the logic in REST makes it usable by other systems and forces you to handle edge cases,
error responses, and performance constraints that batch processing can ignore.

**Why do the platform contradiction rules matter more than they look?** They are the only failure
signal computable entirely from the record itself, with no external database. A UAV cannot perform
compliant contact manipulation; a gantry cannot cover open field at scale. These are physical
facts, not statistical tendencies — which makes them the most reliable ground truth in the system.

---

## 💡 Bonus Challenges (Optional)

1. **Multi-hop evaluation questions** — "Does this platform's locomotion mode support the
   interaction mode the task requires?" · "Are the claimed capabilities consistent with the
   deployment roles and outcomes?"
2. **Feedback classification** — thumbs up/down on API responses, logged to Braintrust
3. **Advanced RAG** — vector-store the dossiers, implement "find similar platforms"
4. **Prompt template optimization** — A/B test templates, measure validation rates
5. **Synthetic data augmentation** — generate corrected versions of failed dossiers, compare
   failure rates before/after

---

## 🚀 Getting Started Hints

### Recommended development order

1. **Schemas** — Pydantic models with all validation rules
2. **Generators** — one template first
3. **Validation** — catch and categorize errors
4. **Failure labeling** — Jaccard first, then platform contradictions (cheapest and most
   reliable), then the rest
5. **Visualizations** — prove the labeling works
6. **API**
7. **Correction loop**
8. **Observability** — Braintrust/Logfire if desired

### Common pitfalls

* **Don't hardcode prompts** — templates with variable injection
* **Don't skip normalization** — capability matching fails without it
* **Don't ignore edge cases** — missing fields, empty lists, nulls
* **Don't generate all data at once** — batch with progress tracking
* **Don't forget trace IDs** — essential for debugging and linking records
* **Don't cite standards you haven't read** — see §0. This project has already been burned once.

### Storage strategy

**JSONL** for generated data · **JSON** for summaries · **CSV** for tabular exports · **PNG** for
visualizations

---

## 📚 Key Concepts

### Jaccard Similarity

```
Jaccard(A, B) = |A ∩ B| / |A ∪ B|

Dossier capabilities: {row following, rtk gnss, ripeness classification, compliant grasping}
Task requirements:    {row following, rtk gnss, weed discrimination, spray actuation}

Intersection: {row following, rtk gnss} = 2
Union: {row following, rtk gnss, ripeness classification, compliant grasping,
        weed discrimination, spray actuation} = 6
Jaccard = 2/6 = 0.33 (poor overlap)
```

### Autonomy Level Mapping

```
Teleoperated:         0
Operator-Assisted:    1
Supervised Autonomy:  2
Conditional Autonomy: 3
Full Field Autonomy:  4

Mismatch if |dossier_level - task_level| > 1
```

*(Project-defined scale — see §5 and §0.)*

### Field-Season Calculation

```
For each deployment:
  if end_date exists:
    duration = end_date - start_date
  else:
    duration = today - start_date   # ongoing deployment

total_field_seasons = sum(all durations)
```

---

## ✅ Final Success Criteria

### Data Generation
* [ ] 50+ field task orders generated across diverse agricultural sub-sectors
* [ ] 5–10 dossiers per task order with controlled fit levels (250+ total pairs)
* [ ] All 5 fit levels represented (excellent, good, partial, poor, mismatch)
* [ ] All 5 prompt templates used (vendor datasheet, agtech pitch, model card, field-trial metrics, cross-embodiment)
* [ ] Niche-task detection flag set correctly
* [ ] All records have trace IDs and timestamps

### Schema Validation
* [ ] Pydantic models defined for RobotDeploymentDossier, FieldTaskOrder, and DeploymentPair
* [ ] Validation rules enforced (email format, date ordering, field_readiness_grade range, etc.)
* [ ] Validation success rate > 90%
* [ ] Error categorization by type (missing fields, type mismatches, format violations, logical inconsistencies)
* [ ] Valid and invalid records saved separately with proper filenames

### Failure Labeling
* [ ] All 6 failure metrics calculated for every pair (capability overlap, field-season mismatch, autonomy mismatch, missing core capabilities, unsupported claims, buzzword density)
* [ ] Capability normalization implemented (lowercase, version removal, suffix stripping, alias table)
* [ ] Jaccard similarity correctly calculated and spot-checked on 10+ pairs
* [ ] Unsupported-claim detection covers low-season overclaiming, excessive Production-Certified ratings, impossible timelines, and platform contradictions
* [ ] Buzzword detection catches density and repetitive patterns

### Correction Loop
* [ ] Correction prompt includes specific Pydantic error messages
* [ ] Maximum 3 retry attempts per record
* [ ] Correction success rate > 50%
* [ ] Statistics tracked (attempts per success, failure reasons)

### LLM-as-Judge
* [ ] Evaluates unsupported claims, awkward language, fit assessment, red flags
* [ ] Structured output with scores and explanations
* [ ] Optional (can be enabled/disabled per request)

### API
* [ ] POST /review-deployment responds in < 2s (without judge), < 10s (with judge)
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
* [ ] Unsupported claims by autonomy level (stacked bar chart)
* [ ] All charts saved as PNG with Matplotlib, Seaborn, or Plotly

### Iteration Logs and Traceability
* [ ] Every threshold or weight change documented with reason, before/after metrics, and delta
* [ ] At least 3 iteration log entries showing experimentation
* [ ] Final configuration decisions traceable to specific iteration log entries

### Testing and Documentation
* [ ] Pipeline runs end-to-end without crashes
* [ ] Output files saved with timestamps (dossiers, task orders, pairs, labels, summaries)
* [ ] Pipeline summary JSON includes total records, validation rate, failure distribution, correction rate, timing
* [ ] You can explain why any given dossier was labeled with specific failure modes

### Domain Integrity
* [ ] No standard is cited as defining autonomy levels
* [ ] The autonomy ladder carries its project-defined disclaimer wherever it appears
* [ ] `field_readiness_grade` carries no standards citation
* [ ] Failure mode #5 is framed as an evaluation-validity gap, not as dishonesty
* [ ] Only verified niche tasks appear in generated data

**Remember**: This isn't about following a step-by-step tutorial. It's about understanding the
problem, making architectural decisions, and proving your solution works through rigorous
evaluation.
