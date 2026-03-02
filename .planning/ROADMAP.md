# Roadmap: ArchAI BOM MVP

## Overview

Take the solid backend (PDF ingestion, layout generation, persistence, async jobs) and build the missing pieces that turn it into a sellable product: a deterministic BOM calculator, a React frontend, export capabilities, and demo polish — targeting YC S26 application window (~mid-April 2026).

## Milestones

- ✅ **v0.1 Concerns Remediation** - Phase 1 (shipped 2026-02-27)
- 🚧 **v1.0 Sellable MVP** - Phases 2-6 (in progress)

## Phases

<details>
<summary>✅ v0.1 Concerns Remediation (Phase 1) - SHIPPED 2026-02-27</summary>

- [x] **Phase 1: Concerns Remediation** - Fix bugs, security, persistence, performance, async architecture (10 plans, all complete)

</details>

### 🚧 v1.0 Sellable MVP

- [x] **Phase 2: BOM Calculator Engine** - Deterministic geometry-to-priced-materials pipeline with Indian market pricing
- [ ] **Phase 2.5: Firebase Migration** - Replace Supabase PostgreSQL with Firebase Firestore
- [ ] **Phase 3: React SPA — Shell + Upload Flow** - Frontend skeleton with PDF upload, job polling, and results routing
- [ ] **Phase 4: Layout Visualization + BOM Table** - The money shot — 2D layout rendering and editable BOM spreadsheet
- [ ] **Phase 5: Export + Demo Polish** - Excel/PDF export and end-to-end demo flow under 3 minutes
- [ ] **Phase 6: Client-Facing Readiness** - Self-serve materials management, project history, deploy to public URL

## Phase Details

### Phase 2: BOM Calculator Engine
**Goal**: Generated layout geometry goes in, priced material line items come out — the core value proposition
**Depends on**: Phase 1
**Requirements**: BOM-01, BOM-02, BOM-03, BOM-04, BOM-05
**Success Criteria** (what must be TRUE):
  1. Calling the generate endpoint with a floorplan prompt returns a BOM with itemized materials, quantities, units, rates (₹), and a non-zero grand total
  2. Different room types produce different material lists (a bathroom includes waterproofing, a server room includes raised flooring)
  3. Materials pricing table is seeded with ~40-50 real Indian market items across all major categories (walls, flooring, ceiling, doors, electrical, paint)
  4. BOM quantities are geometrically correct — wall area drives drywall quantity, room area drives flooring quantity, perimeter drives baseboard quantity
**Plans:** 3 plans

Plans:
- [x] 02-bom-calculator-engine-01-PLAN.md — BOM domain types, materials catalog expansion (~45 items), materials repository
- [x] 02-bom-calculator-engine-02-PLAN.md — Deterministic BOM calculator engine (TDD: geometry → priced line items)
- [x] 02-bom-calculator-engine-03-PLAN.md — Wire BOM calculator into generate pipeline + integration tests

### Phase 2.5: Firebase Migration
**Goal**: Replace Supabase PostgreSQL (blocked in India) with Firebase Firestore — same API contracts, string IDs, all tests passing against emulator
**Depends on**: Phase 2
**Requirements**: MIG-01
**Success Criteria** (what must be TRUE):
  1. All 5 repositories use Firestore client instead of SQLAlchemy sessions
  2. All model classes are plain Pydantic BaseModel (no SQLModel/SQLAlchemy imports)
  3. ID fields are strings throughout the codebase (routes, services, workers, tests)
  4. All tests pass against Firestore emulator (`FIRESTORE_EMULATOR_HOST=localhost:8080`)
  5. No references to "supabase" or "SQLModel" remain in application code
**Plans:** 5 plans

Plans:
- [ ] 02.5-firebase-migration-01-PLAN.md — Firebase core infra (client singleton, config) + model conversion (SQLModel → Pydantic)
- [ ] 02.5-firebase-migration-02-PLAN.md — Rewrite all 5 repositories for Firestore
- [ ] 02.5-firebase-migration-03-PLAN.md — Type propagation (int→str IDs) across services/workers/routes + seed script
- [ ] 02.5-firebase-migration-04-PLAN.md — Test updates (Firestore emulator fixtures, str ID assertions)
- [ ] 02.5-firebase-migration-05-PLAN.md — Documentation + cleanup (CLAUDE.md, AGENTS.md, codebase docs)

### Phase 3: React SPA — Shell + Upload Flow
**Goal**: A contractor can open a browser, upload a PDF with a prompt, and watch it process
**Depends on**: Phase 2
**Requirements**: FE-01, FE-02, FE-06
**Success Criteria** (what must be TRUE):
  1. Opening localhost shows a clean upload page with drag-drop PDF zone and text prompt input
  2. Submitting an upload hits the backend API and shows a processing state with live status polling
  3. When the job completes, the user lands on a results page (content populated in Phase 4)
  4. API errors (no file, bad format, server error) show clear user-facing messages
**Plans:** 2 plans

Plans:
- [ ] 03-react-spa-shell-upload-01-PLAN.md — Frontend scaffold (Vite+React+TS+Tailwind), combined /process endpoint, /results endpoint, typed API client, React Router shell
- [ ] 03-react-spa-shell-upload-02-PLAN.md — Upload page (drag-drop PDF + prompt), Processing page (live job polling), Results page (BOM summary table + error handling)

### Phase 4: Layout Visualization + BOM Table
**Goal**: The dual-pane view that sells the product — visual layout on the left, priced BOM on the right
**Depends on**: Phase 3
**Requirements**: FE-03, FE-04, FE-05
**Success Criteria** (what must be TRUE):
  1. Generated rooms render as colored polygons on a 2D canvas/SVG with labels (room name + type)
  2. Interior walls, doors, and fixtures are visible and distinguishable
  3. BOM table displays all line items with material, quantity, unit, rate (₹), amount (₹), and grand total
  4. Editing a quantity or rate in the BOM table recalculates the line amount and grand total in real time
**Plans**: TBD

### Phase 5: Export + Demo Polish
**Goal**: The output is something a contractor would hand to a client, and the full demo runs flawlessly in <3 minutes
**Depends on**: Phase 4
**Requirements**: EXP-01, EXP-02, POL-01, POL-02, POL-03
**Success Criteria** (what must be TRUE):
  1. Clicking "Export Excel" downloads a formatted .xlsx with project header, BOM line items, and totals
  2. Clicking "Export PDF" downloads a report with the layout image, BOM table, and project summary
  3. Full end-to-end demo (upload → process → view layout + BOM → export) completes in under 3 minutes
  4. No crashes, no raw error traces, no broken states during a live demo walkthrough
**Plans**: TBD

### Phase 6: Client-Facing Readiness
**Goal**: A contractor can use the product without you in the room
**Depends on**: Phase 5
**Requirements**: CR-01, CR-02, CR-03, CR-04
**Success Criteria** (what must be TRUE):
  1. User can add, edit, and delete materials with custom rates from a settings/materials page
  2. User can see a list of past projects and re-open any previous result
  3. First-time user sees guidance on what to upload and what to type
  4. Product is live on a public URL — anyone with the link can use it
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 2 → 2.5 → 3 → 4 → 5 → 6

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Concerns Remediation | 10/10 | Complete | 2026-02-27 |
| 2. BOM Calculator Engine | 3/3 | Complete | 2026-02-28 |
| 2.5. Firebase Migration | 0/5 | Planned | - |
| 3. React SPA — Shell + Upload | 0/2 | Not started | - |
| 4. Layout Viz + BOM Table | 0/? | Not started | - |
| 5. Export + Demo Polish | 0/? | Not started | - |
| 6. Client-Facing Readiness | 0/? | Not started | - |
