**English** · [Español](README.es.md)

# Nemea

Nemea is a web app for managing and calculating costs, prices, and profit margins for leather goods. Each product gets its cost breakdown (leather, supplies, packaging, etc.), and a calculator works out the margin for a given sale price, accounting for costs + taxes + payment fees + installment interest. It replaces the Google Sheets that used to hold all of this, and adds Tiendanube configuration, pricing-scenario simulation, and an investor dashboard.

> **Note:** this is the *orchestrator* repository. It ties together the two sub-projects — front and back — which live in their own repositories and are cloned inside this one. The orchestrator's job is to run **GSD (get-shit-done)**, a framework that structures application development phase by phase.

---

## The two sub-projects

| Project | Repo | What it does | Stack | Link |
|---------|------|--------------|-------|------|
| **Frontend** | `nemea-front` | The web app: products, costs, supplies, expenses, calculator, scenarios, dashboard. Google auth + demo login. | Next.js 16 · TypeScript strict · Tailwind v4 · Shadcn/ui · Auth.js v5 | [→ repo](https://github.com/basualdofelipe/nemea-front) |
| **Backend** | `nemea-back` | REST API: business logic, cost engine, pricing engine, RBAC, persistence. | NestJS 11 · TypeScript strict · TypeORM · PostgreSQL 16 · JWT | [→ repo](https://github.com/basualdofelipe/nemea-back) |

```
┌──────────┐      ┌────────────────────┐      ┌────────────────────┐      ┌──────────────┐
│  User    │ ───► │  nemea-front       │ ───► │  nemea-back        │ ───► │  PostgreSQL  │
│ (browser)│      │  Next.js   · :3000 │      │  NestJS    · :4000 │      │   (Docker)   │
└──────────┘      └────────────────────┘      └────────────────────┘      └──────────────┘
                       Auth.js v5                  JWT + RBAC
                    (Google OAuth)             (11 permissions/role)
```

Each sub-repo has its own README with setup, scripts, internal architecture, and data model.

---

## How GSD works

Every phase goes through a pipeline of versioned artifacts kept in `.planning/`: define *what* to build, then *how*, plan the code, review the plan, execute, and review what was built.

### The cycle

It starts with `/new-project`, which creates a **MILESTONE** (a large goal, e.g. *"replace the Google Sheets"*) and splits it into **PHASES**. Each phase runs this cycle:

```
                       ┌──────────────────────── MILESTONE ────────────────────────┐
     /new-project ───► │   large goal  ──►  split into PHASES (1, 2, 3 …)           │
                       └─────────────────────────────┬──────────────────────────────┘
                                                      │
            ┌──────────────────  for each PHASE  ─────┘
            ▼
   ╭───────────╮   ╭───────────╮   ╭───────────╮   ╭───────────────╮
   │ ① SPECS   │──►│ ② DISCUSS │──►│  ③ PLAN   │──►│ ④ PLAN REVIEW │
   │   what?   │   │   how?    │   │ the plan  │   │  looks ok?    │
   ╰───────────╯   ╰───────────╯   ╰───────────╯   ╰───────┬───────╯
     success        design          task by         missing? breaks?
     criteria       decisions        task                  │
                                                          │ ──► back to ③ (re-plan)
                                                          ▼ ok
   ╭───────────╮   ╭───────────────╮   ╭─────────────────╮
   │ ⑥ VERIFY  │◄──│ ⑤ CODE REVIEW │◄──│   EXECUTE       │
   │  meets    │   │  any bugs?    │   │  writes code    │
   │  goal?    │   │  security?    │   │  in front/back  │
   ╰───────────╯   ╰───────┬───────╯   ╰─────────────────╯
                     bugs? │ ──► REVIEW-FIX (fix and re-review)
```

The cycle isn't linear: the *plan review* can send things back to re-planning before any code is written, and the *code review* can trigger fixes (`REVIEW-FIX`) before the phase closes.

### Each stage maps to an artifact

Every box in the diagram is a real file under `.planning/phases/<phase>/`:

| Stage | What happens | Artifact |
|-------|--------------|----------|
| **① Specs** | Define *what* to build and the success criteria. | `REQUIREMENTS.md` · `NN-SPEC.md` |
| **② Discuss** | Decide *how* to solve it. | `NN-CONTEXT.md` (+ `NN-RESEARCH.md`) |
| **③ Plan** | An agent reads the real repo and proposes a code plan, task by task. | `NN-PP-PLAN.md` |
| **④ Plan review** | The plan is audited before coding: assumptions, missing steps, regressions. | `NN-REVIEWS.md` |
| **Execute** | The plan is executed and the code is written in the sub-repo, with atomic commits. | `NN-PP-SUMMARY.md` |
| **⑤ Code review** | The written code is reviewed: bugs, security, conventions, tests. | `NN-REVIEW.md` |
| **⑥ Verify** | UAT against the phase's success criteria. | `NN-VERIFICATION.md` / `NN-UAT.md` |

> `NN` = phase number, `PP` = plan number within the phase. A phase can have several plans (phase 5 has 4, phase 12.4 has 8). Not every phase has every artifact, and naming isn't 100% uniform across phases.

### What lives in `.planning/`

```
.planning/
├── PROJECT.md          # Context: what it is, who for, constraints, decisions
├── ROADMAP.md          # Milestone phases, status, and success criteria
├── REQUIREMENTS.md     # Requirements with traceability
├── STATE.md            # Current project state + metrics
├── research/           # Pre-milestone research
├── codebase/           # Map of the existing code
└── phases/
    └── 05-products-and-bom/
        ├── 05-CONTEXT.md            # ② discuss
        ├── 05-RESEARCH.md           # ② research
        ├── 05-01-PLAN.md  …         # ③ plan (one per unit of work)
        ├── 05-01-SUMMARY.md  …      # execute
        └── 05-VERIFICATION.md       # ⑥ verify
```

### Example — Phase 5 (Products & BOM)

- **② Discuss** settled the SKU format `type.name.finish.color.size` → `1.2.1.3.0` (inherited from the original Google Sheets), batch creation per product as the Cartesian product of colors × sizes, and BOM versioning via `is_active`.
- **③ Plan** was split into 4 plans (entity + migrations → BOM + prices → products table → UI editors), each listing the exact files to touch.
- **⑥ Verify** closed with 5/5 criteria verified, with line-level evidence. E.g.: when the BOM changes, the previous one is preserved with `is_active=false` — verified in `products.service.ts:296-301`, inside a transaction.

It's all in `.planning/phases/05-products-and-bom/`.

### By the numbers

The project went through **17 phases** and **54 executed plans**, all versioned in `.planning/`.

---

## Run it locally

Clone-and-run with the demo login: no Google Cloud or OAuth setup needed. You'll need **Node 20+**, **npm**, **Docker** (for PostgreSQL), and two terminals.

### 1. Clone the orchestrator and the two repos inside it

```bash
git clone https://github.com/basualdofelipe/Nemea-web.git
cd Nemea-web
git clone https://github.com/basualdofelipe/nemea-back.git
git clone https://github.com/basualdofelipe/nemea-front.git
```

### 2. Backend + database

```bash
cd nemea-back
npm install
cp .env.example .env
docker compose up -d postgres     # PostgreSQL 16 on :5432
npm run start:dev                 # API at http://localhost:4000/api
```

Migrations run automatically on startup (`migrationsRun: true`), including the one that seeds the demo user (`demo@nemea.app`).

### 3. Frontend (another terminal)

```bash
cd nemea-front
npm install
cp .env.example .env.local
npm run dev                       # app at http://localhost:3000
```

The `.env.example` ships an `AUTH_SECRET` placeholder that runs as-is locally (generate your own with `npx auth secret` if you want).

### 4. Log in

Open `http://localhost:3000` and use the **"Entrar como demo"** button: it logs you in as `demo@nemea.app` (admin role, all permissions). The demo login is enabled by default in the `.env.example` files and turned off in production (`DEMO_LOGIN_ENABLED=false`).

> No hosted instance. The infra is wired for Vercel (front) + Railway (back + DB), but the project runs locally. Setup details are in each sub-repo's README.

---

## Roadmap

| Milestone | Scope | Status |
|-----------|-------|--------|
| **v1.0** | Google Sheets replacement: supplies/prices CRUD, products/costs (versioned BOM), expenses, and auth. | Done |
| **v1.1** | Hardening, granular RBAC, product UX, Tiendanube config, forward/inverse calculator, scenarios, and investor dashboard. | In progress |
| **v2+** | Tiendanube API integration (real sales), B2B clients, orders, and purchase requests to workshops. | Planned |

Per-phase detail in [`.planning/ROADMAP.md`](.planning/ROADMAP.md).

---

## Repo structure

```
Nemea-web/
├── .planning/        # GSD: specs, roadmap, phases, research, state
├── CLAUDE.md         # Project conventions
├── README.md         # this file (default, English)
├── README.es.md      # Spanish version
├── nemea-front/      # cloned here (separate repo, gitignored)
└── nemea-back/       # cloned here (separate repo, gitignored)
```
