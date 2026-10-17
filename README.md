# llm-observability-platform

[![ci](https://github.com/hammasbuilds/llm-observability-platform/actions/workflows/ci.yml/badge.svg)](https://github.com/hammasbuilds/llm-observability-platform/actions/workflows/ci.yml)

We built llm-observability-platform as monitoring for LLM applications: every call is recorded with
its cost, the model that actually served it, whether a fallback was used, its grounding score and the
feature and tenant it came from. It reports cost per feature and per successful answer, latency
percentiles, prompt drift without storing prompts, and alerts that do not fire on noise. It is for
teams running LLM features in production who need to know which feature is expensive, slow or broken.
The core has no dependencies.

A full web UI for this lives in [LLM Control Room](https://github.com/hammasbuilds/llm-control-room)
(Observability page).

## What it does

- **Cost attribution:** spend per feature, tenant and model, plus cost per *successful* outcome, so a
  feature that is both expensive and failing stands out (`top_spend("feature")`).
- **Percentiles, not means:** p50/p95/p99 latency; cache hits are excluded so a 2 ms hit cannot hide
  the tail a user on a miss sees.
- **Prompt drift without prompts:** Population Stability Index (PSI) over prompt shape (question type,
  length bucket, a hash of the structural skeleton with content words removed), model mix, latency and
  cost. The prompt text is discarded at record time.
- **Alerts:** every rule has a sample floor, a cooldown and, where one exists, a baseline; rules run per
  feature as well as globally, so one broken feature is visible inside a healthy aggregate.
- **Time-based windows:** drift compares adjacent time windows, not a window against all history.
- **Fits the gateway:** `LLMCall` maps onto what [llm-gateway](https://github.com/hammasbuilds/llm-gateway)
  records and what [rag-forge](https://github.com/hammasbuilds/rag-forge) produces (grounding, refusal).

## Sample input / output

`python demo.py` measures how often PSI raises a false "moderate shift" alarm on two windows drawn from
the same distribution (no drift applied), at different window sizes:

```
INPUT
   two windows per row, both from the same generator
   cache hit rate     30%  ->  1.0-4.0 ms
   cache miss         70%  ->  180.0-900.0 ms
   no shift is applied to either window at any sample size

OUTPUT
   300 independent trials per row. A 'false alarm' is PSI >= 0.10
   -- the conventional 'moderate shift' threshold -- on two windows that
   are drawn from the same distribution.

        n   median psi   worst psi   false alarms
   --------------------------------------------------
      150       0.1168      0.3887     192 / 300  (64.0%)
      300       0.0553      0.1545      24 / 300  ( 8.0%)
      500       0.0327      0.0914       0 / 300  ( 0.0%)
     2000       0.0085      0.0257       0 / 300  ( 0.0%)
```

Recording a call and reading it back:

```python
>>> obs.record(LLMCall(feature="search", tenant="acme", model="m", prompt_tokens=820,
...     completion_tokens=140, usd=0.0046, latency_ms=980, grounding=0.82, fallback_used=True),
...     prompt="What is the capital of Pakistan?")
>>> obs.top_spend("feature")
[{'feature': 'search', 'total_usd': 0.0046, 'calls': 1, 'usd_per_success': 0.0046, 'error_rate': 0.0}]
>>> obs.drift()
{'status': 'insufficient data', 'reference': 0, 'current': 1}
```

## Results

- With 150 calls per window, PSI crosses the 0.10 alarm threshold in 64% of trials where nothing
  changed, and its worst draw (0.3887) passes the 0.25 "significant" line. At 500 calls per window it
  never fired in 300 trials. This is why every alert rule here carries a sample floor.
- 49 tests cover percentiles, aggregation (cache exclusion, cost per success, missing vs zero
  grounding), PSI edge cases, prompt signatures, drift and alerts, including a feature failing 100% of
  the time inside 2,000 healthy calls. No numpy, network or model.

PSI reading used throughout: below 0.10 stable, 0.10 to 0.25 moderate shift, above 0.25 significant.

## Tech stack

- Python 3.12, standard library only (PSI is implemented in plain Python)
- JSONL sink for recorded calls
- Dev: pytest, ruff, uv

## Run it

```bash
git clone https://github.com/hammasbuilds/llm-observability-platform
cd llm-observability-platform
uv sync --all-groups        # or: pip install -e . pytest ruff
uv run python demo.py
uv run pytest -q
```

```python
from pathlib import Path
from llmobs import Observatory, LLMCall

obs = Observatory(sink=Path("runs/calls.jsonl"))
obs.record(LLMCall(feature="search", model="claude-sonnet-5", usd=0.0046, latency_ms=980),
           prompt=user_question)   # signature derived, text discarded
obs.summary(3600)          # percentiles, rates, cost per success
obs.by("feature")          # the same, broken out
obs.top_spend("feature")   # what to switch off
obs.drift()                # PSI across prompt shape, model mix, latency, cost
obs.check_alerts()         # global and per feature
```

Storage is in-process with a JSONL sink. Grounding scores are supplied by the caller; this platform
aggregates them, it does not compute them.

## Licence

MIT, see [LICENSE](LICENSE).
