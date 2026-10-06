# Digital Twin Generator Agent — Framework

## 1. The core idea

Every digital twin we could build — patient monitoring, industrial pumps,
smart buildings, vehicles — follows the **same four-stage pipeline**:

```
Source (simulate/ingest) → Process (score/predict) → Store → Visualize
```

What changes between domains is just the *content* inside each stage: which
variables exist, what "normal" looks like, how they degrade, and what
scoring/thresholds apply. That means the pipeline itself can be
**templated**, and a generator agent's real job is just: take a
specification of the domain, and fill in the template.

This is exactly what I did manually for your patient twin — the generator
agent automates that manual process.

---

## 2. The specification schema (the "contract")

This is what a user (or another agent) provides as input. Everything the
generator needs to produce a working twin comes from this document.

```yaml
domain:
  name: patient_health          # or industrial_pump, smart_building, etc.
  entity_label: "Patient"       # what a single instance is called
  description: "Vital sign monitoring for early deterioration detection"

inputs:                          # the sensors / variables being tracked
  - name: heart_rate_bpm
    baseline: 75.0
    noise_std: 2.0
    unit: bpm
  - name: temperature_C
    baseline: 37.0
    noise_std: 0.1
    unit: "°C"
  # ... one entry per sensor/variable

process:
  degradation_curve: exponential   # exponential | linear | logistic | custom
  scenarios:                       # different "modes" of deterioration/failure
    - name: sepsis
      label: "Sepsis / infection progression"
      deltas:                      # shift per input at severity = 1.0
        heart_rate_bpm: 55
        temperature_C: 3.2
    - name: cardiac_stress
      label: "Cardiac stress"
      deltas: { heart_rate_bpm: 65, temperature_C: 0.3 }

output:
  rule_based_scoring:
    enabled: true
    framework: NEWS2               # or a custom threshold table
  ml_prediction:
    enabled: true
    model_type: random_forest      # or logistic_regression, isolation_forest
  risk_labels: [Normal, Low Risk, Medium Risk, High Risk]

behavior:
  entity_identifier: patient_name  # what distinguishes one instance from another
  storage: sqlite                  # or influxdb, postgres
  visualization: streamlit_dashboard
  update_mode: manual_step          # or live_interval

metadata:
  disclaimers:
    - "Educational proof-of-concept, not a diagnostic tool"
```

Notice: this is literally the design decisions we already made for the
patient twin, written down as data instead of code.

---

## 3. Generator agent architecture

```
┌───────────────┐   ┌───────────────┐   ┌────────────────┐   ┌───────────────┐   ┌────────────────┐
│ Spec parser & │──▶│ Domain filler │──▶│ Template engine │──▶│ Code generator │──▶│ Validator +    │
│ validator     │   │ (agent-assisted)│  │ (Jinja2)        │   │ (writes files)│   │ report writer  │
└───────────────┘   └───────────────┘   └────────────────┘   └───────────────┘   └────────────────┘
```

**Stage 1 — Spec parser & validator**
Reads the YAML/JSON spec, checks required fields are present, fills sane
defaults for anything optional (e.g. default noise levels if omitted).

**Stage 2 — Domain filler (this is where an LLM agent adds real value)**
If the user only says *"industrial centrifugal pump"* without specifying
baseline values, an LLM agent can research/recall typical operating
parameters (normal RPM, vibration ranges, etc.) and propose them — flagged
for human review rather than auto-trusted, since domain accuracy matters.

**Stage 3 — Template engine**
Five parameterized templates, one per file we already built manually:
- `simulator.py.jinja` → becomes `patient_simulator.py` / `pump_simulator.py` / etc.
- `scenario_engine.py.jinja` → becomes `disease_engine.py` / `failure_mode_engine.py`
- `scoring.py.jinja` → becomes `risk_scoring.py`, generated from the threshold
  table in the spec (NEWS2-style, or a custom one)
- `ml_predictor.py.jinja` → becomes `ml_predictor.py`, model type swappable
- `dashboard.py.jinja` → becomes `dashboard.py`

**Stage 4 — Code generator**
Fills each template with the spec's values and writes out a complete,
runnable project folder — exactly like the 6 files you now have, just
produced automatically instead of by hand.

**Stage 5 — Validator + report writer**
Runs the generated code through basic sanity checks (does the scoring
respond correctly to a "severe" preset? does the ML model train without
error?) — the same kind of tests I ran manually with `bash_tool` throughout
this conversation — then auto-generates the architecture report and
diagram, just like `PROJECT_REPORT.md`.

---

## 4. How your patient twin maps onto this framework

| Framework concept | What it became in your project |
|---|---|
| `inputs` | heart rate, temperature, resp rate, BP, SpO2 |
| `process.scenarios` | sepsis, cardiac stress, respiratory distress, fever |
| `process.degradation_curve` | exponential severity curve |
| `output.rule_based_scoring` | NEWS2-based `risk_scoring.py` |
| `output.ml_prediction` | Random Forest in `ml_predictor.py` |
| `behavior.entity_identifier` | patient name |
| `behavior.storage` | SQLite |
| `behavior.visualization` | the Streamlit dashboard |

Your entire project is a **single instantiation** of this generic
framework, with `domain.name = patient_health`. That's a genuinely strong
thing to say in a presentation — you didn't just build one twin, you built
proof that a domain-general pattern works.

---

## 5. Example: the same framework, different domain

To prove generality, here's the pump project sketched in the same spec
shape (no code, just to show the schema transfers):

```yaml
domain:
  name: industrial_pump
  entity_label: "Pump"

inputs:
  - name: temperature_C
    baseline: 60.0
  - name: vibration_mm_s
    baseline: 2.0
  - name: rpm
    baseline: 1500.0

process:
  scenarios:
    - name: bearing_wear
      deltas: { temperature_C: 35, vibration_mm_s: 6, rpm: -250 }

output:
  rule_based_scoring:
    framework: custom_thresholds
  ml_prediction:
    model_type: isolation_forest
```

Same shape, same generator, completely different physical system.

---

## 6. Two ways to actually build the generator

**Option A — Deterministic templates (Jinja2 + Python)**
Fully predictable output, fast, no LLM needed at generation time. Best for
well-understood domains where the pipeline shape truly doesn't change.

**Option B — Agent-driven generation (LLM writes the code directly)**
Give an LLM the spec and the *pattern* (not rigid templates), let it write
domain-appropriate code directly, then run the validator stage to catch
mistakes. More flexible for domains the template author didn't anticipate,
at the cost of needing a validation/test loop (which you should have
anyway).

**Recommended: hybrid.** Use templates for the parts that never change
(the four-stage pipeline skeleton, the database layer, the dashboard
scaffold), and let an agent fill in domain-specific content (baseline
values, degradation deltas, scoring thresholds) when the user hasn't fully
specified them.

---

## 7. Suggested build roadmap

1. Write the spec schema formally (a JSON Schema or Pydantic model) so it
   can be validated automatically.
2. Convert your existing 6 files into Jinja2 templates by replacing
   hardcoded values (BASELINE, DISEASE_PROFILES, NEWS2 thresholds) with
   template variables.
3. Build a CLI: `generate_twin.py --spec my_domain.yaml --out ./output/`
4. Test it by regenerating your existing patient twin from a spec file —
   if the output matches what you have now, the generator works.
5. Test it again on a second domain (e.g. the pump) to prove it
   generalizes, not just replays one case.
6. Add the validator + auto-report stage last, once generation itself is solid.
