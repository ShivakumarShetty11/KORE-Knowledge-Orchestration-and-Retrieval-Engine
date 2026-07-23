# KORE — Knowledge Orchestration and Retrieval Engine

An enterprise AI brain that watches your engineering activity in real-time (GitHub PRs, Jira tickets, Git commits, Slack messages) and answers forensic questions like "who merged what and caused this incident?" — automatically.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     EVENT SOURCES                            │
│  GitHub (PRs) │ Jira (Tickets) │ Git (Commits) │ Slack      │
└────────┬────────────────┬───────────────┬──────────┬─────────┘
         │                │               │          │
         └────────────────┴───────────────┴──────────┘
                          │
                    Kafka Topics
         ┌────────────────┴────────────────┐
         │  raw-git-prs │ raw-jira-tickets │
         │  raw-git-commits │ raw-slack-chats │
         └────────────────┬────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  Kafka Consumer       │
              │  (kafka_consumer.py)  │
              │                       │
              │  • State Management   │
              │  • Selective Indexing │
              │  • Lifecycle Tracking │
              └───┬───────────────┬───┘
                  │               │
         ┌────────▼────┐    ┌────▼──────┐
         │   Neo4j     │    │ ChromaDB  │
         │   (Graph)   │    │ (Vectors) │
         │             │    │           │
         │ • WHO owns? │    │ • Policies│
         │ • Relations │    │ • Events  │
         │ • State     │    │ • Search  │
         └─────────────┘    └───────────┘
                  │               │
                  └───────┬───────┘
                          │
         ┌────────────────▼────────────────┐
         │         AGENT LAYER             │
         │                                 │
         │  Interactive        Autonomous  │
         │  ┌──────────┐      ┌──────────┐│
         │  │Researcher│      │Sentinel  ││
         │  │Writer    │      │Commander ││
         │  └──────────┘      └──────────┘│
         │                                 │
         │  Tools (tools.py):              │
         │  • Expert Finder (WHO)          │
         │  • Document Search (WHAT)       │
         │  • Recent Activity (WHEN)       │
         │  • State Checkers               │
         └─────────────────────────────────┘
                          │
                          ▼
              ┌───────────────────┐
              │   Streamlit UI    │
              │   (app.py)        │
              └───────────────────┘
```

---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running
- Python 3.10+
- A **Google API key** (Gemini LLM) — get one free at https://aistudio.google.com
- A **Cohere API key** (embeddings) — get one free at https://dashboard.cohere.com

---

## How to Run

### Step 1 — Create your `.env` file

Create a `.env` file in the project root with the following content:

```
GOOGLE_API_KEY=your_google_api_key_here
COHERE_API_KEY=your_cohere_api_key_here
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=password
CHROMA_HOST=localhost
SIMULATION_FILE=src/simulation/events.json
```

---

### Step 2 — Start the infrastructure

```bash
docker compose up -d
```

This starts Kafka, Neo4j, and ChromaDB in the background. Wait ~30 seconds, then verify:

```bash
docker compose ps
```

All three services should show `running`.

---

### Step 3 — Set up Python environment

```bash
python -m venv .venv

# Windows
.venv\Scripts\activate

# Mac/Linux
source .venv/bin/activate

pip install -r requirements.txt
```

---

### Step 4 — Initialize Kafka topics

```bash
python src/ingestion/init_kafka.py
```

Creates the 7 Kafka topics the system needs (4 raw data + 3 agent coordination).

---

### Step 5 — Start the Kafka consumer (Terminal 1)

Open a new terminal, activate the venv, then:

```bash
python src/ingestion/kafka_consumer.py
```

Leave this running — it reads incoming events and stores them in Neo4j and ChromaDB.

---

### Step 6 — Run the simulation to load test data (Terminal 2)

Open a new terminal, activate the venv, then:

```bash
python src/simulation/scenarios.py
```

This sends sample events from `src/simulation/events.json` into Kafka. You should see the consumer (Terminal 1) start processing them.

---

### Step 7 — Start the AI brain (Terminal 3)

Open a new terminal, activate the venv, then:

```bash
python src/brain/main_crew.py
```

This starts the interactive AI agent that listens for questions from the UI.

---

### Step 8 — Launch the UI (Terminal 4)

```bash
streamlit run src/ui/app.py
```

Opens the dashboard at **http://localhost:8501**

---

## Dashboards & Service URLs

| Service | URL | Credentials |
|---------|-----|-------------|
| KORE UI | http://localhost:8501 | — |
| Neo4j Browser | http://localhost:7474 | `neo4j` / `password` |
| Kafka UI | http://localhost:8080 | — |
| ChromaDB | http://localhost:8000 | — |

---

## Notes

- You will have **4 terminals** running simultaneously: consumer, simulation, brain, UI.
- Make sure any Jira ticket is related to some repo/PR and vice versa — the graph relationships depend on this.
- The autonomous Policy Sentinel runs automatically inside `main_crew.py` and will flag hardcoded secrets, Friday deploys, and other policy violations without any prompting.