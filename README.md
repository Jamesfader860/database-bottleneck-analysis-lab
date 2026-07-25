# Multi-Layer Performance Tuning: AWS RDS PostgreSQL & Flask API

## 📐 System Architecture

```
                        ┌─────────────────────────────────────────┐
                        │      Multi-Threaded Load Client         │
                        │      (1,000 Requests / 20 Threads)      │
                        └────────────────────┬────────────────────┘
                                             │ HTTP / REST
                                             ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ AWS EC2 (Application Layer)                                                            │
│                                                                                        │
│   ┌────────────────────────────────────────────────────────────────────────────────┐   │
│   │ Gunicorn WSGI Server (4 Sync Workers)                                          │   │
│   │   └─► Parallel request handling under thread concurrency                       │   │
│   └────────────────────────────────────────┬───────────────────────────────────────┘   │
│                                            │                                           │
│                                            ▼                                           │
│   ┌────────────────────────────────────────────────────────────────────────────────┐   │
│   │ Flask Search API (`/search`)                                                   │   │
│   │   └─► `psycopg2.pool.ThreadedConnectionPool` (5 min / 20 max connections)      │   │
│   └────────────────────────────────────────┬───────────────────────────────────────┘   │
└────────────────────────────────────────────┼───────────────────────────────────────────┘
                                             │ Reused Persistent TCP/TLS Connections
                                             ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ AWS RDS PostgreSQL (Database Layer)                                                    │
│                                                                                        │
│   • Table: `customers` (100,000+ rows)                                                 │
│   • Extension: `pg_trgm` (Generalized Inverted Trigram Indexing)                       │
│   • Active Indexes: `idx_customers_email_trgm`, `idx_customers_bio_trgm`               │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

A systematic performance optimization project demonstrating how evidence-based troubleshooting across the database, network, and application layers resolved severe search latency and database CPU spikes under load.

---

## 🎯 Business Problem & System Symptoms

During periods of high traffic, our customer search API (`/search`) became extremely sluggish, causing request queuing and near-100% CPU utilization on our primary database.

### Symptoms Under Load (1,000 Concurrent Requests / 20 Threads)
* **High User Latency:** Average response times spiked above **220 ms** (with individual requests taking up to 550 ms).
* **Slow Throughput:** The 1,000-request benchmark took **over 11 seconds** to complete.
* **Resource Exhaustion:** AWS RDS CPU utilization spiked heavily, threatening overall database availability.

---

## 🕵️ Systematic Investigation (Following the Evidence)

Instead of guessing or immediately scaling up AWS infrastructure, I investigated each layer of the architecture individually using AWS CloudWatch and PostgreSQL `EXPLAIN ANALYZE`:

* **Compute Layer:** CloudWatch metrics confirmed EC2 CPU and RAM remained low and stable—ruling out server compute capacity as the primary bottleneck.
* **Database Layer:** Query execution plans revealed PostgreSQL was performing a full `Seq Scan` across 100,000 rows. Because the search query used leading wildcards (`LIKE '%term%'`), standard B-Tree indexes were bypassed.
* **Network / Connection Layer:** Telemetry showed every single incoming request established a brand-new TCP/TLS connection to RDS, adding ~150 ms of network handshake latency before the query even ran.
* **Application Server Layer:** Flask’s built-in development server was executing synchronously on a single thread, queuing concurrent traffic under high request volume.

---

## 💡 Why Not Vertically Scale RDS Hardware First?

> **Engineering Principle:** Scaling infrastructure without diagnosing root-cause software bottlenecks increases monthly AWS costs while leaving underlying inefficient queries unchanged.
> 
> The evidence proved that inefficient query execution plans and connection overhead—not instance capacity—were driving the CPU spikes and latency. Fixing the software layer preserved cloud budget while permanently resolving the bottleneck.

---

## 🧪 Staged Optimization & Incremental Results

To measure cause-and-effect, optimizations were implemented and benchmarked in stages:

| Optimization Stage | Primary Change Made | Avg Response Time | Total Test Duration | Key Finding / Impact |
|---|---|---|---|---|
| **0. Unoptimized Baseline** | Standard Flask + Unindexed DB | **220.14 ms** | **11.02 sec** | Postgres forced into `Seq Scan` across 100k rows (~118ms DB time). |
| **1. Database Indexing** | Added GIN Trigram Indexes (`pg_trgm`) | **145.20 ms** | **7.80 sec** | DB execution time dropped from 118ms to **1.28ms** (`Bitmap Index Scan`). |
| **2. Connection Pooling** | Implemented `ThreadedConnectionPool` | **88.40 ms** | **4.90 sec** | Eliminated per-request TCP/TLS handshake overhead to RDS. |
| **3. Production WSGI** | Deployed with Gunicorn (4 Workers) | **63.08 ms** | **3.46 sec** | Enabled true parallel request handling under thread concurrency. |

---

## 🛠️ Code & Implementation Highlights

### 1. Database Indexing with Trigram Extension (`pg_trgm`)
```sql
-- Enable trigram extension for wildcard pattern matching
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Create GIN trigram indexes
CREATE INDEX idx_customers_email_trgm ON customers USING gin (email gin_trgm_ops);
CREATE INDEX idx_customers_bio_trgm ON customers USING gin (bio gin_trgm_ops);

ANALYZE customers;
2. Connection Pooling in Python (psycopg2)
Python
from psycopg2 import pool

# Initialize persistent connection pool
db_pool = pool.ThreadedConnectionPool(
    minconn=5,
    maxconn=20,
    host=DB_HOST,
    database=DB_NAME,
    user=DB_USER,
    password=DB_PASS,
    port=DB_PORT
)

# Borrow connection inside route
conn = db_pool.getconn()
try:
    cursor = conn.cursor()
    # Execute query...
finally:
    db_pool.putconn(conn) # Always return connection to pool
3. Production Deployment with Gunicorn
Bash
gunicorn -w 4 -b 0.0.0.0:5000 app:app
🧠 Lessons Learned & Key Takeaways
Diagnose Before You Scale: CloudWatch metrics and DB execution plans eliminated compute capacity as the root cause, preventing costly and unnecessary infrastructure upgrades.

Standard Indexes Have Limits: Standard B-Tree indexes cannot index leading wildcards (%string). Trigram GIN indexes (pg_trgm) are required for fast pattern matching.

Connection Overhead Adds Up: Database connections are expensive. Reusing persistent connections via application pooling yields massive latency reductions in microservices.

Dev Servers Belong in Dev: Single-threaded development servers choke under concurrent traffic; multi-worker WSGI servers (like Gunicorn) are mandatory for production Python workloads.

📁 Repository Structure
Plaintext
database-bottleneck-analysis-lab/
├── app.py                      # Flask API with psycopg2 connection pooling
├── load_test.py                # Multi-threaded benchmark testing script
├── README.md                   # Project documentation
└── screenshots/                # Evidence of performance milestones
    ├── 01-load-test-baseline.png
    ├── 07-bottleneck-cloudwatch.png
    ├── 08-trgm-indexes-created.png
    ├── 09-explain-analyze-bitmap-scan.png
    ├── 10a-connection-pool-config.png
    ├── 10b-connection-pool-endpoint.png
    ├── 11-optimized-load-test-results.png
    └── 12-optimized-cloudwatch-metrics.png
