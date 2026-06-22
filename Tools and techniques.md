# 🔬 Tools & Techniques — SentinelAI

> A complete technical reference for every tool, library, algorithm, and concept used in SentinelAI.

---

## 🐍 Core Language

### Python 3.10+
- **Why:** Rich data-science ecosystem (scikit-learn, pandas), readable for security tooling, cross-platform
- **Key features used:** dataclasses, regex (`re`), type hints, argparse, pathlib

---

## 📚 Libraries & Frameworks

### 1. scikit-learn (v1.5.2)
| Property | Detail |
|---|---|
| **Purpose** | Isolation Forest implementation + StandardScaler |
| **Install** | `pip install scikit-learn` |
| **Docs** | https://scikit-learn.org |
| **Key classes used** | `IsolationForest`, `StandardScaler` |
| **Why this library?** | Industry-standard, well-tested, fast (Cython-backed) implementation |

### 2. pandas (v2.2.3)
| Property | Detail |
|---|---|
| **Purpose** | Tabular data manipulation — grouping events into behavioral windows |
| **Install** | `pip install pandas` |
| **Key methods used** | `DataFrame`, `groupby`-style aggregation via dict grouping, `.sort_values()`, `.to_json()` |

### 3. numpy (v2.1.2)
| Property | Detail |
|---|---|
| **Purpose** | Numerical operations on feature matrices |
| **Install** | `pip install numpy` |
| **Used for** | Feature matrix construction, score normalization |

### 4. anthropic (v0.39.0)
| Property | Detail |
|---|---|
| **Purpose** | Official Claude API client |
| **Install** | `pip install anthropic` |
| **Used for** | LLM narration of ML findings |
| **Model used** | `claude-sonnet-4-5` |
| **Fallback** | Rule-based template engine if no API key present |

### 5. python-dotenv (v1.0.1)
| Property | Detail |
|---|---|
| **Purpose** | Load `ANTHROPIC_API_KEY` from `.env` file |
| **Install** | `pip install python-dotenv` |

### 6. pytest (v8.3.3)
| Property | Detail |
|---|---|
| **Purpose** | Testing framework |
| **Install** | `pip install pytest` |
| **Used for** | 31 unit + integration tests |

---

## 🤖 Machine Learning Technique: Isolation Forest

```
Algorithm class:    Unsupervised anomaly detection
Paper:              Liu, Ting, Zhou (2008) "Isolation Forest"
Core idea:          Anomalies are "few and different" — they get
                    isolated by random partitioning faster than
                    normal points
Complexity:         O(n log n) training, O(log n) scoring per point
Hyperparameters used:
  n_estimators=200    → number of random trees in the ensemble
  contamination=0.1   → expected proportion of anomalies (tunable)
  random_state=42     → reproducibility
```

### Why not other algorithms?

| Algorithm | Why not chosen for this task |
|---|---|
| **K-Means clustering** | Requires choosing K upfront; doesn't naturally model "outlier-ness" |
| **DBSCAN** | Sensitive to density parameter tuning; doesn't scale as well |
| **One-Class SVM** | Slower on larger datasets; more sensitive to feature scaling |
| **Supervised classifiers (Random Forest, XGBoost)** | Require labeled "attack" vs "normal" data — not available for arbitrary servers |
| **LSTM/sequence models** | Overkill for this feature set; better suited for raw sequential log streams (noted as future enhancement) |

**Isolation Forest** was chosen because it's unsupervised (no labels needed), fast, and works directly on the tabular behavioral features engineered from log data.

---

## 🧮 Feature Engineering Technique: Time-Windowed Behavioral Aggregation

```
Raw events (point-in-time)  →  Behavioral windows (aggregated)

Why aggregate?
  A single "Failed password" line tells you almost nothing.
  9 failed logins + 6 invalid users + 7 distinct usernames,
  all within 10 minutes from one IP, tells a clear story.

Technique: Sliding/fixed time-window grouping by entity (IP/user)
  Similar to: network flow analysis, fraud detection feature
              engineering, SIEM correlation rules
```

---

## 🧠 LLM Technique: Constrained Narration (Not Decision-Making)

```
Pattern: "ML detects, LLM explains"

This is a form of:
  - Retrieval-Augmented Generation (RAG)-adjacent design —
    the LLM is grounded in structured, pre-computed facts
    rather than asked to reason from scratch
  - Defense-in-depth for AI reliability — the security-critical
    decision (is this anomalous?) never depends on LLM output;
    only the human-readable explanation does
```

**Prompt engineering techniques used:**
- Role assignment ("You are a SOC analyst assistant")
- Structured output format enforcement (`SUMMARY:`, `LIKELY ATTACK PATTERN:`, `RECOMMENDED ACTION:`)
- Explicit grounding ("Do not invent details not present in the data")
- Low max_tokens (300) to keep responses concise and on-task

---

## 🔍 Log Parsing Technique: Regex-Based Syslog Parsing

```
Format parsed: RFC 3164-style syslog
  "MMM DD HH:MM:SS hostname process[pid]: message"

Sub-patterns extracted via targeted regex:
  - Failed password attempts (SSH)
  - Accepted logins (password/publickey)
  - Invalid user attempts
  - sudo command execution
  - PAM session open/close
  - SSH disconnects
```

This is the same general approach used by production log-parsing pipelines (e.g. Logstash grok patterns, Fluentd parsers) — pattern-matching known formats into structured fields before downstream processing.

---

## 🏗️ Architecture & Design Patterns

### 1. Pipeline Architecture
```
log_parser.py → feature_builder.py → anomaly_detector.py → llm_narrator.py
```
Each stage has a single, well-defined responsibility and clean interfaces (lists of dataclasses → DataFrames → scored DataFrames → text reports).

### 2. Graceful Degradation
- `llm_narrator.py` works identically whether or not `ANTHROPIC_API_KEY` is set
- Errors in LLM calls fall back to template narration rather than crashing

### 3. Dataclasses for Structured Events
- `AuthEvent` dataclass provides type safety and clear field documentation versus raw dicts

### 4. CLI Design (argparse)
- Sensible defaults for every flag — tool is usable with just `--log`
- `--no-llm` and `--no-banner` flags support scripting/automation use cases

---

## 🧪 Testing Approach

### Test Categories (31 total)
```
TestAuthLogParser      (13 tests) — all event types, timestamps, edge cases
TestFeatureBuilder     (5 tests)  — windowing, grouping, matrix shape
TestAnomalyDetector    (6 tests)  — fitting, scoring, ranking, error handling
TestLLMNarrator        (6 tests)  — all template patterns, offline mode
TestEndToEndPipeline   (1 test)   — full pipeline smoke test on sample log
```

### Testing Philosophy
- **Unit tests** per pipeline stage — isolate parser logic from ML logic from narration logic
- **Synthetic attack injection** — tests construct known attack patterns and assert correct detection
- **Offline-first** — LLM tests never make real API calls, ensuring tests are fast, free, and deterministic

---

## 📊 Complexity Analysis

| Operation | Time Complexity | Notes |
|---|---|---|
| Log parsing | O(n) | n = number of log lines |
| Feature building | O(n) | Single pass with dict grouping |
| Isolation Forest fit | O(t · ψ · log ψ) | t = trees, ψ = subsample size |
| Isolation Forest score | O(t · log ψ) per point | Very fast at inference time |
| LLM narration | O(k) API calls | k = number of flagged anomalies (bounded by `--top`) |

---

## 📌 Comparison: SentinelAI vs Traditional Approaches

| Method | SentinelAI | Fixed-threshold alerts | Manual review |
|---|---|---|---|
| Detects novel attack patterns | ✅ Yes (unsupervised) | ❌ Only known thresholds | ⚠️ Depends on analyst |
| Scales to large logs | ✅ Yes | ✅ Yes | ❌ No |
| Explains *why* in plain English | ✅ Yes (LLM) | ❌ No | ✅ Yes (if analyst available) |
| Works offline | ✅ Yes | ✅ Yes | ✅ Yes |
| Requires labeled training data | ❌ No | ❌ No | ❌ No |
| Catches multi-signal patterns (e.g. enumeration) | ✅ Yes | ⚠️ Only if explicitly coded | ⚠️ Depends on analyst |
