# Cosmos DB Partition-Key Design POC

> A Forward Deployed Engineering (FDE) proof of concept. This repository contains both the **pre-transformation** (before) and **post-transformation** (after) states of the solution, so the impact of the work can be seen side by side.

---

## 1. Overview

| Item | Details |
|------|---------|
| **POC Name** | Cosmos DB Partition-Key Design: Generate and Govern |
| **Scenario / Customer** | Insurance motor-claims assistant (illustrative scenario) storing case records in Azure Cosmos DB (Core/SQL API) |
| **Owner** | Aditya Jagtap |
| **Status** | Completed |
| **Program** | Birlasoft FORGE FDE Academy, Sprint 1, Day 18 |

**One-line summary:** Choosing an immutable Cosmos DB partition key from measured Request Unit (RU) cost, partition skew and change-feed behaviour on a live account, instead of from intuition.

---

## 2. Problem Statement

A claims-processing service needs a persistence layer. Cosmos DB requires exactly one partition key per container, and it cannot be changed later without migrating all data. Three access patterns compete for that single choice, and generic database advice (use `/id`, add an index, provision more throughput) is wrong here in ways that cost real money.

- Three access patterns, one key: `get by case_id` (40,000/month), `list by applicant_id` (3,000/month), `filter by status` (6,000/month).
- A skewed key creates a hot partition that returns 429 errors while the container dashboard still shows spare capacity.
- The Cosmos change feed is easy to mistake for an audit trail, but it silently drops intermediate states.

---

## 3. Pre-Transformation (Before)

The starting point is the reference material and an unexecuted notebook: a design playbook, an RU-economics walkthrough, a checklist template, a video-insights note, the course deck and the exercise brief. There is no measured evidence for any design choice.

| Aspect | Current State |
|--------|---------------|
| Process | Partition key would be picked by intuition or generic relational advice |
| Model / Tooling | Reference documents and a notebook template only; no live account run |
| Cost | Unknown; no RU measurements |
| Latency / Performance | Unknown; no per-pattern RU baseline |
| Accuracy / Quality | Hot-partition risk and change-feed gaps not quantified |
| Pain points | Immutable key chosen without evidence; "provision more RU" proposed as a fix; change feed assumed to be an audit log |

**Location in repo:** [Pre-Transformation/](Pre-Transformation/)

---

## 4. Post-Transformation (After)

The notebook was run live against a real Cosmos DB account. It created and tore down 7 containers, measured RU for every pattern against every candidate key, and exported the result to `cosmos_design.json`.

| Aspect | New State |
|--------|-----------|
| Process | Same 400 synthetic documents loaded into three containers (partitioned by `case_id`, `applicant_id`, `status`); 3 query patterns x 3 containers, 12 samples each, median RU recorded |
| Model / Tooling | Python, `azure-cosmos` SDK, `python-dotenv`; `case_id` chosen as the partition key; serverless throughput recommended for the POC |
| Cost | 586,125 RU/month with `case_id` (about $0.16 at $0.28 per million RU) versus 655,325 RU/month for either rejected key |
| Latency / Performance | `get by case_id` is a 1.14 RU point read; `status` key would drive about 2,900 RU/s onto one partition against a roughly 1,000 RU/s ceiling |
| Accuracy / Quality | Skew measured, not assumed: `case_id` 400 logical partitions (largest 0.3%), `status` 5 partitions (largest 72.5%) |
| Benefits | Evidence-backed key, a named rejection reason for each alternative, a measured audit-trail gap with a costed mitigation, and a cleanup that leaves nothing billing |

**Location in repo:** [Post-Transformation/](Post-Transformation/)

---

## 5. Before vs. After

The comparison is between the naive alternatives that would have been chosen without measurement and the measured, chosen design.

| Metric | Pre-Transformation | Post-Transformation | Change |
|--------|--------------------|---------------------|--------|
| RU per `get by case_id` | 2.87 (query on `applicant_id` / `status` key) | 1.14 (point read on `case_id` key) | -60% |
| Monthly RU at projected volumes | 655,325 | 586,125 | -10.6% |
| Largest partition share | 72.5% (`status` key) | 0.3% (`case_id` key) | -72.2 points |
| RU per status update | 14.1 (single large document) | 10.67 (hot/cold split) | -24% |
| Audit coverage of state changes | 25% (change feed, 30 of 120 observed) | 100% (append-only transition record, 7.38 RU each) | +75 points |

Source: [Post-Transformation/cosmos_design.json](Post-Transformation/cosmos_design.json)

Measured RU per pattern:

| Partition key | get by case_id | list by applicant_id | filter by status |
|---|---|---|---|
| `case_id` (chosen) | 1.14 | 3.09 | 24.54 |
| `applicant_id` | 2.87 | 3.09 | 24.54 |
| `status` | 2.87 | 3.09 | 24.54 |

---

## 6. Approach

1. **Discover** – Identify the three access patterns, their monthly volumes and the immutability constraint on the partition key.
2. **Baseline** – Load identical documents into three containers, each partitioned differently, and record write RU (24.0 RU per write in all three).
3. **Evaluate** – Measure RU per query pattern, partition skew, peak-load throttling, document shape and change-feed loss.
4. **Build** – Implement the RU meter (exhausting the lazy query iterator before reading the charge), the containers and the measurement loops.
5. **Measure** – Export results to `cosmos_design.json` and project monthly cost at serverless pricing.
6. **Hand off** – Record the design with named rejections and review triggers, then delete every `ds_` container.

### Selection Criteria

| Criterion | Weight | Why |
|---|---|---|
| Measured monthly RU cost | High | RU is the unit that is billed and provisioned |
| Partition cardinality and even distribution | High | The roughly 1,000 RU/s ceiling applies to a physical partition, so both properties are needed |
| Fit with the hottest access pattern | High | 40,000 reads/month should be a point read |
| Auditability of state changes | Medium | Compliance needs every intermediate state, not just the latest |

---

## 7. Repository Structure

```
Assignment11/
├── README.md
├── Pre-Transformation/      # "Before" state: reference material and unexecuted notebook
└── Post-Transformation/     # "After" state: executed notebook and measured outputs
    ├── DecisionStream_Day18_CosmosDB_Colab.ipynb
    ├── cosmos_design.json
    ├── project_analysis.md
    ├── Day18_CosmosDB_Generate_and_Govern_Exercises.md
    ├── .env.example
    └── (playbook, walkthrough, checklist and deck files)
```

---

## 8. Getting Started

### Prerequisites

- Python 3 (Google Colab or local Jupyter)
- An Azure Cosmos DB account using the **Core (SQL) API** (MongoDB and Cassandra APIs will not work)
- The account URI and primary key

### Setup

```powershell
cd Post-Transformation
copy .env.example .env
# edit .env and set COSMOS_URL and COSMOS_KEY
pip install azure-cosmos python-dotenv
```

### Run

Open `DecisionStream_Day18_CosmosDB_Colab.ipynb` in Colab or Jupyter and run the cells in order. Leaving the Cosmos URI blank runs a fully offline simulated mode. Run the final cleanup cell to delete every `ds_` container.

---

## 9. Results & Learnings

- **Key result:** `case_id` is the partition key. It wins on cost (586,125 vs 655,325 RU/month) and on skew (largest partition 0.3% vs 72.5% for `status`).
- **What worked:** Measuring RU rather than latency; point reads for the hot path; showing container headroom next to the single-partition ceiling; isolating the change-feed baseline before writing.
- **What didn't:** The hot/cold document split saved about 1.3x per status update (14.1 vs 10.67 RU), well short of the roughly 6x the exercise brief predicted. It should be treated as a small saving, not a reason on its own.
- **Recommendations:** Use serverless for the POC (provisioned has a 400 RU/s floor); do not rely on the change feed for audit; write an append-only transition record before releasing a result. Revisit the key if a new access pattern appears, volume exceeds the tested range, or skew rises.

> Caveats: 400 synthetic documents, one account, one day, low concurrency. The peak-load figures (500 ops/sec, 8 RU each) are assumed, not measured production traffic. Pricing figures should be re-checked against current Azure rates before quoting.

---

## 10. Next Steps

- [ ] Re-run with real traffic volumes and compare the actual RU bill to the projection
- [ ] Evaluate all-versions change-feed mode against the transition-record approach on the target account
- [ ] Investigate why the document-split saving was only about 1.3x (likely fixed per-write overhead at small document sizes)

---

## 11. Contact

Aditya Jagtap · adiraj2503@gmail.com
