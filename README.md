# Transformer-Based End-to-End Web Application Firewall (WAF)

> **Problem Statement ID:** 25172;
> **Title:** Transformer-Based End-to-End Web Application Firewall (WAF) Pipeline
> ---

## Overview

A production-grade Web Application Firewall that leverages a **Transformer Autoencoder** to detect malicious HTTP traffic (SQL injection, XSS, path traversal, etc.) through anomaly detection. The system learns the patterns of **normal** HTTP requests during training and flags any request that deviates significantly from that learned representation — making it inherently capable of detecting **zero-day** and **novel** attack vectors without needing labeled attack data.


## Key Features

- **Transformer Autoencoder** — Custom 4-layer Transformer encoder that learns to reconstruct normal HTTP request token sequences; high reconstruction loss indicates anomalous traffic
- **Zero-Day Detection** — Since the model is trained exclusively on benign traffic, any unseen attack pattern naturally produces a high anomaly score
- **Real-Time Inference** — Sub-millisecond detection latency per request via FastAPI + uvicorn
- **Structured Tokenization** — 21 special tokens encode HTTP semantics (method, path, headers, body) with hash-based vocabulary for O(1) tokenization
- **Automatic Normalization** — Regex-based replacement of dynamic values (IPs, timestamps, JWTs, UUIDs, large numbers) with abstract tokens to prevent model drift
- **Nginx Integration** — Reverse proxy with per-IP rate limiting, upstream load balancing, and route-based request forwarding
- **Admin Retraining API** — Upload new benign data and trigger model retraining at runtime without downtime
- **Observability** — Real-time stats dashboard, event logs with filtering, request replay for debugging, and anomaly threshold tuning
- **Docker Compose Deployment** — 7-service architecture with a single `docker compose up` command

## Architecture

```
                              ┌─────────────────────────────────┐
                              │           Client                │
                              └──────────────┬──────────────────┘
                                             │
                                             ▼
                              ┌─────────────────────────────────┐
                              │         Nginx (:80)             │
                              │   Rate Limit · Route · LB       │
                              └──────┬──────────────┬───────────┘
                                     │              │
                            /api/*   │              │  /*
                                     ▼              ▼
                    ┌────────────────────┐   ┌──────────────────────┐
                    │    WAF Detection   │   │   Sample Backend     │
                    │    API (:8000)     │   │   Apps (:5001-5003)  │
                    └────────┬───────────┘   └──────────────────────┘
                             │
              ┌──────────────┼──────────────────┐
              ▼              ▼                   ▼
     ┌─────────────┐ ┌─────────────┐   ┌──────────────────┐
     │  Preprocess │ │  Transformer│   │  SQLite Storage  │
     │  Pipeline   │ │  Autoencoder│   │  (events + data) │
     └─────────────┘ └─────────────┘   └──────────────────┘

  ┌──────────────────────────────────────────────────────────┐
  │                   Preprocessing Pipeline                 │
  │                                                          │
  │  Parse HTTP → Normalize → Tokenize → Tensor → Inference  │
  │                                                          │
  │  ┌────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
  │  │ Parser │→ │Normalizer│→ │Tokenizer │→ │  Model   │    │
  │  │        │  │          │  │          │  │  Encoder │    │
  │  └────────┘  └──────────┘  └──────────┘  └──────────┘    │
  └──────────────────────────────────────────────────────────┘
```


## Tech Stack

| Layer | Technology |
|-------|-----------|
| **ML Framework** | PyTorch (Transformer Autoencoder) |
| **Web Framework** | FastAPI + Uvicorn |
| **Reverse Proxy** | Nginx Alpine |
| **Rate Limiting** | Nginx `limit_req_zone` + Redis |
| **Storage** | SQLite (detection events + training data) |
| **Containerization** | Docker + Docker Compose |
| **Sample Backends** | Flask 3.0 |
| **Logging** | Loguru (rotating, compressed) |
| **Configuration** | Pydantic Settings + YAML |


## Project Structure

```
.
├── config.yaml                  # Central configuration (model, API, training, storage)
├── docker-compose.yml           # 7-service orchestration
├── Dockerfile                   # Shared image for detection + update APIs
├── requirements.txt             # Python dependencies
├── integration/
│   └── nginx.conf               # Nginx reverse proxy configuration
├── scripts/
│   ├── demo.sh                  # End-to-end demo (benign + attack requests)
│   ├── generate_benign.py       # Synthetic benign request generator
│   └── train.py                 # CLI training script
├── src/
│   ├── api/
│   │   ├── detection_api.py     # Core detection endpoints (detect, logs, stats, replay)
│   │   └── update_api.py        # Admin endpoints (retrain, upload benign data)
│   ├── ingestion/
│   │   ├── batch.py             # Log file batch ingestion
│   │   └── stream.py            # Async queue-based streaming ingestion
│   ├── models/
│   │   ├── transformer.py       # Custom Transformer Autoencoder (PyTorch)
│   │   └── trainer.py           # Training loop with early stopping + checkpointing
│   ├── preprocessing/
│   │   ├── parser.py            # HTTP request + access log parsing
│   │   ├── normalizer.py        # Regex-based dynamic value normalization
│   │   └── tokenizer.py         # Structured tokenization with 21 special tokens
│   ├── storage/
│   │   └── db.py                # SQLite storage layer (events + training data)
│   └── utils/
│       ├── config.py            # Pydantic-based settings loader
│       └── logger.py            # Loguru configuration
├── sample_apps/
│   ├── app1/                    # E-commerce demo backend (port 5001)
│   ├── app2/                    # User/Orders demo backend (port 5002)
│   └── app3/                    # Upload API demo backend (port 5003)
└── tests/
    └── __init__.py              # (Tests to be implemented)
```


## How It Works

### 1. Training (Learn Normal Traffic)

```
Benign HTTP Request JSON
        │
        ▼
  Parse → Normalize → Tokenize → [CLS] <METHOD> GET <PATH> /products <SEP>
        │
        ▼
  Transformer Autoencoder learns to reconstruct the token sequence
        │
        ▼
  Loss = Cross-Entropy (predicted vs actual tokens)
        │
        ▼
  Threshold = 95th percentile of training losses
```

The model is trained **only on benign (normal) traffic**. It learns to reconstruct the token sequences of legitimate HTTP requests with low loss.

### 2. Detection (Flag Anomalies)

```
Incoming HTTP Request
        │
        ▼
  Parse → Normalize → Tokenize → Tensor
        │
        ▼
  Transformer Autoencoder → Reconstructed Token Logits
        │
        ▼
  Anomaly Score = Mean Cross-Entropy Loss per token
        │
        ▼
  score > threshold? ──→ YES: Block (anomaly)
       │
       └──→ NO:  Allow (benign)
```

Malicious requests (SQLi, XSS, path traversal) contain token patterns the model has **never seen during training**, resulting in high reconstruction loss and detection as anomalies.

### 3. Tokenization

Requests are encoded into a structured format:

```
<CLS> <METHOD> get <PATH> /api /users /123 <PARAM> search <EQ> <script>alert(1)</script> <SEP>
```

- **21 special tokens** for HTTP structure (`<METHOD>`, `<PATH>`, `<PARAM>`, `<HEADER>`, `<BODY>`, etc.)
- **21 special tokens** for structural characters (`<EQ>`, `<AMP>`, `<SLASH>`, `<QMARK>`, etc.)
- **Hash-based vocabulary**: text tokens → `hash(token) % vocab_size` for deterministic, memory-efficient mapping

### 4. Normalization

Dynamic values are replaced before tokenization to prevent model confusion:

| Pattern | Replaced With |
|---------|--------------|
| IP addresses | `<IP>` |
| Timestamps | `<TS>` |
| UUIDs | `<UUID>` |
| JWTs | `<TOKEN>` |
| Large numbers (10+ digits) | `<NUM>` |
| Emails | `<EMAIL>` |

## Model Architecture---
```
┌──────────────────────────────────────────┐
│         Transformer Autoencoder          │
│                                          │
│   Input Token IDs                        │
│        │                                 │
│        ▼                                 │
│   Embedding(10000, 256) + √d scaling     │
│        │                                 │
│        ▼                                 │
│   Sinusoidal Positional Encoding         │
│        │                                 │
│        ▼                                 │
│   ┌──────────────────────────────┐       │
│   │  Transformer Encoder (×4)    │       │
│   │  ┌────────────────────────┐  │       │
│   │  │ Multi-Head Attention   │  │       │
│   │  │ (8 heads, 256d)        │  │       │
│   │  ├────────────────────────┤  │       │
│   │  │ Feed-Forward Network   │  │       │
│   │  │ (256 → 1024 → 256)    │  │        │
│   │  │ GELU Activation        │  │       │
│   │  └────────────────────────┘  │       │
│   └──────────────────────────────┘       │
│        │                                 │
│        ▼                                 │
│   Decoder MLP: 256 → 512 → 256 (GELU)    │
│        │                                 │
│        ▼                                 │
│   Linear(256, 10000) → Token Logits      │
│                                          │
│   Anomaly Score = CE Loss(logits, input) │
└──────────────────────────────────────────┘
```

| Parameter | Value |
|-----------|-------|
| Vocabulary Size | 10,000 |
| Embedding Dimension | 256 |
| Attention Heads | 8 |
| Transformer Layers | 4 |
| FFN Dimension | 1,024 |
| Max Sequence Length | 512 |
| Dropout | 0.1 |
| Activation | GELU |
| Anomaly Threshold | 0.75 (auto-tuned at 95th percentile) |

## Getting Started

### Prerequisites

- Docker & Docker Compose (v2+)
- 4 GB RAM minimum (for model training)

### Quick Start

```bash
# Clone the repository
git clone https://github.com/Bluepill0x00/Transformer_based_WAF_SIH.git
cd Transformer_based_WAF_SIH

# Start all services
docker compose up -d

# Verify services are running
docker compose ps
```

The following services will be available:

| Service | URL |
|---------|-----|
| **Nginx Gateway** | `http://localhost:80` |
| **Detection API** | `http://localhost:80/api/detect` |
| **Update API** | `http://localhost:80/api/retrain` |
| **Dashboard** | `http://localhost:80/api/history` |
| **Sample Apps** | `http://localhost/` |

### Training the Model

```bash
# Generate synthetic benign training data (500 samples)
docker compose exec waf-api python scripts/generate_benign.py

# Train the model inside the container
docker compose exec waf-api python scripts/train.py \
    --num-samples 500 \
    --epochs 50 \
    --batch-size 32

# Or train from the host (if running locally)
pip install -r requirements.txt
python scripts/generate_benign.py --num-samples 1000
python scripts/train.py --num-samples 1000 --epochs 50
```

### Running the Demo

```bash
# Run the full demo (benign + attack requests)
bash scripts/demo.sh
```


## API Reference

### Detection API (`:8000`)

All endpoints (except `/health` and `/history`) require the `X-API-Key` header.

#### `POST /detect` — Analyze a single request

```bash
curl -X POST http://localhost/api/detect \
  -H "Content-Type: application/json" \
  -H "X-API-Key: waf-key-dev-001" \
  -d '{
    "method": "GET",
    "path": "/products",
    "query_params": {"search": ["laptop"]},
    "headers": {"User-Agent": "Mozilla/5.0"},
    "body": ""
  }'
```

**Response:**
```json
{
  "request_id": "uuid",
  "anomaly_score": 0.32,
  "is_anomaly": false,
  "latency_ms": 12.5,
  "model_version": "1.0.0"
}
```

#### `POST /detect/batch` — Analyze multiple requests

```bash
curl -X POST http://localhost/api/detect/batch \
  -H "Content-Type: application/json" \
  -H "X-API-Key: waf-key-dev-001" \
  -d '{
    "requests": [
      {"method": "GET", "path": "/products", "headers": {}},
      {"method": "GET", "path": "/admin' OR 1=1--", "headers": {}}
    ]
  }'
```

#### `GET /logs` — Query detection logs

```bash
# All events
curl http://localhost/api/logs?limit=50 -H "X-API-Key: waf-key-dev-001"

# Anomalies only
curl "http://localhost/api/logs?anomaly_only=true&limit=20" -H "X-API-Key: waf-key-dev-001"

# Filter by method
curl "http://localhost/api/logs?method=POST" -H "X-API-Key: waf-key-dev-001"
```

#### `GET /stats` — Aggregate statistics

```bash
curl http://localhost/api/stats -H "X-API-Key: waf-key-dev-001"
```

**Response:**
```json
{
  "total_requests": 1500,
  "anomalies_detected": 45,
  "benign_requests": 1455,
  "anomaly_rate": 3.0,
  "avg_anomaly_score": 0.28,
  "method_distribution": {"GET": 900, "POST": 500, "PUT": 100}
}
```

#### `POST /replay/{request_id}` — Replay a stored request

```bash
curl -X POST http://localhost/api/replay/{request_id} \
  -H "X-API-Key: waf-key-dev-001"
```

#### `POST /threshold` — Update anomaly threshold

```bash
curl -X POST http://localhost/api/threshold \
  -H "Content-Type: application/json" \
  -H "X-API-Key: waf-key-dev-001" \
  -d '{"threshold": 0.85}'
```

#### `GET /history` — Dashboard (HTML)

Open in browser: `http://localhost/api/history`


### Update API (`:8001`)

Requires admin API key (`waf-admin-key-001`).

#### `POST /add_benign_data` — Upload benign request samples

```bash
curl -X POST http://localhost/api/add_benign_data \
  -H "Content-Type: application/json" \
  -H "X-API-Key: waf-admin-key-001" \
  -d '{
    "requests": [
      {"method": "GET", "path": "/products", "headers": {}},
      {"method": "POST", "path": "/login", "headers": {}, "body": "user=test"}
    ]
  }'
```

#### `POST /retrain` — Trigger model retraining

```bash
curl -X POST http://localhost/api/retrain \
  -H "X-API-Key: waf-admin-key-001"
```

#### `GET /retrain/status` — Check training progress

```bash
curl http://localhost/api/retrain/status \
  -H "X-API-Key: waf-admin-key-001"
```

**Response:**
```json
{
  "is_training": true,
  "current_epoch": 12,
  "train_loss": 2.34,
  "val_loss": 2.51,
  "status": "training"
}
```

## Configuration

All settings are in `config.yaml`. Key sections:

```yaml
app:
  name: waf-project
  version: "1.0.0"

detection_api:
  host: "0.0.0.0"
  port: 8000
  api_keys:
    - "waf-key-dev-001"
    - "waf-key-dev-002"

update_api:
  host: "0.0.0.0"
  port: 8001
  admin_api_keys:
    - "waf-admin-key-001"

model:
  vocab_size: 10000
  embed_dim: 256
  num_heads: 8
  num_layers: 4
  max_seq_len: 512
  ff_dim: 1024
  dropout: 0.1
  anomaly_threshold: 0.75

training:
  batch_size: 32
  learning_rate: 0.0001
  num_epochs: 50
  early_stopping_patience: 5
  validation_split: 0.1

preprocessing:
  replace_patterns:
    ip_address: "(?<!\\d)\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}(?!\\d)"
    timestamp: "\\d{4}-\\d{2}-\\d{2}[T ]\\d{2}:\\d{2}"
    # ... more patterns

storage:
  db_path: "data/detections.db"
  max_events: 100000

nginx:
  upstreams:
    detection_api: "waf-api:8000"
    update_api: "waf-update-api:8001"
    backend_apps:
      - "sample-app-1:5001"
      - "sample-app-2:5002"
      - "sample-app-3:5003"
```

## Docker Services

| Service | Container | Port | Description |
|---------|-----------|------|-------------|
| `nginx` | nginx:alpine | 80 | Reverse proxy, rate limiting, routing |
| `waf-api` | Custom | 8000 | Detection API (inference) |
| `waf-update-api` | Custom | 8001 | Admin API (retraining, data upload) |
| `redis` | redis:7-alpine | 6379 | Rate limiting backend |
| `sample-app-1` | Flask | 5001 | Demo backend |
| `sample-app-2` | Flask | 5002 | Demo backend |
| `sample-app-3` | Flask | 5003 | Demo backend |

```bash
# View logs
docker compose logs -f waf-api

# Restart a single service
docker compose restart waf-api

# Stop everything
docker compose down
```

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project was developed an MVP required for internal college hackathon **Smart India Hackathon 2026** under Problem Statement ID **25172**.

<p align="center">Built with Transformer Attention for a Safer Web</p>
