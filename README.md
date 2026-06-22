# 🛡️ SentinelAI — AI-Powered Linux Auth Log Threat Detector

> A Python CLI tool that combines **unsupervised machine learning** (Isolation Forest) with **LLM narration** (Claude) to detect and explain suspicious authentication activity in Linux `/var/log/auth.log` files — in plain English.

```
   _____           __  _            _    _____
  / ____|         / / (_)          | |  / ____|
 | (___   ___ ___/ /_  _ _ __   ___| | | |  __  ___ _ __
  \___ \ / _ Y __| __|| | '_ \ / _ \ | | | |_ |/ _ Y '_ \
  ____) |  __|\__ \ |_ | | | | |  __/ |_| |__| |  __/ | | |
 |_____/ \___||___/\__||_|_| |_|\___|\___/\_____|\___|_| |_|
```

---

## 🎯 What It Does

```
ML detects anomalies  →  LLM explains them in plain English
```

| Stage | Technique | Output |
|---|---|---|
| 1️⃣ **Parse** | Regex-based syslog parser | Structured login/sudo/session events |
| 2️⃣ **Feature Engineer** | Behavioral aggregation per IP/time-window | Numeric feature vectors |
| 3️⃣ **Detect** | Isolation Forest (unsupervised ML) | Risk score 0-100 per behavioral window |
| 4️⃣ **Explain** | Claude LLM (or offline templates) | Plain-English summary + attack pattern + recommended action |

**No labeled attack data needed.** Isolation Forest learns what "normal" looks like for *your* logs and flags statistical outliers — brute force, credential stuffing, account enumeration, and privilege escalation patterns all surface automatically.

---

## ⚡ Quick Start

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Run on the bundled sample log (includes 3 real attack patterns)
python src/sentinel.py --log sample_logs/sample_auth.log

# 3. Run on your own system's auth log
sudo python src/sentinel.py --log /var/log/auth.log
```

That's it — works immediately, even without an API key (falls back to offline narration).

---

## 📸 Example Output

```
🚨 4 anomalous behavior window(s) detected
============================================================

[1] Source: 203.0.113.55 | Window: 2026-06-13 09:10:00 | Risk: 100/100
------------------------------------------------------------
SUMMARY: Source 203.0.113.55 triggered a risk score of 100/100 with 9 failed
login(s), 6 invalid-user attempt(s), and 7 distinct username(s) tried, at a
rate of 16.0 events/minute.
LIKELY ATTACK PATTERN: Account enumeration / credential stuffing
RECOMMENDED ACTION: Rate-limit or block 203.0.113.55; verify no usernames
leaked from this list match real accounts.

[2] Source: local:support | Window: 2026-06-13 02:30:00 | Risk: 93/100
------------------------------------------------------------
SUMMARY: Source local:support triggered a risk score of 93/100 with 4 sudo
command(s) against 2 distinct target user(s).
LIKELY ATTACK PATTERN: Suspicious privilege escalation attempts
RECOMMENDED ACTION: Audit sudoers configuration and review the user's
recent command history immediately.
```

---

## 🔬 Why Isolation Forest?

| Property | Why it matters for log analysis |
|---|---|
| **Unsupervised** | No need for pre-labeled "this was an attack" training data — works on *your* logs from day one |
| **Fast** | O(n log n) — scales to large, busy production servers |
| **Built for outliers** | Isolates anomalies by random partitioning; anomalies need fewer splits to isolate than normal points |
| **Tabular-native** | Works directly on the behavioral features we engineer (failed logins, distinct usernames, request rate, etc.) |

## 🧠 Why pair it with an LLM?

A risk score of `93/100` tells you *something* is wrong. It doesn't tell you **what** or **what to do**. The LLM narrator translates ML output into a SOC-analyst-style explanation:

```
ML Model says:        risk_score=93, sudo_count=4, distinct_targets=2
LLM Narrator says:    "Suspicious privilege escalation — audit sudoers
                       config and review command history immediately."
```

**Design principle:** the LLM never *decides* what's anomalous — that stays 100% with the deterministic, auditable ML model. The LLM only *explains* already-flagged findings, so detection logic never depends on LLM judgment calls.

---

## 🚀 Installation

See **[INSTALL.md](INSTALL.md)** for full step-by-step setup (Windows/Linux/macOS).

```bash
git clone https://github.com/YOUR_USERNAME/SentinelAI.git
cd SentinelAI
pip install -r requirements.txt
cp .env.example .env   # optional — only needed for LLM narration
```

---

## 🖥️ Usage

```bash
python src/sentinel.py --log <path-to-auth.log> [options]
```

| Flag | Description | Default |
|---|---|---|
| `--log`, `-l` | Path to auth.log file (**required**) | — |
| `--top`, `-n` | Number of top anomalies to report | `10` |
| `--contamination`, `-c` | Expected fraction of anomalous behavior (0.0–0.5) | `0.1` |
| `--window`, `-w` | Time window in minutes for behavioral aggregation | `10` |
| `--json`, `-j` | Write full structured results to a JSON file | — |
| `--no-llm` | Force offline template narration (skip API call) | off |
| `--year` | Year to assume for log timestamps | current year |
| `--no-banner` | Suppress ASCII banner | off |

### Examples

```bash
# Basic scan
python src/sentinel.py --log /var/log/auth.log

# More sensitive detection (flags more as anomalous)
python src/sentinel.py --log /var/log/auth.log --contamination 0.2

# Export full results for SIEM ingestion
python src/sentinel.py --log /var/log/auth.log --json report.json

# Offline mode — no API key, no internet call
python src/sentinel.py --log /var/log/auth.log --no-llm
```

---

## 🧩 How It Works

```
/var/log/auth.log
       │
       ▼
┌─────────────────┐
│   log_parser.py │  Regex-parses syslog lines into structured events:
│                 │  failed_login, accepted_login, invalid_user,
│                 │  sudo_command, session_open/close, disconnect
└────────┬────────┘
         ▼
┌─────────────────┐
│feature_builder.py│ Groups events by (source_ip, 10-min window) and
│                 │  computes 11 behavioral features per window:
│                 │  failed_login_count, distinct_usernames_tried,
│                 │  events_per_minute, night_time_activity, etc.
└────────┬────────┘
         ▼
┌──────────────────┐
│anomaly_detector.py│ Isolation Forest scores each window 0-100.
│                  │  Higher score = more statistically abnormal
│                  │  compared to the rest of THIS log's behavior.
└────────┬─────────┘
         ▼
┌─────────────────┐
│ llm_narrator.py │  For each flagged window, asks Claude (or uses
│                 │  offline templates) to explain WHY it's
│                 │  suspicious, in plain English.
└────────┬────────┘
         ▼
   Threat Report (terminal + optional JSON export)
```

Full deep-dive with diagrams: **[HOW_IT_WORKS.md](HOW_IT_WORKS.md)**

---

## 📁 Project Structure

```
SentinelAI/
├── src/
│   ├── sentinel.py          ← Main CLI entry point
│   ├── log_parser.py        ← Regex-based auth.log parser
│   ├── feature_builder.py   ← Behavioral feature engineering
│   ├── anomaly_detector.py  ← Isolation Forest ML model
│   └── llm_narrator.py      ← Claude LLM narration + offline fallback
├── tests/
│   └── test_sentinel.py     ← 31 unit + integration tests
├── sample_logs/
│   └── sample_auth.log      ← Realistic log with 3 embedded attack patterns
├── docs/
│   └── (additional guides)
├── requirements.txt
├── .env.example
├── .gitignore
├── INSTALL.md
├── HOW_IT_WORKS.md
├── TOOLS_AND_TECHNIQUES.md
├── LICENSE
└── README.md
```

---

## 🧪 Testing

```bash
python -m pytest tests/ -v
```

**31 tests covering:**
- Log parsing (all event types, edge cases, malformed lines)
- Feature engineering (windowing, aggregation correctness)
- Anomaly detection (fitting, scoring, ranking, error handling)
- LLM narration (all attack-pattern templates, offline mode)
- Full end-to-end pipeline integration

---

## 🔐 Privacy & Security Notes

- SentinelAI reads log files **locally** — nothing is uploaded except the small, already-anonymized behavioral *summary* sent to Claude for narration (counts and rates, not raw log lines or full IPs in identifying contexts beyond what's needed)
- Running with `--no-llm` guarantees **zero external network calls**
- Never commit real production `auth.log` files to version control — see `.gitignore`

---

## 🛠️ Tech Stack

- **Python 3.10+**
- **scikit-learn** — Isolation Forest implementation
- **pandas / numpy** — feature engineering & data handling
- **anthropic** — Claude API client for LLM narration
- **pytest** — testing framework

Full technical breakdown: **[TOOLS_AND_TECHNIQUES.md](TOOLS_AND_TECHNIQUES.md)**

---

## 📈 Future Enhancements

- [ ] Real-time log tailing (`tail -f` style continuous monitoring)
- [ ] Slack/email alerting integration
- [ ] Support for additional log formats (Windows Event Log, cloud audit logs)
- [ ] Web dashboard (FastAPI + React) for visual exploration
- [ ] GeoIP enrichment for source IP context
- [ ] Ensemble detection (combine Isolation Forest with LSTM sequence modeling)

---

## 📜 License

MIT License — free to use, modify, and distribute.
