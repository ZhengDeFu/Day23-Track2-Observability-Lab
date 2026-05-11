# Day 23 Lab Reflection

**Student:** _fill your name_
**Submission date:** 2026-05-11
**Lab repo URL:** _fill public GitHub URL_

---

## 1. Hardware + setup output

I ran `python3 00-setup/verify-docker.py` with elevated Docker access enabled by Codex. It wrote `00-setup/setup-report.json` with this result while the observability stack was already running:

```json
{
  "docker": {
    "ok": true,
    "version": "29.4.2"
  },
  "compose_v2": {
    "ok": true,
    "version": "5.1.3"
  },
  "ram_gb_available": 14.96,
  "ram_ok": true,
  "required_ports": [
    8000,
    9090,
    9093,
    3000,
    3100,
    16686,
    4317,
    4318,
    8888
  ],
  "bound_ports": [
    8000,
    9090,
    9093,
    3000,
    3100,
    16686,
    4317,
    4318,
    8888
  ],
  "all_ports_free": false
}
```

The ports are reported as bound because `docker compose up -d` had already started the lab stack. Before a clean pre-flight run, stop the stack with `make down`; during the actual lab run, these ports being bound is expected.

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels

Pending screenshot: `submission/screenshots/dashboard-overview.png`

The dashboard JSON files were statically validated with:

```text
make lint-dashboards
```

Result:

```text
[OK] ai-service-overview.json
[OK] cost-and-tokens.json
[OK] slo-burn-rate.json
```

### Burn-rate panel

Pending screenshot: `submission/screenshots/slo-burn-rate.png`

Run `make up`, wait for Grafana provisioning, then run `make load`. If the burn-rate window is empty, keep the app under load long enough for Prometheus recording rules to populate the 5m/30m windows.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| T0 | Kill `day23-app` with `make alert` | pending screenshot `alertmanager-firing.png` |
| T0+90s | `ServiceDown` fires | pending screenshot `slack-firing.png` |
| T1 | App restored by script | — |
| T1+60s | Alert resolves | pending screenshot `slack-resolved.png` |

Slack webhook evidence is pending because no real `SLACK_WEBHOOK_URL` was provided. Alertmanager was started with a placeholder webhook URL so the core stack and `make verify` can pass; replace it with the real Slack webhook before collecting the fire/resolve screenshots.

### One thing surprised me about Prometheus / Grafana

The useful part of the dashboards is not the number of panels, but whether the labels and recording rules make the panels cheap to query and easy to compare. For this lab, the split between RED metrics, SLO burn rate, and cost/token metrics makes the dashboard operational instead of just decorative.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Pending screenshot: `submission/screenshots/jaeger-trace.png`

After Docker access is fixed, run:

```text
make up
make trace
```

Then open Jaeger at `http://localhost:16686`, search service `inference-api`, and capture a `POST /predict` trace showing `embed-text`, `vector-search`, and `generate-tokens`.

### Log line correlated to trace

Structured JSON log line from `docker compose logs app`:

```text
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 41, "quality": 0.884, "duration_seconds": 0.1793, "trace_id": "f0404eac4a0095cc7c477d20f6e2064f", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T04:41:27.915038Z"}
```

### Tail-sampling math

The configured policy keeps 100% of traces with `ERROR` status, 100% of traces slower than 2 seconds, and 1% of healthy traces. If the service produces `N` healthy traces/sec and no slow/error traces, the collector keeps about `0.01 * N` traces/sec. If there are `E` error traces/sec and `S` slow traces/sec, the retained rate is approximately `E + S + 0.01 * healthy`, minus overlap between categories.

---

## 4. Track 04 — Drift Detection

### PSI scores

`make drift` completed and generated both:

- `04-drift-detection/reports/drift-summary.json`
- `04-drift-detection/reports/drift-report.html`

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Which test fits which feature?

For `prompt_length`, I would use PSI for monitoring because it is an interpretable binned distribution shift metric and prompt length is easy to bucket for dashboarding. KS is also useful as a statistical backstop because prompt length is continuous.

For `embedding_norm`, I would use KS for the scalar norm and MMD for the full embedding vector in production. A single norm can miss directional embedding drift, while MMD can compare high-dimensional embedding distributions without reducing them to one scalar.

For `response_length`, I would use PSI for operational monitoring and KS for investigation. Length is continuous but easy to bucket, and a PSI threshold is easier to explain to an on-call engineer than a p-value alone.

For `response_quality`, I would use KS when quality is a continuous score and PSI when the score is bucketed into ranges such as poor/acceptable/good. KL is useful when comparing full probability distributions, but it is more sensitive to zero or tiny bins and needs smoothing.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The hardest metric to expose would likely be the vector store or model serving metric from Day 19/20 because it depends on another local service being reachable from inside the Docker network, usually through `host.docker.internal`. The Prometheus scrape target is simple once networking is correct, but the operational failure mode is subtle: the dashboard can render while the target is silently down or showing "No Data".

Pending screenshot: `submission/screenshots/cross-day-dashboard.png`

---

## 6. The single change that mattered most

The single change that matters most in this stack is treating AI-specific signals as first-class metrics instead of only collecting generic HTTP latency and error rate. Adding token counters and a quality gauge changes the dashboard from "the API is up" to "the AI service is useful, affordable, and behaving within expectation." That is the difference between basic service monitoring and observability for an AI system.

The deck's RED/USE model is still necessary: request rate, errors, duration, active requests, and GPU utilization tell me whether the service is healthy. But for an inference service, tokens and quality are the signals that connect infrastructure behavior to product behavior. A service can have low latency and still be failing users if quality drops or token cost spikes. Putting those metrics next to burn-rate panels makes the stack actionable for both SRE and platform decisions.
