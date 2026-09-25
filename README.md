Building an Agentic Fraud Investigation System with TigerGraph Savanna
1. Problem Statement

Fraud detection systems built on flat, tabular models see transactions in isolation — a $9,000 withdrawal in Mumbai and a $9,000 withdrawal in Delhi seven minutes later look like two unrelated rows unless something explicitly joins them. Fraud, in practice, is relational: shared devices, shared IPs, shared merchants, and repeated behavioral patterns across accounts are what actually expose fraud rings and account takeovers. A rules engine or a standalone ML classifier scoring one transaction at a time structurally can't see that.

The HHGOA (TigerGraph Agentic Fraud Investigation) hackathon brief made this concrete: build an AI agent that doesn't just score a transaction, but investigates it — pulls connected entities, checks internal fraud policy, weighs its own confidence, decides whether it has enough evidence to act, and recommends a next-best-action under real approval constraints (some actions can't just be auto-executed by a model). Working dataset: HHGOA_IEEE, adapted from IEEE-CIS/Vesta — roughly 590K card transactions across ~13.5K customers over six months, with risk scores but no ground-truth fraud flag, plus 20 held-out benchmark cases to evaluate against.

2. Solution Overview

The system is a graph-native, multi-phase investigation agent:

TigerGraph Savanna stores customers, accounts, transactions, devices, IPs, and merchants as a connected graph, so "who else used this device" is a graph traversal, not a join across millions of rows.
A 6-phase LangGraph workflow runs each investigation: query the graph → gather evidence → assess confidence → (conditionally) request more evidence → recommend an action → record the case back into the graph.
An MCP tool layer exposes 13 discrete TigerGraph operations (query user, query transaction, check policies, retrieve evidence, create/update case, etc.) as callable tools, so the LLM never talks to the graph directly — it calls named, typed tools.
GraphRAG (FAISS + sentence embeddings) retrieves relevant fraud policies, known fraud-pattern definitions, and similar historical cases as semantic context for the LLM's reasoning.
A deterministic policy engine sits downstream of the LLM's confidence score and enforces who is allowed to approve what — the model recommends, but critical actions still require a human role (ANALYST, SENIOR_ANALYST, or MANAGER).
A React dashboard (cases, investigations, approvals, network visualization, analytics) lets a human see and act on what the agent found.
3. System Architecture
React (Vite) frontend
        │  REST (axios)
        ▼
FastAPI backend (routers: investigations, cases, fraud, health, data)
        │
        ├── LangGraph InvestigationAgent (6-phase workflow)
        │        │
        │        ├── MCP tool layer (13 tools) ──► TigerGraph Savanna (GSQL queries)
        │        ├── GraphRAG retriever (FAISS index) ──► policies / patterns / past cases
        │        └── ChatOpenAI (gpt-4o-mini) ──► reasoning + recommendation text
        │
        └── Policy Engine (deterministic risk/confidence → action + approval-role rules)

The LLM never has write access to the graph or the final say on a BLOCK action — it produces a recommendation and a confidence score; the policy engine (plain Python, no model in the loop) decides whether that action can auto-execute or needs a named approval role. That separation was a deliberate choice: it's the difference between "an LLM decided to block your card" and "an LLM flagged this, and policy required manager sign-off."

4. React Frontend

Built with React 19 + Vite + TypeScript, using reactflow for the graph/network visualization page and recharts for analytics. Pages: LandingPage, DashboardPage, CasesPage / CaseDetailPage, InvestigationsPage / InvestigationPage, ApprovalsPage (where a human with the right role approves or rejects a recommended action), NetworkVisualizationPage (renders the fraud-ring/shared-device graph), TransactionsPage, AnalyticsPage, and ReportsPage. axios talks to the FastAPI backend over REST; there's no websocket layer, so the UI polls/refetches rather than streaming phase-by-phase agent updates live.

5. FastAPI Backend

Routers are split by concern: investigations.py (the 6-phase workflow endpoints), cases.py, fraud.py (transaction-level lookups and an event-ingestion endpoint), data.py, and health.py. Worth being upfront about: the investigations router currently seeds itself with case_storage: Dict[str, Dict[str, Any]] — in-memory storage — and a USE_MOCK = True flag with hardcoded example cases (CASE-2026-001 etc.) as a fallback path, explicitly commented "in production, use database." That's a reasonable hackathon-timeline decision, not a hidden flaw, but it's worth saying in a blog rather than implying a persistent production datastore.

6. Agent Workflow

The core is InvestigationAgent in backend/agents/investigation_agent.py, built as a LangGraph StateGraph over six phases:

Query graph — pulls user profile, transaction details, and merchant profile via MCP tool calls.
Get evidence — calls retrieve_evidence (GraphRAG) and check_policies against the case context, populating matched patterns, related cases, and policy violations.
Assess uncertainty — computes a confidence score from a weighted formula: min(1.0, (evidence_count*0.3 + user_risk*0.4 + indicator_count*0.3) / 10.0). If confidence falls below settings.agent.uncertainty_threshold (default 0.4), the graph branches to phase 4; otherwise it skips straight to phase 5.
Request additional evidence — the LLM drafts a natural-language ask for more evidence (in the current build, the customer response to that ask is simulated: "Customer confirmed legitimate travel", rather than wired to a live customer channel). Loops back to phase 3, capped by an iteration count.
Recommend action — the LLM is prompted to output one of ALLOW / HOLD / CHALLENGE / BLOCK with reasoning; the response is parsed by keyword match ("BLOCK" in response.upper(), etc.) rather than structured output/function-calling.
Record case — writes the recommendation and status back into TigerGraph via update_case_status and add_case_evidence, and stamps total execution time.

Each phase writes to a running case_notes log and an errors list, so a failed phase doesn't crash the run — it degrades gracefully (phase 5, for instance, defaults to the safe "HOLD" action on exception).

7. LLM Integration

gpt-4o-mini via langchain_openai.ChatOpenAI, temperature 0.0, called directly (not through the MCP layer) at two points in the workflow: drafting the evidence request (phase 4) and producing the action recommendation with reasoning (phase 5). The LLM's role is deliberately narrow — it never queries the graph itself; the MCP tool layer does that, and the LLM only ever reasons over structured results it's handed. Prompts are plain f-strings rather than a prompt-management framework, and the action-recommendation parsing is string matching rather than a typed output schema — a straightforward place to harden if you extend this past hackathon scope.

8. TigerGraph Graph Schema

The schema (tigergraph/schema/fraud_graph_schema.gsql) defines 11 vertex types — Customer, Account, Transaction, Device, IPAddress, Merchant, FraudCase, FraudPattern, Evidence, Action, InvestigationLog — connected by edges like Customer -PERFORMS-> Transaction, Customer -USES_DEVICE-> Device, Customer -USES_IP-> IPAddress, Transaction -TRANSACTION_TO-> Merchant, and case-side edges (FraudCase -INVESTIGATES-> Transaction, -HAS_EVIDENCE->, -RECOMMENDS_ACTION->, -MATCHES_PATTERN->). Every edge is created with a REVERSE_EDGE, so traversals work both directions — e.g. "customers who used this device" is just as cheap as "devices this customer used." The graph is the shared-fact layer between the agent and the case-record: investigations aren't just computed and thrown away, they're written back as first-class FraudCase/Evidence/Action vertices.

9. Fraud Investigation Queries

Six GSQL queries do the actual pattern-matching, each purpose-built rather than generic:

find_fraud_network — starting from one customer, walks out through shared devices and shared IPs to find other connected customers, then scores each by a simple connection-count "centrality" (CRITICAL above 50 connections, down to LOW).
detect_transaction_velocity — pulls all transactions for a customer within a time window and derives an anomaly score from transaction count, average/max amount, and merchant diversity (e.g. +0.3 score if tx_count > 10, +0.2 if max_amount > 10000).
find_shared_devices / find_shared_ips — identify other accounts touching the same device or IP as the subject account.
find_similar_historical_cases — filters closed FraudCase vertices by matching fraud_pattern_detected and risk range, ranks by a similarity score, and surfaces what action was taken on similar past cases.
get_investigation_context — the aggregate query the agent calls to assemble everything above into one investigation payload.

These are hand-written heuristic thresholds (> 10 transactions, > $500 avg, > 5 merchants), not learned from the data — a fair thing to name plainly rather than imply they're statistically calibrated.

10. Evidence Collection

Evidence retrieval (backend/graphrag/retriever.py) combines two retrieval paths: a FAISS vector index (sentence-transformers/all-MiniLM-L6-v2, 384-dim) over policy documents, fraud-pattern definitions, and historical case summaries, semantically searched using the case's risk indicators as the query; and direct graph neighbor lookups for the subject customer. The retriever assembles a structured evidence object — policies, patterns, related_cases, graph_neighbors — which the agent both hands to the LLM as context and stores as Evidence vertices linked back to the FraudCase in TigerGraph, so every case carries a durable, queryable trail of what evidence supported it.

11. Next-Best-Action Generation

This is the one part of the pipeline that's not left to the LLM. backend/policies/policy_engine.py implements a deterministic FraudPolicy rule table keyed on (risk_score, confidence_score):

Condition	Action	Approval required
risk ≥ 0.95, confidence ≥ 0.85	BLOCK_ACCOUNT	MANAGER
risk ≥ 0.85, confidence ≥ 0.75	BLOCK_ACCOUNT	ANALYST
risk ≥ 0.85, 0.5 ≤ confidence < 0.75	REQUEST_AUTH	none
risk ≥ 0.85, confidence < 0.5	HOLD_TRANSACTION	ANALYST
0.65 ≤ risk < 0.85	HOLD_TRANSACTION	ANALYST
risk < 0.65	ALLOW	none

The LLM's job ends at producing a risk/confidence read and a rationale; this table is what actually decides whether the system can auto-act or must stop and wait for a named human role. That was the intended answer to the brief's "policy and approval constraints" requirement — next-best-action isn't a model output, it's a governed decision the model feeds into.

12. Results / Demo

Across the 20 HHGOA benchmark cases, the project's own tracking (QUICK_ANSWERS.txt) reports ~85% accuracy against the labeled outcomes, sub-500ms query latency from TigerGraph Savanna, and an estimated ₹980,000+ in fraud exposure caught across the case set — e.g. case HHG-001 was flagged CRITICAL at 92% confidence for an "impossible travel" pattern (Mumbai → Delhi ATM withdrawals 7 minutes apart), tracing back to 8 connected entities and 12 pattern matches in the graph. (Worth re-verifying these three headline numbers against your actual benchmark run output before publishing — they should be defensible if a judge asks how they were computed.)

13. Challenges and Lessons Learned
LLM output as control flow is brittle. Parsing "BLOCK" in response.upper() to route a workflow works in a demo but is one prompt drift away from silently misrouting. A structured-output/function-calling contract for phase 5 would be the first thing I'd harden.
Separating recommendation from execution mattered more than expected. Keeping the policy engine deterministic and outside the LLM's control turned out to be the cleanest way to satisfy "policy and approval constraints" — it also makes the system's decisions auditable independent of whatever the model happened to say.
Mocked pieces are still mocked. The customer-evidence-response in phase 4 and the in-memory case store with a USE_MOCK fallback are honest gaps, not implementation details to gloss over — worth naming them as "next steps" rather than as a finished production pipeline.
Graph-native fraud queries paid off. Fraud-ring and shared-device detection that would be a multi-way SQL join became direct traversals (find_fraud_network, find_shared_devices) — this is the strongest argument for TigerGraph over a relational store for this problem specifically.
Thresholds are still heuristics. The velocity/confidence formulas are hand-tuned constants, not fit to the data — fine for a hackathon under time pressure, but the obvious next step is calibrating them against the labeled portion of the dataset instead of guessing.
