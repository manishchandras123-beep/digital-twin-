# Patient Digital Twin (Simulated) — Internship Project

⚠️ Educational proof-of-concept only. All patient data is synthetic. The risk
score is a simplified teaching version of NEWS2, not a real clinical tool.

## What this is

A digital twin architecture (Source → Processing → Storage → Visualization)
applied to a simulated patient's vital signs, modeling a sepsis-like
deterioration pattern and scoring risk using a simplified NEWS2 clinical score.

## Files

- `patient_simulator.py` — generates synthetic vitals (heart rate, temp, resp
  rate, blood pressure, SpO2), either as a historical batch or a live stream.
- `risk_scoring.py` — simplified NEWS2 scoring logic, turns raw vitals into a
  risk score (Normal / Low / Medium / High).
- `database.py` — SQLite storage layer for the time-series readings.
- `dashboard.py` — Streamlit dashboard: the live, human-facing view of the twin.

## How to run

```bash
pip install -r requirements.txt
streamlit run dashboard.py
```

Then open the local URL Streamlit prints (usually http://localhost:8501).
Click "Generate next reading" repeatedly (or check "Auto-generate 10 readings")
to watch the simulated patient deteriorate and the risk score climb.

## Suggested next steps (see project roadmap discussion)

- Replace the simplified NEWS2 rules with an ML model (e.g. Isolation Forest,
  or a regression model predicting time-to-high-risk) trained on
  `generate_historical_batch()` data.
- Add alerting (e.g. highlight/flash when risk crosses "High").
- Add a "what-if" slider to manually adjust one vital and see the resulting score.
- Swap SQLite for a time-series DB (InfluxDB/TimescaleDB) if you want to show
  you understand production-grade IoT storage.
