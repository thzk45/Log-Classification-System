# Log Classification System

A hybrid log classification system that intelligently categorizes log messages using a three-tier pipeline: **Regex** → **BERT (Sentence Transformers + Logistic Regression)** → **LLM (Groq / DeepSeek)**. Exposed as a REST API via FastAPI.

---

## How It Works

Log messages are routed through the pipeline based on their source system:

```
Incoming Log
     │
     ├─── source == "LegacyCRM" ──────────────────► LLM Classifier (Groq)
     │                                               (Workflow Error / Deprecation Warning)
     │
     └─── All other sources
               │
               ├─► Regex Classifier
               │     └─ matched? ──► return label
               │
               └─► BERT Classifier (fallback)
                     (Sentence Transformer + Logistic Regression)
```

| Processor | File | When Used |
|---|---|---|
| Regex | `processor_regex.py` | Fast pattern matching for structured log formats |
| BERT | `processor_bert.py` | Semantic classification for unstructured logs |
| LLM | `processor_llm.py` | LegacyCRM logs requiring reasoning (via Groq API) |

---

## Project Structure

```
Log-Classification-System/
├── classify.py            # Orchestration logic — routes logs through the pipeline
├── server.py              # FastAPI server — exposes POST /classify/ endpoint
├── processor_regex.py     # Tier 1: Regex-based classification
├── processor_bert.py      # Tier 2: BERT + Logistic Regression classification
├── processor_llm.py       # Tier 3: LLM classification via Groq API
├── requirements.txt       # Python dependencies
├── .env.example           # Environment variable template
├── models/                # Saved model files (.joblib, embeddings)
├── resources/             # Sample CSV files and output
└── training/              # Training notebooks / scripts for BERT model
```

---

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/thzk45/Log-Classification-System.git
cd Log-Classification-System
```

### 2. Create a virtual environment

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure environment variables

```bash
cp .env.example .env
```

Open `.env` and add your Groq API key:

```
GROQ_API_KEY=your_groq_api_key_here
```

Get a free key at [console.groq.com](https://console.groq.com).

### 5. Start the server

```bash
uvicorn server:app --reload
```

The API will be available at `http://127.0.0.1:8000`.

---

## API Usage

### `POST /classify/`

Upload a CSV file containing log messages. The file must have two columns:

| Column | Description |
|---|---|
| `source` | The system that generated the log (e.g. `ModernCRM`, `LegacyCRM`, `BillingSystem`) |
| `log_message` | The raw log text |

#### Example request

```bash
curl -X POST "http://127.0.0.1:8000/classify/" \
  -H "accept: application/json" \
  -F "file=@resources/test_logs.csv"
```

#### Example input CSV

```csv
source,log_message
ModernCRM,IP 192.168.1.1 blocked due to potential attack
BillingSystem,User 12345 logged in successfully
LegacyCRM,Case escalation for ticket ID 7324 failed because the assigned support agent is no longer active.
LegacyCRM,The ReportGenerator module will be retired in version 4.0.
```

#### Example output CSV

The response is a CSV file identical to the input, with an added `target_label` column:

```csv
source,log_message,target_label
ModernCRM,IP 192.168.1.1 blocked due to potential attack,Security Alert
BillingSystem,User 12345 logged in successfully,User Activity
LegacyCRM,Case escalation for ticket ID 7324 failed...,Workflow Error
LegacyCRM,The ReportGenerator module will be retired...,Deprecation Warning
```

#### Interactive docs

FastAPI provides automatic docs at `http://127.0.0.1:8000/docs`.

---

## Classification Labels

| Label | Description |
|---|---|
| `Security Alert` | Blocked IPs, access escalation, intrusion attempts |
| `User Activity` | Logins, logouts, user actions |
| `File Operations` | Uploads, downloads, file management |
| `System Operations` | Backups, reboots, maintenance tasks |
| `HTTP Request` | API/HTTP access logs with status codes |
| `Workflow Error` | Failed processes, broken workflows (LegacyCRM) |
| `Deprecation Warning` | Feature retirement notices (LegacyCRM) |
| `Unclassified` | Logs that don't match any known pattern |

---

## Requirements

- Python 3.9+
- A [Groq API key](https://console.groq.com) (free tier available)

---

## License

MIT
