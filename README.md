# Data Engineering: Internals and Architecture

A comprehensive, Distinguished Engineer–level guide to Data Engineering. Covers both the bleeding-edge architecture of 2026 (Agentic AI, HBM memory walls, Shared-Storage) and the foundational technologies of the 2010s/2020s (Kafka, Spark, Airflow, Flink, Data Governance).

## 📖 Access the Book
Read the full book online: [https://atulranaa.github.io/data-engineering/](https://atulranaa.github.io/data-engineering/) *(Replace with actual URL once deployed)*

## 🧠 Core Topics
- **Part I: Fundamentals of 2026** - Physics of AI Agents, Vector DBs, Edge Computing.
- **Part II: Distributed Evolution** - Shift to Shared-Storage engines, Consensus algorithms (Paxos/Raft).
- **Part III: Agentic AI** - Infrastructure for Action Engines and Multi-Agent Systems.
- **Part IV: Deep Internals & Code** - Kafka zero-copy, Spark Catalyst AST, Lakehouse commits (Delta/Iceberg).
- **Part V: Unpacking the Titans** - Deconstructing Nvidia (NVLink), OpenAI, and Google (TPUs).
- **Part VI: Basic Engineering** - Pipelines, Governance, Observability, Formats, and Architecture.
- **Part VII: Basic Deep Dives** - Hands-on teardowns of Kafka, Spark, Flink, Airflow, Cassandra, and S3.

## 🛠️ Running Locally

The documentation is built using [MkDocs](https://www.mkdocs.org/) and [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/).

1. **Clone the repository:**
   ```bash
   git clone https://github.com/atulRanaa/data-engineering.git
   cd data-engineering
   ```

2. **Set up a virtual environment (Recommended):**
   ```bash
   python -m venv .venv
   source .venv/bin/activate
   ```

3. **Install Dependencies:**
   Ensure you have all the required plugins installed.
   ```bash
   pip install mkdocs-material pymdown-extensions
   ```
   *(If a `requirements.txt` is present, run `pip install -r requirements.txt`)*

4. **Serve locally:**
   ```bash
   mkdocs serve
   ```
   Open `http://127.0.0.1:8000/` in your browser.

## 🚀 Deployment

This repository uses GitHub Actions to automatically deploy changes from the `main` branch to GitHub Pages. See `.github/workflows/deploy.yml` for the CI/CD pipeline.
