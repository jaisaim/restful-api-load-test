# restful-api.dev Load Test Pipeline

> **PerPilot 6-Stage AI Load Testing Pipeline** for `https://api.restful-api.dev`
> Covers all 9 endpoints across GET / POST / PUT / PATCH / DELETE + JWT Auth

---

## Quick Start (3 Steps)

**Step 1 — Add your API Key as a GitHub Secret**
```
Settings > Secrets and variables > Actions > New repository secret
Name  : RESTFUL_API_KEY
Value : <your x-api-key from api.restful-api.dev>
```

**Step 2 — Trigger the pipeline**
```
Actions tab > PerPilot | restful-api.dev Load Test > Run workflow
```

**Step 3 — Fill in the 5 inputs and click Run workflow**

---

## What to Enter — Input Guide

| # | Input Field | What to Enter | Example | Origin Tag |
|---|-------------|---------------|---------|------------|
| 1 | `target_url` | Base URL of the API | `https://api.restful-api.dev` | `[INPUT]` |
| 2 | `environment` | Selects NFR threshold profile | `development` | `[INPUT]` |
| 3 | `virtual_users` | How many concurrent users | `5` (smoke) or `10` (load) | `[INPUT]` |
| 4 | `duration` | How long the test runs | `1m` (smoke) or `5m` (load) | `[INPUT]` |
| 5 | `collection_name` | The `{collectionName}` path param | `products` | `[INPUT]` |

---

## Parameter Colour Key

Every comment in the workflow and k6 script is tagged with one of these labels:

| Tag | Meaning | Example |
|-----|---------|---------|
| `[HARDCODED]` | Fixed value that never changes | `password: LoadTest@2024!` |
| `[INPUT]` | Provided by you at run time | `collection_name: products` |
| `[PASSED]` | Output by a previous stage via `GITHUB_OUTPUT` | `p95_ms` from Stage 02 → Stage 04 |
| `[EXTRACTED]` | Parsed from a live API response body | `jwt` from `POST /register` response |
| `[BUILT]` | Constructed at runtime from other values | `email = u<timestamp>@lt.io` |

---

## End-to-End Workflow

```
YOU (provide 5 inputs)
     │
     ▼
┌──────────────────────────────────────────────────────────────────┐
│ Stage 01: Target Application                                     │
│   INPUT  : target_url [INPUT]                                    │
│   ACTION : GET /collections with x-api-key [INPUT via Secret]   │
│   OUTPUT : http_status, reachable → [PASSED to all stages]       │
└──────────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────────┐
│ Stage 02: NFR Gathering                                          │
│   INPUT  : environment [INPUT]                                   │
│   ACTION : Selects threshold profile:                            │
│            development → p95<5000ms, err<5%,  tput>=5/s          │
│            staging     → p95<2000ms, err<1%,  tput>=20/s         │
│            production  → p95<500ms,  err<0.5%, tput>=50/s        │
│   OUTPUT : p95_ms, p99_ms, error_rate, throughput → [PASSED]     │
└──────────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────────┐
│ Stage 03: Workload Design                                        │
│   INPUT  : virtual_users [INPUT], duration [INPUT]               │
│   ACTION : Classifies test type + computes ramp-up               │
│            5 VUs  → smoke test  | 10-50 VUs → load test          │
│   OUTPUT : test_type, ramp_up, est_requests → [PASSED]           │
└──────────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────────┐
│ Stage 04: Script Generation                                      │
│   INPUTS : All [PASSED] from S01+S02+S03 + COLLECTION [INPUT]    │
│   ACTION : Generates k6 JS script with 9 endpoints:              │
│                                                                   │
│   SCENARIO A (40%%) — GET Endpoints                               │
│     1. GET /collections                                          │
│        x-api-key = [INPUT Secret] (sent on ALL requests)         │
│     2. GET /collections/{name}/objects                           │
│        {name} = COLLECTION [INPUT]                               │
│     3. GET /collections/{name}/objects/{id}                      │
│        {id} = [EXTRACTED] first id from GET /objects list        │
│                                                                   │
│   SCENARIO B (30%%) — Auth + Create                               │
│     4. POST /register                                            │
│        email = [BUILT] u<timestamp>@lt.io (unique per VU)        │
│        password = [HARDCODED] LoadTest@2024!                     │
│        RESPONSE: jwt token = [EXTRACTED] → stored in var         │
│     5. POST /login                                               │
│        email = [BUILT] same as register                          │
│        password = [HARDCODED] LoadTest@2024!                     │
│     6. POST /collections/{name}/objects                          │
│        {name} = COLLECTION [INPUT]                               │
│        Authorization: Bearer <jwt> = [EXTRACTED from /register]  │
│        body = [HARDCODED] product object                         │
│        RESPONSE: object id = [EXTRACTED] → stored in var         │
│                                                                   │
│   SCENARIO C (20%%) — Update                                      │
│     7. PUT /collections/{name}/objects/{id}                      │
│        {id} = [EXTRACTED] from POST /objects response            │
│        body = [HARDCODED] full replacement                       │
│     8. PATCH /collections/{name}/objects/{id}                    │
│        {id} = [EXTRACTED] same id                                │
│        body = [HARDCODED] partial update (name only)             │
│                                                                   │
│   SCENARIO D (10%%) — Delete                                      │
│     9. DELETE /collections/{name}/objects/{id}                   │
│        {id} = [EXTRACTED] from POST /objects                     │
│        body = [HARDCODED] null                                   │
│                                                                   │
│   OUTPUT : scripts/restful-api-test.js artifact                  │
└──────────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────────┐
│ Stage 05: Load Test Execution                                    │
│   INPUT  : script from S04, thresholds [PASSED S02]              │
│   ACTION : Installs k6, runs test, parses JSON output            │
│   OUTPUT : p95_actual, error_actual, rps_actual → [PASSED S06]  │
└──────────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────────┐
│ Stage 06: Store Results to GitHub                                │
│   INPUT  : all outputs from S01-S05                              │
│   ACTION : Publishes GitHub Step Summary + 3 artifacts           │
│   OUTPUT : load-test report (30-day retention)                   │
└──────────────────────────────────────────────────────────────────┘
     │
     ▼
GITHUB (Step Summary + Artifacts: script + raw results + report)
```

---

## How Parameters are Hardcoded vs Passed

**Hardcoded in the k6 script (Stage 04):**
- Register password: `LoadTest@2024!`
- Register display name: `LoadTestUser`
- POST body fields: `year: 2024`, `price: 999.99`, `cpu: 'Intel i7'`
- PUT body: `name: 'PUT Updated'`, `color: 'silver'`
- PATCH body: `name: 'PATCH Updated Name'`
- Ramp-down duration: `15s`
- Think time between iterations: `sleep(1)`
- Fallback object ID for GET single: `'1'`

**Passed between API calls (within the k6 script):**
```
POST /register -> response.token  ─── stored in: jwt
                                       passed to:  POST /objects (Bearer header)
                                                   PUT /objects/{id}
                                                   PATCH /objects/{id}
                                                   DELETE /objects/{id}

POST /objects  -> response.id     ─── stored in: oid
                                       passed to:  PUT /objects/{id} as path param
                                                   PATCH /objects/{id} as path param
                                                   DELETE /objects/{id} as path param

GET /objects   -> response[0].id  ─── stored in: sid
                                       passed to:  GET /objects/{id} as path param
```

**Passed between pipeline stages (via GITHUB_OUTPUT):**
```
Stage 01 → target_url, http_status → Stage 04
Stage 02 → p95_ms, p99_ms, error_rate, throughput → Stage 04, Stage 05
Stage 03 → test_type, ramp_up, virtual_users, duration → Stage 04
Stage 05 → p95_actual, error_actual, rps_actual → Stage 06
```

---

## Repository Structure

```
restful-api-load-test/
├── .github/
│   └── workflows/
│       └── restful-api-pipeline.yml  ← 6-stage pipeline (380 lines)
└── README.md                         ← This guide
```

---

## NFR Threshold Profiles

| Environment | P95 | P99 | Error Rate | Min Throughput |
|-------------|-----|-----|------------|----------------|
| `development` | < 5000ms | < 8000ms | < 5% | >= 5 req/s |
| `staging` | < 2000ms | < 3000ms | < 1% | >= 20 req/s |
| `production` | < 500ms | < 800ms | < 0.5% | >= 50 req/s |

---

*PerPilot AI Load Testing Pipeline | github.com/jaisaim/restful-api-load-test*
