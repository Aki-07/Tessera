# Tessera Legendary Plan (Implementation Blueprint)

This document captures the end-to-end plan to evolve Tessera into the “Autonomic LLM Defense Grid” with session-based, multi-agent testing, reproducible evidence, calibrated judge ensembles, RAG defenses, and attack seasons. It is designed so every subsystem has clear owners, artifacts, and acceptance criteria.

## Objectives (What Good Looks Like)
- High-precision breach detection via calibrated ensemble judges (rules + reward models + anomaly as weak signal) with reproducible evidence packs.
- Deterministic replays and counterfactuals for both battles and multi-turn sessions, even with non-deterministic RAG.
- Session-based, multi-agent evaluations (planner/executor/sentinel/critic graphs) with attack injections and drift detection.
- Attack seasons: rotating adversarial prompt library with budgeted online sampling and offline refresh.
- RAG Lab: poisoning/watermark checks, retrieval anomaly scoring, and pre-generation blocking.
- CI/CD gate + attestations; exportable audit reports with signed hashes.

## Architecture (Target Stack)
- **Orchestrator**: FastAPI + LangGraph for agent graph execution; Postgres for runs/config; Redis/KeyDB for queues.
- **Agent Graph Store**: Neo4j for versioned agent graphs; optional fallback to Postgres JSON if Neo4j unavailable.
- **Evidence/Replay**: S3-compatible storage for snapshots; hashes stored in Postgres. Local dev uses /data.
- **Judges**: Rule engine (OPA/rego or inline), safety reward models (served via vLLM/Triton), anomaly detector (embedding OOD with conformal bounds).
- **Vector Store**: Weaviate/Qdrant for anomaly reference corpora; RAG Lab uses same store to hash and verify ingested docs.
- **Event Bus**: NATS/Kafka for streaming judge outputs and replays (optional; start with in-process events).
- **UI**: Next.js + Tailwind + Framer Motion; graph viz via Cytoscape/Visx; timeline via Virtuoso.

## Data Model (Incremental)
- **Postgres tables (migrations planned)**:
  - `runs` (id, type=battle|session, status, seed, graph_version, snapshot_ref, metrics, created_at, finished_at).
  - `judgments` (id, run_id, round_idx, verdict, confidence, scores_json, reasons_json, evidence_ref, created_at).
  - `retrieval_events` (id, run_id, round_idx, query, emb_model, reranker_model, k, results_json, content_hashes, created_at).
  - `attack_library` (id, variant, model_family, class, payload, efficacy, freshness, metadata).
  - `snapshots` (id, run_id, payload_hash, location, created_at, signature).
- **Neo4j (or JSON fallback)**:
  - Nodes: Agent {id, role, prompt_version, tools_allowed, guardrails, embeddings?}
  - Edges: {from, to, condition, priority}
  - Graph metadata: {graph_id, tenant, version, purpose, created_at}

## Key Tracks and Deliverables

### 1) Judge Ensemble (Backend)
- **Features ingested**: rule hits (one-hot + severity), RM scores, anomaly distance, tool/policy violations, RAG flags (watermark/poison), latency/context.
- **Aggregation**: calibrated logistic/GBM with temperature scaling; hard-deny/allow overrides; “uncertain” when confidence < τ.
- **APIs**:
  - `POST /judge/evaluate` → {verdict, confidence, scores{rules, rm, anomaly}, reasons[], evidence_id}
  - `GET /judge/evidence/{id}` → serialized inputs/outputs, hashes.
- **Acceptance**: ECE + Brier on held-out; FN/FP tracked per attack class; override logic unit-tested.

### 2) Evidence Packs & Reproducibility
- **Snapshot contents**: seed; prompts; model ids/versions; tool calls; LangGraph graph version; retrieval requests (query, emb model/version hash, reranker, k, filters); retrieved items (ids + full text + rank + score + hash); timestamps; judge outputs; configs.
- **Replay rules**: on replay, attempt re-fetch by ID; if drift, fall back to stored content; mark mode in evidence. Refuse “exact replay” if embedding/reranker versions changed unless forced snapshot mode.
- **APIs**:
  - `POST /replay/snapshot/{run_id}` → snapshot_id, hash, signature.
  - `POST /replay/run` with snapshot_id + perturbations → new run_id.
  - `GET /replay/diff/{base}/{cf}` → structured diff.
- **Acceptance**: deterministic replays for fixed seeds; evidence hash stable; drift surfaced in metadata.

### 3) Session-Based Multi-Agent Runner
- **Session DSL**: turns (actor/message), attack injections (turn-based), branches (conditions like judge.uncertain), latency budgets.
- **Agent graph execution**: LangGraph nodes for planner/executor/sentinel/critic; shared memory with chat history + retrieved docs; policy-as-code gate before tools.
- **APIs**:
  - `POST /session/start` (script + graph_id + seed) → session_id.
  - `GET /session/status/{id}`, `POST /session/stop/{id}`, `POST /session/replay`, `GET /session/evidence/{id}`.
- **Acceptance**: end-to-end run with stored evidence; judge per turn + session summary; branches honored.

### 4) RAG Lab & Retrieval Safety
- **Ingestion**: watermark/poison scan; hash chunks; store doc_id + hash + metadata.
- **Pre-gen checks**: block if doc watermark fails or poison heuristics trigger; log flag in evidence.
- **Retrieval anomaly**: reference corpus OOD score with conformal bound; used as weak signal.
- **APIs**:
  - `POST /rag/ingest`, `POST /rag/scan`, `POST /rag/report`.
- **Acceptance**: ingestion emits hashes; replay can reconstruct context from stored chunks; blocking toggles configurable.

### 5) Attack Seasons Library
- **Offline generation**: nightly/weekly AutoDAN-style suffix search on beefy workers; store top-N per model family/class with efficacy scores and freshness.
- **Online selection**: bandit sampler with per-run budget (e.g., 3–5 suffixes); rotate weekly; retire low-efficacy variants.
- **APIs**:
  - `GET /attacks/sample?model_family=X&class=jailbreak&n=5`
- **Acceptance**: cache hit rate high; online loop never performs expensive search; rotation policy documented.

### 6) UI (Next.js)
- **Views**: Battle/Session Theater timeline, Why tab (rules/RM/anomaly/reasons), Context (tools/docs), Fix (prompt/policy diffs + rerun), Replays (base vs counterfactual), Graph viz (agent graph), RAG Lab dashboard.
- **States**: handle uncertain verdicts; display replay mode (live vs snapshot); show drift warnings for RAG.
- **Acceptance**: API data wired; evidence download; confidence bars; attack season picker.

### 7) CI/CD & Compliance
- **CLI**: `tessera check` → JUnit/Allure + signed attestation; fail on breach_rate > threshold.
- **Exports**: downloadable evidence bundle (JSON + hash + signature); badge endpoint “Breach rate past 7d.”
- **Acceptance**: sample pipeline in repo; attestation validates hash/signature.

## Execution Phases (Two-Day Spike → Subsequent)
1. **Spike (current window)**: add schemas/migrations; stub judge API; add snapshot/replay scaffolding; add session endpoints (skeleton); add docs (this file).
2. **Judge MVP**: implement rule layer + single safety RM + anomaly (weak); calibration script with seed corpus; persist judgments/evidence.
3. **Session Runner**: LangGraph integration; DSL parser; evidence logging; UI stubs for timeline/Why tab.
4. **RAG Safety**: retrieval logging + hash storage; replay fallback; watermark/poison scan hooks.
5. **Attack Library**: offline job scaffolding; sampling API; budgeted sampler in runner.
6. **UI Wiring**: connect new APIs; confidence/uncertain surfacing; replay diff view.
7. **CI/Attestation**: CLI tool, JUnit emit, evidence signing; sample GH Action.

## Evaluation & Calibration
- **Datasets**: mix of jailbreak corpora (AdvBench, AutoDAN variants), PII/secret leak sims, benign domain dialogs, RAG with benign and poisoned docs.
- **Metrics**: FN/FP per class, ECE/Brier, AUROC for RM; anomaly FPR bounded via conformal; replay determinism rate; coverage by attack class; MTTR to fix (Fix loop).
- **Governance**: auto down-weight anomaly if FPR spikes; weekly calibration run; track overrides fired.

## Calibration Details (Concrete)
- **Ground truth**: seed 5–10k samples per tenant class mix: 40% jailbreak/adv prompts (AdvBench + AutoDAN variants), 30% PII/secret leak sims (templates), 30% benign domain dialogs + tool/RAG calls. Label via strong-model adjudication + human spot-check (at least 10% manually verified).
- **Splits**: 70/15/15 train/val/test per tenant; add a global baseline model; per-attack-class stratification. If tenant sample <1k, fall back to global baseline with light per-tenant recalibration.
- **Training**: shallow logistic/GBM on features; tune on val; evaluate on test; report ECE/Brier/FN/FP per class.
- **Calibration**: temperature scaling (or Platt) on held-out val; run per tenant weekly; per attack class if sample ≥200; otherwise fall back to global scaling. Store calibration params versioned and include in evidence.
- **Runtime**: hard-deny/allow rules override; otherwise apply calibrated model; if confidence < τ (configurable per tenant/class), emit `uncertain`.

## Attack Seasons (Compute/Budget Reality)
- **Offline only for heavy search**: AutoDAN-style suffix search runs nightly/weekly on dedicated workers per top model families (e.g., GPT-4.x, Claude, Llama variants). Budget cap: e.g., 2–4 hours per family, top-N saved with efficacy scores.
- **Caching**: store suffixes with model_family, class, date, efficacy; retire if efficacy drops below threshold in online A/B.
- **Online use**: no in-loop expensive search. Per run/session, sample 3–5 cached attacks (budget) via bandit on cached pool; these are executed, not generated.
- **Freshness**: rotate weekly; trigger targeted refresh when a new defender/policy version lands or efficacy drops.

## Uncertain Verdict Handling
- **Definition**: verdict where calibrated confidence < τ (per tenant/class). Output `uncertain` with reasons and scores.
- **Behavior**: session DSL may branch on `judge.uncertain`; default path: log, tag run for human review, optionally re-run with higher-cost judge (e.g., larger model) or stricter policies. CI gate treats `uncertain` as configurable (fail or warn).
- **UI**: show uncertainty band; provide “escalate/re-run” CTA.

## RAG Replay Semantics
- **Exact replay mode**: only claimed when embedding/reranker versions and corpus IDs match; retrieval re-executed; context rebuilt from live results.
- **Snapshot replay mode**: if drift detected, rebuild context from stored retrieved chunks (content + hashes) to reproduce the exact original conversation; evidence marks this mode.
- **Live efficacy check**: optionally re-run against live retriever after snapshot replay to see if attack still works; differences logged in diff report.
- **Evidence**: include retrieval mode, hashes, version info, and whether live check diverged.

## Integration Points (Operational Semantics)
- **Runner ↔ Judge**: per turn, after agent output assembled and before commit to state, call judge; judge verdict stored; if `breach` and policy says block, halt or branch; if `uncertain`, follow DSL branch or default policy.
- **Policy gate timing**: tool calls pass through policy-as-code **before** execution (prevent unsafe tools); post-call check may also run to inspect outputs.
- **Attack injection**: session DSL injects attacker messages as turns (message-level). Node-level override is optional extension; default is a synthetic actor inserting a turn at specified step.
- **Replay perturbations**: perturbations apply at specified turns; prior history (turns 1–2) stays identical from snapshot; downstream turns re-simulate with the new injected message, preserving seed for determinism unless explicitly changed.

## Risk Mitigation
- **Disagreement in ensemble**: hard-deny/allow overrides; calibrated model for the rest; expose “uncertain” path.
- **RAG nondeterminism**: snapshot content + IDs + model versions; mark replay mode; refuse exact replay on drift unless forced.
- **Attack freshness**: offline refresh + rotation; budgeted online sampling; leaderboards to retire stale variants.
- **Compute**: keep heavy search offline; in-loop bounded attack budget; small RM served via vLLM/Triton.

## Integration Notes for Current Repo
- Keep `/data/battles` for local dev; add Postgres/Neo4j migrations alongside SQLite for compatibility.
- Extend existing battle/session state to include: seeds, retriever metadata, retrieved docs (id+content+hash), judge outputs, evidence ids.
- Add new routes under `services/orchestrator/app/api`: judge, session, replay, rag, attacks.
- Introduce background workers (RQ/Arq/Celery) only where needed (attack refresh, heavy search).
- Preserve capsule interface (`/call_tool`), but allow agent graph nodes to target capsules or hosted models.

## Deliverables Checklist
- [ ] DB migrations (runs, judgments, retrieval_events, attack_library, snapshots)
- [ ] Judge service with calibration script + tests
- [ ] Snapshot/replay APIs + evidence pack format spec implemented
- [ ] Session runner + DSL + endpoints
- [ ] RAG safety hooks + retrieval logging
- [ ] Attack sampling API + offline job scaffold
- [ ] UI pages updated (timeline, Why/Context/Fix/Replays, graph, RAG Lab)
- [ ] CLI `tessera check` + attestation + sample CI config
- [ ] Metrics/observability dashboards (Prometheus/Grafana) wiring for new flows
