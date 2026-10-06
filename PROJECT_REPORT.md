# Patient Digital Twin — Simulated Sepsis Progression Monitor
### Internship Project Report — Architecture & Design Rationale

---

## 1. Overview

This project implements a **digital twin architecture** applied to patient
vital-sign monitoring. It simulates a patient's vitals over time, models a
sepsis-like physiological deterioration pattern, scores clinical risk using
a simplified version of the real **NEWS2** (National Early Warning Score 2)
system, and visualizes everything through a live, interactive dashboard.

**Important scope note:** this is an educational proof-of-concept. All
patient data is synthetic (no real patients, sensors, or clinical data are
involved), and the risk scoring is a simplified teaching version of NEWS2 —
not a validated diagnostic or clinical decision-support tool. The value of
the project is in demonstrating the *engineering pattern* behind real-world
digital twins, which generalizes to any domain (industrial equipment, smart
buildings, vehicles, etc.), not in the specific medical claims.

---

## 2. What Is a Digital Twin?

A digital twin is a **live, continuously data-connected virtual replica**
of a physical entity. It differs from a plain simulation in one key way:
a simulation runs a model once and stops; a digital twin stays synchronized
with new incoming data from a real (or in this case, simulated) instance
over time, so its state always reflects the current state of the thing it
represents.

Every digital twin, regardless of domain, has the same three components:

1. **Physical entity** — the real-world object or system (here: a patient)
2. **Virtual model** — the software representation and logic (here: the
   vitals data model + NEWS2 scoring)
3. **Data connection** — the mechanism feeding data from the physical
   entity into the virtual model (here: the simulated sensor stream)

---

## 3. System Architecture

The project follows the standard digital twin / IoT pipeline pattern:

```
┌─────────────────┐     ┌─────────────┐     ┌──────────────┐     ┌───────────────┐
│   SOURCE         │ --> │  PROCESSING │ --> │   STORAGE    │ --> │ VISUALIZATION │
│ patient_simulator│     │risk_scoring │     │  database.py │     │  dashboard.py │
│ (synthetic vitals)│    │  (NEWS2)    │     │  (SQLite)    │     │  (Streamlit)  │
└─────────────────┘     └─────────────┘     └──────────────┘     └───────────────┘
```

Each stage is implemented as a **separate, independent module** with a
narrow interface to the next stage. This is a deliberate design choice, not
an accident — see Section 5 (Design Decisions) for why it matters.

### 3.1 Source — `patient_simulator.py`

Generates synthetic vital signs: heart rate, temperature, respiratory rate,
systolic blood pressure, and SpO2 (oxygen saturation). A single hidden
"severity" value (0 = healthy, 1 = severe) drives all five vitals together,
using an **exponential progression curve** — deterioration is slow at first
and accelerates over time, which mirrors how real physiological decline is
qualitatively described in clinical literature, rather than a straight-line
decline. Gaussian noise is layered on top of every reading to simulate
realistic sensor/measurement variability.

Two modes are provided:
- `generate_historical_batch()` — produces a full historical dataset at once
  (used for testing and would be used for training an ML model)
- `PatientVitalsStream` — a generator that produces one new reading at a
  time, simulating a live monitor feed

### 3.2 Processing — `risk_scoring.py`

Implements a simplified version of **NEWS2**, a real early-warning scoring
system used in hospitals. Each vital is scored 0–3 based on deviation from
its normal clinical range; the five scores sum into a total (0–15), which
maps to a risk label: Normal, Low, Medium, or High Risk.

This stage is what turns raw numbers into actionable meaning — the same
role "processing" plays in any digital twin (e.g. converting raw vibration
readings into a "bearing wear" estimate in an industrial twin).

### 3.3 Storage — `database.py`

A SQLite database stores every reading as a timestamped row. The module
exposes exactly two operations to the rest of the system: `insert_reading()`
and `get_all_readings()`. This narrow interface means the storage backend
could be swapped for a production time-series database (e.g. InfluxDB,
TimescaleDB) without changing any other file in the project.

### 3.4 Visualization — `dashboard.py`

A Streamlit web application with two views:

- **Live Monitor** — runs the automated pipeline end-to-end: generates a
  reading, scores it, stores it, and displays live-updating vitals trends,
  current risk status, and a NEWS2 trend chart.
- **What-If Simulator** — a manual sandbox where any vital can be set via
  slider and the resulting NEWS2 score is computed instantly, independent
  of the live patient or database. This demonstrates a core digital-twin
  capability: testing hypothetical scenarios against the virtual model
  without needing (or risking) the real physical system.

---

## 4. Data Flow Example

1. `PatientVitalsStream.next_reading()` produces a reading, e.g.
   `{heart_rate_bpm: 128, temperature_C: 39.5, ...}`
2. `calculate_news2()` scores it: HR contributes 2 points, temp 2, resp rate
   3, BP 3, SpO2 3 → total score 13 → labeled "High Risk"
3. The scored reading is written to the SQLite `vitals` table
4. The dashboard queries the last N readings and renders them as trend
   charts and a color-coded status indicator (🟢/🟡/🟠/🔴)

---

## 5. Design Decisions & Rationale

| Decision | Rationale |
|---|---|
| Modular pipeline (4 separate files) | Mirrors real IoT/digital-twin architecture; each stage can be independently tested, replaced, or scaled |
| SQLite over a full server DB | Zero setup, runs anywhere; interface is written so a production time-series DB is a drop-in replacement later |
| NEWS2-based scoring over custom rules | Grounds the "processing" logic in a real, validated clinical framework rather than arbitrary thresholds, improving credibility and giving a concrete benchmark to compare against |
| Exponential (not linear) degradation curve | Matches how physiological/mechanical deterioration is generally described — subtle early signal, rapid late decline — making the simulated data more realistic for testing detection logic |
| Streamlit for the dashboard | Fast to build in pure Python, sufficient for an interactive demo without requiring separate frontend work |
| Explicit synthetic-data disclaimers throughout UI and code | Ensures the project is clearly framed as an educational proof-of-concept, not a diagnostic tool |

---

## 6. Limitations

- Vitals are generated from a single hidden severity parameter, so
  correlations between vitals are simpler than in real physiology (real
  patients don't deteriorate on one unified axis).
- The NEWS2 implementation is simplified: it omits consciousness level and
  supplemental oxygen use, both part of the official clinical scale.
- No real sensor integration — this is a "digital twin" whose physical
  entity is entirely synthetic, so it demonstrates the architecture pattern
  rather than a deployable clinical system.
- No trained ML model yet — risk scoring is currently rule-based (NEWS2),
  not learned from data.

---

## 7. Future Work

- **Predictive modeling**: train an ML model (e.g. Isolation Forest for
  anomaly detection, or a regression model for time-to-high-risk
  estimation) on the historical batch data, and compare its predictions
  against the rule-based NEWS2 score.
- **Alerting**: push notifications or visual/audio alerts when risk crosses
  into the High Risk band.
- **Multi-patient support**: extend the twin to track several simulated
  patients simultaneously, closer to a real ward-monitoring dashboard.
- **Real sensor integration**: replace `PatientVitalsStream` with real
  wearable/IoT sensor input (e.g. a pulse oximeter or smartwatch API),
  demonstrating that the rest of the pipeline requires no changes — the
  practical payoff of the modular design in Section 5.

---

## 8. Conclusion

This project demonstrates the complete digital twin pattern — a live
data source, a processing/analytics layer, persistent storage, and an
interactive visualization — applied to a healthcare monitoring scenario.
While built on synthetic data, the architecture is directly representative
of how production digital twins are structured across domains (industrial
equipment, smart buildings, healthcare), making the underlying engineering
skills transferable well beyond this specific use case.
