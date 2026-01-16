# Agentic Data Pipeline

An end-to-end **intelligent, self-correcting data pipeline** built with Apache Airflow 3.x that performs sentiment analysis on Yelp reviews using local LLM inference with Ollama.

![Python](https://img.shields.io/badge/Python-3.11+-blue.svg)
![Airflow](https://img.shields.io/badge/Apache%20Airflow-3.x-green.svg)
![Ollama](https://img.shields.io/badge/Ollama-LLaMA%203.2-orange.svg)
![License](https://img.shields.io/badge/License-MIT-yellow.svg)

## 🎯 Overview

This pipeline demonstrates a **production-grade, self-healing architecture** that:

- **Automatically detects and repairs** malformed, missing, or corrupted review data
- **Performs sentiment analysis** using a local LLM (LLaMA 3.2 via Ollama)
- **Provides health monitoring** with real-time pipeline health reports
- **Supports batch processing** with configurable parallelism
- **Gracefully degrades** when services are unavailable

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AIRFLOW DAG                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐    ┌──────────────┐                               │
│  │  load_model  │    │ load_reviews │                               │
│  │   (Ollama)   │    │   (JSON)     │                               │
│  └──────┬───────┘    └──────┬───────┘                               │
│         │                   │                                        │
│         │                   ▼                                        │
│         │         ┌─────────────────────┐                           │
│         │         │ diagnose_and_heal   │  ◄── Self-Healing Logic   │
│         │         │      _batch         │                           │
│         │         └──────────┬──────────┘                           │
│         │                    │                                       │
│         ▼                    ▼                                       │
│  ┌──────────────────────────────────────┐                           │
│  │      batch_analyze_sentiment         │  ◄── LLM Inference        │
│  │           (Ollama)                   │                           │
│  └──────────────────┬───────────────────┘                           │
│                     │                                                │
│                     ▼                                                │
│  ┌──────────────────────────────────────┐                           │
│  │        aggregate_results             │  ◄── Results & Metrics    │
│  └──────────────────┬───────────────────┘                           │
│                     │                                                │
│                     ▼                                                │
│  ┌──────────────────────────────────────┐                           │
│  │      generate_health_report          │  ◄── Health Monitoring    │
│  └──────────────────────────────────────┘                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## ✨ Features

### 🔧 Self-Healing Capabilities

The pipeline automatically detects and heals various data quality issues:

| Issue Type | Detection | Healing Action |
|------------|-----------|----------------|
| Missing text | `text is None` | Fill with placeholder |
| Empty text | `text.strip() == ""` | Fill with placeholder |
| Wrong type | `not isinstance(text, str)` | Type conversion |
| Special chars only | No alphanumeric chars | Replace with marker |
| Text too long | `len(text) > 2000` | Truncate with ellipsis |

### 📊 Health Status Monitoring

The pipeline reports health status based on processing metrics:

- **HEALTHY**: All records processed successfully
- **WARNING**: >50% records required healing
- **DEGRADED**: Some records failed but recovered
- **CRITICAL**: >10% records in degraded state

### 🚀 Batch Processing

Process millions of records with configurable batch sizes and parallelism:

```bash
# Process 5M records with 5 parallel DAG runs
python scripts/batch_runner.py --total 5000000 --batch-size 1000 --parallel 5
```

## 📁 Project Structure

```
agentic-data-pipeline/
├── dags/
│   └── agentic_pipeline_dag.py   # Main Airflow DAG
├── scripts/
│   └── batch_runner.py           # Batch processing utility
├── input/
│   └── yelp_academic_dataset_review.json  # Sample data (11K reviews)
├── output/                        # Generated analysis results
├── logs/                          # Airflow logs
├── models/                        # Ollama model cache
├── airflow.cfg                    # Airflow configuration
├── requirements.txt               # Python dependencies
└── README.md                      # This file
```

## 🛠️ Installation

### Prerequisites

- **Python 3.11+**
- **Ollama** - [Download here](https://ollama.com/download)
- **Git** (optional, for cloning)

### Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/farhanarif67576/Hello-world.git
cd Hello-world

# 2. Create virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Set Airflow home
export AIRFLOW_HOME=$(pwd)

# 5. Initialize Airflow database
airflow db migrate

# 6. Pull the LLM model
ollama pull llama3.2:1b

# 7. Start services
ollama serve &
airflow scheduler &

# 8. Wait for DAG registration and unpause
sleep 15
airflow dags reserialize
airflow dags unpause self_healing_pipeline
```

## 🚀 Usage

### Single DAG Run

```bash
# Trigger with default parameters (100 reviews)
airflow dags trigger self_healing_pipeline

# Trigger with custom parameters
airflow dags trigger self_healing_pipeline --conf '{"batch_size": 500, "offset": 0}'
```

### Batch Processing

```bash
# Process 500 records with 100 per batch, 5 parallel runs
python scripts/batch_runner.py --total 500 --batch-size 100 --parallel 5

# Dry run (preview without executing)
python scripts/batch_runner.py --total 5000 --batch-size 1000 --dry-run

# Resume from specific offset
python scripts/batch_runner.py --total 5000 --batch-size 1000 --start 2000
```

### Command Line Options

| Option | Description | Default |
|--------|-------------|---------|
| `--total` | Total records to process | Required |
| `--batch-size` | Records per DAG run | 1000 |
| `--parallel` | Concurrent DAG runs | 1 |
| `--start` | Starting offset | 0 |
| `--delay` | Delay between triggers (sec) | 1.0 |
| `--dry-run` | Preview without executing | False |

## ⚙️ Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `PIPELINE_BASE_DIR` | Base directory | `/workspaces/Hello-world` |
| `PIPELINE_INPUT_FILE` | Input JSON file path | `{BASE_DIR}/input/yelp_...json` |
| `PIPELINE_OUTPUT_DIR` | Output directory | `{BASE_DIR}/output/` |
| `PIPELINE_MAX_TEXT_LENGTH` | Max text length before truncation | 2000 |
| `OLLAMA_HOST` | Ollama server URL | `http://localhost:11434` |
| `OLLAMA_MODEL` | LLM model name | `llama3.2:1b` |
| `OLLAMA_TIMEOUT` | Request timeout (sec) | 120 |
| `OLLAMA_RETRIES` | Retry attempts | 3 |

### DAG Parameters

Parameters can be passed at runtime via `--conf`:

```json
{
  "input_file": "/path/to/reviews.json",
  "batch_size": 100,
  "offset": 0,
  "ollama_model": "llama3.2:1b"
}
```

## 📊 Output Format

Each run generates a JSON summary file in `output/`:

```json
{
  "run_info": {
    "timestamp": "2026-01-16T05:42:26.571435",
    "batch_size": 100,
    "offset": 0,
    "input_file": "/path/to/reviews.json"
  },
  "totals": {
    "processed": 100,
    "success": 95,
    "healed": 5,
    "degraded": 0
  },
  "rates": {
    "success_rate": 0.95,
    "healing_rate": 0.05,
    "degradation_rate": 0.0
  },
  "sentiment_distribution": {
    "POSITIVE": 60,
    "NEGATIVE": 25,
    "NEUTRAL": 15
  },
  "star_sentiment_correlation": {
    "5_star": {"POSITIVE": 45, "NEGATIVE": 2, "NEUTRAL": 3},
    "1_star": {"POSITIVE": 1, "NEGATIVE": 18, "NEUTRAL": 1}
  },
  "average_confidence": {
    "success": 0.92,
    "healed": 0.85,
    "degraded": 0.5
  }
}
```

## 🧪 Testing

```bash
# Test DAG parsing
python dags/agentic_pipeline_dag.py

# Verify DAG is registered
airflow dags list | grep self_healing_pipeline

# Test single task
airflow tasks test self_healing_pipeline load_reviews 2025-12-07
```

## 🔍 Monitoring

### Check Pipeline Status

```bash
# View DAG runs
airflow dags list-runs -d self_healing_pipeline

# Check task logs
airflow tasks logs self_healing_pipeline load_reviews <run_id>
```

### Health Report Example

```json
{
  "pipeline": "self_healing_pipeline",
  "health_status": "HEALTHY",
  "metrics": {
    "total_processed": 100,
    "success_rate": 1.0,
    "healing_rate": 0.0,
    "degradation_rate": 0.0
  }
}
```

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 👤 Author

**Farhan Arif**

## 🙏 Acknowledgments

- [Apache Airflow](https://airflow.apache.org/) - Workflow orchestration
- [Ollama](https://ollama.com/) - Local LLM inference
- [Yelp Dataset](https://www.yelp.com/dataset) - Sample review data