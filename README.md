# 🧭 ROTA — AI-Powered End-to-End Travel Planning

<!-- Banner: assets/ROTA_Banner.png -->

ROTA is a multi-tool AI travel assistant that guides users from destination discovery to budget feasibility, combining a curated knowledge base, live data, an independent recommendation engine, and a persistent memory of who you are.

---

## About the Project

ROTA was developed as the second case study of the CBOT internship program. Where PEGA (the first case study) answered questions from a single, closed knowledge domain, ROTA was scoped as an open-domain assistant — any user could ask about virtually any destination in the world. This shift became the project's central engineering problem: how to stay honest and grounded when the space of possible questions is effectively unbounded.

Beyond the base requirements, three components were added on the developer's own initiative: a persistent, cross-session user profile; an independent Python microservice for explainable, weighted destination scoring; and a live integration with PEGA's own flight-route knowledge base, letting the two case studies share infrastructure rather than exist in isolation.

---

## ✨ Key Features

- **Retrieval-Augmented Generation:** 225 vectorized chunks across 50 curated destinations — destination profile, attractions & local cuisine, accommodation & transport, and visa requirements.
- **Transparent Knowledge Boundary:** Questions outside the curated 50 are answered from general knowledge, but only with an explicit, enforced disclosure — never silently.
- **Four Live Data Tools:** Real-time weather, timezone, currency, and country-information lookups, each chosen for zero-authentication reliability after repeated failures with heavier third-party APIs.
- **Protocol-Based Guardrails:** The system prompt classifies each message into an intent category and applies a dedicated protocol, replacing an earlier flat rule list that was inconsistently followed.
- **Honest Budget Feasibility Engine:** Clarifies missing trip details, converts the user's budget at a live exchange rate, and cross-checks its own arithmetic for consistency.
- **Persistent Cross-Session Profile:** Remembers a user's name and preferences across entirely new conversations, independent of per-session chat memory.
- **Independent Recommendation Microservice:** A standalone Python/FastAPI service scores destinations against budget, interest, and season — explainable, not just a match list.
- **Cross-Project Integration:** Queries PEGA's flight-network knowledge base as a second tool, proactively noting when Pegasus Airlines serves a recommended route.

---

## 💬 What the Agent Can Do

- **Recommend by Preference:** "Ekonomik bütçeli, doğa ve macera seven biriyim, Eylül'de gitmek istiyorum" returns a ranked, reasoned shortlist — not just a filtered list.
- **Build a Day-by-Day Itinerary:** Generates attraction- and cuisine-aware daily plans for any of the 50 curated destinations.
- **Check Budget Feasibility:** Asks clarifying questions, then gives a currency-converted, arithmetic-consistent yes/no on whether a stated budget is realistic.
- **Remember Returning Users:** A user who identifies themselves is recognized in a brand-new session days later, preferences intact.
- **Cross-Check Flights:** Silently checks whether Pegasus Airlines flies a recommended route and mentions it only when true.

---

## 🔬 Engineering Deep Dive

Problems discovered through systematic testing and fixed on the developer's own initiative, beyond the case brief's explicit requirements.

### Unbounded Scope vs. Reliable Coverage
The case brief placed no limit on which destinations ROTA should cover, so the first instinct was to fetch destination and points-of-interest data live, for any city, via a real-time API. More than five different public APIs were evaluated for this over the course of a day; each failed for a distinct, genuine reason — incomplete account verification, a deprecated API version, unreliable rate-limited responses across multiple mirror servers, and an inconsistent response schema. The scope was deliberately narrowed to a curated set of the 50 most commonly requested destinations, paired with an explicit-disclosure fallback — trading unlimited coverage for guaranteed correctness within a known boundary.

### Platform Selection: n8n over Flowise / Langflow
Two alternative platforms were evaluated before reverting to the n8n foundation already proven in PEGA. Flowise was rejected after a reproducible network failure in its token-counting dependency caused multi-minute response delays; Langflow was rejected after its cloud-Redis memory integration failed repeatedly due to an immature SSL layer. Both failures were root-caused before the decision was made, turning the platform choice into evidence rather than assumption.

### Problems Found & Fixed

| Problem | Root Cause | Fix |
|---|---|---|
| Rare destinations vanishing from search results | A JSON loader field split structured metadata into meaningless word-level fragments during embedding | Data simplified to clean `pageContent`-only records before vectorization |
| Duplicate destinations returned in one query | Vector store re-populated without clearing prior data | Enforced clear-before-reload discipline on ingestion |
| Budget answers containing fabricated cost figures | An early version hard-coded per-destination daily costs with no real source | Redesigned around a live exchange-rate lookup, with estimates explicitly labeled non-authoritative |
| Currency conversion giving two different results in one conversation | Model arithmetic error, not a real rate change | Self-consistency check forcing recalculation on material divergence |
| Case-sensitive user profile lookups (`"Furkan"` vs `"furkan"`) | Redis key stored exactly as typed | Keys normalized to lowercase/trimmed on every read and write |
| PEGA flight-route lookups always returning empty | n8n's in-memory vector store resets on container restart | Re-ran PEGA's ingestion workflow; verified both workflows share an identical memory key |
| Rules added as a standalone "side note" were inconsistently followed | Generic instructions compete poorly against the model's default response pattern | Rewritten as a mandatory step embedded inside the relevant protocol |

### Independent Recommendation Microservice
A FastAPI service, deliberately built and hosted outside n8n to establish a real microservice boundary, scores candidate destinations by weighing budget match, interest overlap, and seasonal fit. Before writing any code, the Docker network path from the n8n container to the host machine was verified with a deliberate connectivity test (`host.docker.internal`); a "connection refused" response confirmed correct DNS resolution ahead of implementation. Initial scoring weights over-favored budget match and were rebalanced after testing against multiple example user profiles.

### Validation
The full system — RAG retrieval, all four live tools, the budget engine, the persistent profile, the PEGA integration, and the recommendation microservice — was validated against 20+ manually run scenarios, including vague requests, destinations outside the curated dataset, and repeated-recommendation avoidance.

---

## 🛠️ Technologies Used

| Technology | Purpose |
|---|---|
| **n8n** | Workflow orchestration for ingestion and agent conversation logic |
| **Docker** | Containerized environment for n8n, Redis, and the Python microservice |
| **Redis** | Session-based conversation memory *and* a separate persistent user-profile store |
| **Simple Vector Store** | Native n8n vector index (ROTA's own base and, separately, PEGA's shared flight-network store) |
| **OpenAI Embeddings** | Vectorization of knowledge base chunks |
| **gpt-4o-mini** | Response generation for the conversational agent |
| **Python / FastAPI** | Independent microservice for weighted, explainable destination scoring |
| **OpenWeatherMap API** | Live weather data |
| **timeapi.io** | Live timezone / time-difference data |
| **open.er-api.com** | Live currency exchange rates |
| **REST Countries API** | Country metadata (capital, currency, language) |

---

## 📂 Project Structure

```
rota/
├── workflows/
│   ├── Travel Assistant - Load Knowledge Base.json   # 225-chunk RAG ingestion
│   └── Travel Assistant.json                          # Live agent conversation workflow
├── services/
│   └── smart_recommender/
│       └── main.py                                     # FastAPI scoring microservice
├── assets/
│   ├── Load Knowledge Base.png
│   └── Travel Assistant.png
└── README.md
```

---

## 🚀 Installation & Setup

**1. Start Redis via Docker**
```bash
docker-compose up -d
```

**2. Start the Recommendation Microservice**
```bash
cd services/smart_recommender
pip install fastapi uvicorn
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

**3. Import the Workflows**

In your n8n instance, import in this order:
1. `Travel Assistant - Load Knowledge Base.json`
2. `Travel Assistant.json`

**4. Configure Credentials**

Set OpenAI, Redis, and OpenWeatherMap credentials in their nodes. Confirm the microservice tool's URL points to `http://host.docker.internal:8000/recommend`.

**5. Run the Ingestion Workflow**

Execute **Load Knowledge Base** once to populate the vector store with all 225 chunks.

**6. Activate the Agent**

Activate **Travel Assistant** to start handling live conversations.

---

## ✒️ Development Team
**Nidanur Sığırta** — AI Workflow Designer & RAG Architecture

## 🛡️ License
© 2026 ROTA. All rights reserved.

*Planning Real Journeys on Grounded Information*
