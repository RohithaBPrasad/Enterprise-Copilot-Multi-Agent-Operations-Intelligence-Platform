# Enterprise Copilot: Multi-Agent Operations Intelligence Platform

### Autonomous Enterprise Intelligence & Operations Agent

A collaborative multi-agent AI platform that studies enterprise data, spots problems in day-to-day operations, recommends what to do about them, and executes the workflows that get approved, always routing risky decisions through a human reviewer first.
## Pipeline at a Glance

Each request travels through the following chain of components:

```
Supervisor Agent
   -> Data Agent
   -> SQL Agent
   -> ML Agent
   -> RAG Agent
   -> Decision Agent
   -> Risk Engine
   -> Human Approval
   -> Action
```

**Built with:** FastAPI · Anthropic Claude (LLM / agentic AI) · multi-agent orchestration · RAG · scikit-learn (machine learning) · SQLite (SQL) · ChromaDB (vector database)

**Business areas supported:** Production, Inventory, Quality, Operations, Procurement, HR, Sales, Finance, Projects, and Reporting.

---

## 1. Meet the Agents

| Agent | Source file | What it is responsible for |
|---|---|---|
| Supervisor | `app/agents/supervisor_agent.py` | Coordinates the end-to-end run, stores it as a `workflow`, and wraps things up once a person has made a decision. |
| Data Agent | `app/agents/data_agent.py` | Loads the data for the chosen domain from SQLite and produces summary statistics. |
| SQL Agent | `app/agents/sql_agent.py` | Converts the operator's question into a **read-only** SQL query, which the LLM writes, the system checks, and only then runs. |
| ML Agent | `app/agents/ml_agent.py` | Detects anomalies with IsolationForest and projects a linear trend and forecast for the domain's main metric. |
| RAG Agent | `app/agents/rag_agent.py` | Looks up matching company SOPs and policies in the ChromaDB vector store. |
| Decision Agent | `app/agents/decision_agent.py` | Has the LLM combine the data, SQL, ML, and RAG findings into a root-cause explanation and a ranked list of recommendations. |
| Risk Engine | `app/agents/risk_engine.py` | Scores every recommendation using transparent, rule-based logic and determines whether a person must approve it. |
| Approval Manager | `app/approval/approval_manager.py` | Stores approvals that are waiting; once someone decides, it kicks off execution and updates the workflow. |
| Action Agent | `app/agents/action_agent.py` | Carries out recommendations that were approved or automatically cleared (integrations are simulated, as described in section 7). |

The platform needs **no setup to try out and works fully offline**. When `ANTHROPIC_API_KEY` is missing, any LLM call is answered by a clearly labeled, deterministic placeholder, so the whole pipeline, interface, and approval flow can be explored straight away. Adding the key switches the SQL and Decision agents over to genuine LLM reasoning.

## 2. Repository Layout

```
enterprise-agent/
├── app/
│   ├── main.py                 # FastAPI app + all routes
│   ├── config.py                # env-driven settings
│   ├── database.py              # SQLite schema + synthetic data seeding
│   ├── models.py                 # Pydantic request/response schemas
│   ├── llm_client.py              # Anthropic wrapper + offline fallback
│   ├── vector_store.py            # ChromaDB wrapper (RAG)
│   ├── agents/
│   │   ├── supervisor_agent.py
│   │   ├── data_agent.py
│   │   ├── sql_agent.py
│   │   ├── ml_agent.py
│   │   ├── rag_agent.py
│   │   ├── decision_agent.py
│   │   ├── risk_engine.py
│   │   └── action_agent.py
│   ├── approval/approval_manager.py
│   └── knowledge_base/sample_docs.py   # seed SOP/policy documents for RAG
├── static/index.html            # single-page operations console 
├── scripts/seed_data.py         # standalone DB/vector-store seeding
├── tests/                       # pytest suite-runs fully offline
├── requirements.txt
└── .env
```

## 3. Getting It Running (VS Code)

1. Launch VS Code and open the project folder with `File > Open Folder…`.
2. Make a virtual environment and switch it on:

   ```bash
   python -m venv .venv
   # macOS/Linux
   source .venv/bin/activate
   # Windows
   .venv\Scripts\activate
   ```

3. Install the required packages:

   ```bash
   pip install -r requirements.txt
   ```

4. Duplicate the sample environment file and, if you like, enter your Anthropic API key:

   ```bash
   cp .env.example .env
   # then edit .env and set ANTHROPIC_API_KEY=sk-ant-...
   ```

   You can create a key at https://console.anthropic.com/. The key is not mandatory: leave it out and the application works in full, with the SQL and Decision agents relying on offline placeholders.

5. Launch the server:

   ```bash
   uvicorn app.main:app --reload
   ```

6. Visit the console at **http://localhost:8000/**. This dashboard lets you choose a domain, type in a question, follow each stage of the pipeline as it executes, and approve or reject any recommendations that were flagged.

   Swagger-based API documentation is available at **http://localhost:8000/docs**.

On the first launch, the app builds and fills the database (`data/enterprise.db`) and the vector store (`data/chroma/`) by itself, so nothing needs to be prepared by hand. If you ever want to wipe them and start over with new synthetic data, run:

```bash
python scripts/seed_data.py --force
```

## 4. A Quick Walkthrough

Open the console, choose **Production**, and enter this question:

> Why is Line-A's defect rate rising and what should we do about it?

The seeded sample data is built so that Line-A's defect and downtime rates creep upward across roughly the past 6 days, and its complaint count jumps sharply. The ML Agent therefore marks it as anomalous, and, when a real API key is configured, the Decision Agent follows the problem back to its cause using the SOP that was retrieved. Two more ready-made scenarios are worth a look: ask **Inventory** "which SKUs need reordering?" or ask **Procurement** "is any supplier's on-time delivery degrading?"

Recommendations carrying high risk, such as holding a shipment or changing suppliers, are parked under **Pending approvals** until you either approve or reject them.

## 5. API Endpoints

| Method | Path | Description |
|---|---|---|
| POST | `/api/analyze` | Executes the complete pipeline, with a body of `{ "query": "...", "domain": "production" }` |
| GET | `/api/workflows` | Returns the list of earlier workflow runs |
| GET | `/api/workflows/{id}` | Returns the detailed, step-by-step trace of a single run |
| GET | `/api/approvals?status=pending` | Returns pending approvals, or all of them |
| POST | `/api/approvals/{id}/decide` | Records a decision, with a body of `{ "approved": true, "decided_by": "...", "comment": "..." }` |
| GET | `/api/domains/{domain}/data` | Returns the raw data the Data Agent collects for a domain, useful for debugging |
| GET | `/api/actions` | Returns a log of all actions that have been executed |
| GET | `/api/health` | Reports service status and whether an LLM key is set |

## 6. How Risk Is Judged and Approvals Are Triggered

The `RiskEngine` deliberately uses fixed rules instead of an LLM so that every score can be audited. It gives each recommendation a number from 0 to 100, drawing on these inputs: how critical the domain is, the anomaly rate found by the ML model, how volatile the trend is, the priority the Decision Agent attached to the recommendation, high-impact keywords (for example "hold shipment", "layoff", or "recall"), and certain high-risk `action_type` values.

A recommendation is not executed automatically if its score reaches `RISK_APPROVAL_THRESHOLD` (set to `55` by default and adjustable in `.env`). The same applies whenever it contains a high-impact keyword or uses a high-risk action type, whatever its score. In those cases a human has to approve it first.

## 7. Connecting to Your Own Systems

The handlers in `ActionAgent` (`app/agents/action_agent.py`) are simulations by design. Each records a structured result and marks the exact point where a genuine integration would be attached:

```python
def _handle_create_reorder(rec: dict, domain: str) -> dict:
    # Replace with: your ERP/procurement API call
    return {"system": "ERP/Procurement", "operation": "purchase_order.create", ...}
```

To use a managed vector database such as Pinecone or Weaviate, replace `app/vector_store.py` while keeping its `query()` and `seed_if_empty()` interface unchanged. To point at your own data warehouse, replace the SQLite calls in `app/database.py`; the Data and SQL agents rely only on `get_conn()` and `DOMAIN_TABLE_MAP`.

## 8. Running the Tests

```bash
pytest -v
```

The suite needs no network access: it runs against a throwaway SQLite database and vector store (configured in `tests/conftest.py`). It exercises every agent on its own, the full pipeline driven by the Supervisor, the approval cycle, and the HTTP API.
