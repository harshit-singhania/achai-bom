---
phase: 03-react-spa-shell-upload
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - frontend/package.json
  - frontend/vite.config.ts
  - frontend/tsconfig.json
  - frontend/tailwind.config.js
  - frontend/postcss.config.js
  - frontend/index.html
  - frontend/src/main.tsx
  - frontend/src/App.tsx
  - frontend/src/index.css
  - frontend/src/types/api.ts
  - frontend/src/api/client.ts
  - frontend/src/components/Layout.tsx
  - frontend/src/pages/UploadPage.tsx
  - frontend/src/pages/ProcessingPage.tsx
  - frontend/src/pages/ResultsPage.tsx
  - app/services/process_service.py
  - app/api/routes.py
  - app/workers/job_runner.py
  - app/repositories/bom_repository.py
  - tests/test_process_endpoint.py
  - .gitignore
autonomous: true
requirements: [FE-01, FE-06]

must_haves:
  truths:
    - "Vite dev server starts and serves React app at localhost:5173"
    - "POST /api/v1/process accepts PDF + prompt and returns 202 with job_id"
    - "GET /api/v1/results/<bom_id> returns layout + BOM data"
    - "React Router navigates between /, /processing/:jobId, and /results/:bomId"
    - "API client functions have typed request/response contracts"
  artifacts:
    - path: "frontend/package.json"
      provides: "Vite + React + TypeScript project definition"
      contains: "react-router-dom"
    - path: "frontend/vite.config.ts"
      provides: "Vite config with API proxy to Flask backend"
      contains: "proxy"
    - path: "frontend/src/api/client.ts"
      provides: "Typed API client for process, job polling, results"
      exports: ["uploadAndProcess", "getJobStatus", "getResults"]
    - path: "frontend/src/types/api.ts"
      provides: "TypeScript interfaces matching backend responses"
      exports: ["ProcessResponse", "JobStatusResponse", "BOMResultResponse"]
    - path: "app/services/process_service.py"
      provides: "Combined ingest+generate service for /process endpoint"
      exports: ["enqueue_process", "ProcessValidationError"]
    - path: "app/api/routes.py"
      provides: "POST /process and GET /results/<bom_id> routes"
      contains: "process"
  key_links:
    - from: "frontend/src/api/client.ts"
      to: "/api/v1/process"
      via: "fetch POST with FormData"
      pattern: "fetch.*api/v1/process"
    - from: "frontend/src/api/client.ts"
      to: "/api/v1/jobs"
      via: "fetch GET for polling"
      pattern: "fetch.*api/v1/jobs"
    - from: "frontend/src/App.tsx"
      to: "frontend/src/pages/*.tsx"
      via: "React Router routes"
      pattern: "Route.*path"
    - from: "app/api/routes.py"
      to: "app/services/process_service.py"
      via: "enqueue_process import"
      pattern: "from app.services.process_service import"
---

<objective>
Scaffold the React frontend, create the combined backend endpoint for upload+process, and wire up the API client with routing.

Purpose: Establish the full frontend-to-backend plumbing so that subsequent UI work (Plan 02) has a working scaffold, typed API client, and backend endpoints to call. This is the foundation — no user-facing UI yet, just the skeleton.

Output: A Vite+React+TypeScript project in `frontend/`, a combined `/api/v1/process` backend endpoint, a `/api/v1/results/<bom_id>` endpoint, and React Router with 3 stub pages.
</objective>

<execution_context>
@/Users/harshit/.config/opencode/get-shit-done/workflows/execute-plan.md
@/Users/harshit/.config/opencode/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/STATE.md

<interfaces>
<!-- Backend API contracts the frontend must target -->

From app/api/routes.py (existing endpoints):
```
POST /api/v1/ingest    → 202 {job_id, status, status_url, pdf_id?}
POST /api/v1/generate  → 202 {job_id, status, status_url}
GET  /api/v1/jobs/<id> → {job_id, job_type, status, result_ref, error_message, created_at, started_at, finished_at}
GET  /api/v1/status/<pdf_id> → {pdf_id, status, generation_summary, jobs}
```

From app/services/status_service.py (job status response shape):
```python
# status field values: "queued" | "running" | "succeeded" | "failed"
# result_ref on success: {"result_type": "bom", "result_id": <bom_id>, "success": true, "iterations_used": N}
# error_message on failure: "ExceptionType: message"
```

From app/models/bom.py (BOM response types):
```python
class BOMLineItem(BaseModel):
    material_name: str
    category: str
    quantity: float
    unit: str
    rate_inr: float
    amount_inr: float
    room_name: str | None
    notes: str | None

class BOMResult(BaseModel):
    line_items: list[BOMLineItem]
    grand_total_inr: float
    room_count: int
    total_area_sqm: float
```

From app/workers/job_runner.py (bom_data JSON shape stored in generated_boms):
```python
bom_data = {
    "success": True,
    "iterations_used": N,
    "constraint_result": {...},
    "layout": {rooms: [...], interior_walls: [...], doors: [...], fixtures: [...], ...},
    "prompt": "...",
    "line_items": [{material_name, category, quantity, unit, rate_inr, amount_inr, room_name, notes}, ...],
    "room_count": N,
    "total_area_sqm": N.N,
}
```

From app/core/config.py:
```python
API_V1_PREFIX: str = "/api/v1"
API_AUTH_KEY: str = ""          # Empty in debug mode = no auth required
MAX_CONTENT_LENGTH: int = 16MB
ALLOWED_ORIGINS: str = "*"
```
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Scaffold Vite + React + TypeScript + Tailwind frontend</name>
  <files>
    frontend/package.json
    frontend/vite.config.ts
    frontend/tsconfig.json
    frontend/tsconfig.app.json
    frontend/tailwind.config.js
    frontend/postcss.config.js
    frontend/index.html
    frontend/src/main.tsx
    frontend/src/App.tsx
    frontend/src/index.css
    frontend/src/vite-env.d.ts
    .gitignore
  </files>
  <action>
    1. Run `npm create vite@latest frontend -- --template react-ts` from the project root to scaffold the React+TypeScript project.

    2. Install dependencies in `frontend/`:
       ```
       npm install react-router-dom
       npm install -D tailwindcss @tailwindcss/postcss postcss autoprefixer
       ```

    3. Configure Vite proxy in `frontend/vite.config.ts`:
       - Proxy `/api` → `http://localhost:5000` so frontend dev server forwards API calls to Flask backend
       - Set `changeOrigin: true`

    4. Set up Tailwind CSS:
       - `frontend/tailwind.config.js`: content paths `["./index.html", "./src/**/*.{ts,tsx}"]`
       - `frontend/postcss.config.js`: tailwindcss + autoprefixer plugins
       - `frontend/src/index.css`: Add `@tailwind base; @tailwind components; @tailwind utilities;` directives plus body base styles (bg-gray-50, font-sans, antialiased)

    5. Create folder structure:
       ```
       frontend/src/
       ├── api/         (API client)
       ├── components/  (shared components)
       ├── hooks/       (custom hooks)
       ├── pages/       (route pages)
       └── types/       (TypeScript interfaces)
       ```
       Create empty `.gitkeep` or index files in each to establish the structure.

    6. Update root `.gitignore` to add:
       ```
       frontend/node_modules
       frontend/dist
       ```

    7. Clean up the default Vite template: remove App.css, logo SVGs, default counter code. Replace App.tsx with a minimal "ArchAI BOM" heading for now.
  </action>
  <verify>
    <automated>cd frontend && npm run build && echo "BUILD OK"</automated>
  </verify>
  <done>Vite dev server starts at localhost:5173, TypeScript compiles, Tailwind processes CSS, build produces dist/ output.</done>
</task>

<task type="auto">
  <name>Task 2: Backend — combined /process endpoint + /results endpoint</name>
  <files>
    app/services/process_service.py
    app/api/routes.py
    app/workers/job_runner.py
    app/repositories/bom_repository.py
    tests/test_process_endpoint.py
  </files>
  <action>
    **Why a combined endpoint:** The current API requires two sequential calls (ingest → generate) with the spatial_graph passed between them. The frontend needs a single "upload PDF + prompt → get BOM" action for good UX. A combined `/process` endpoint handles the full pipeline in one background job.

    1. **Create `app/services/process_service.py`** following the exact pattern of `ingest_service.py`:
       - `ProcessValidationError(ValueError)` — for 400s
       - `ProcessEnqueueError(RuntimeError)` — for 500s
       - `ProcessJobResult` dataclass with `job_id: int`, `floorplan_id: int | None`, `status_url: str`
       - `enqueue_process(filename: str, file_stream: IO[bytes], prompt: str) -> ProcessJobResult`:
         - Validate filename is non-empty and ends with `.pdf`
         - Validate prompt is a non-empty string
         - Save file to temp path using `tempfile.NamedTemporaryFile(delete=False, suffix=".pdf")`
         - Create floorplan record via `create_floorplan(pdf_storage_url=filename, status="uploaded")` (non-blocking, log warning on failure)
         - Build payload JSON: `{"pdf_path": tmp_path, "prompt": prompt, "floorplan_id": floorplan_id}`
         - Create job with `job_type="process"`, enqueue via `enqueue_process_job(job_id)` (add this to queue_worker.py alongside existing enqueue functions)
         - Return ProcessJobResult

    2. **Add routes to `app/api/routes.py`**:
       - `POST /process`: Accept multipart form with `file` (PDF) and `prompt` (text field). Import and call `enqueue_process()`. Return 202 with `{job_id, status: "queued", status_url}`. Follow the exact error handling pattern of the existing `/ingest` route.
       - `GET /results/<int:bom_id>`: Import `get_bom_with_data(bom_id)` from bom_repository. Return the full BOM record including `bom_data`, `total_cost_inr`, `created_at`. Return 404 if not found.

    3. **Add `_run_process_job` to `app/workers/job_runner.py`**:
       - Add `"process"` case to the `run_job()` dispatch.
       - `_run_process_job(job)` implementation:
         a. Parse payload for `pdf_path`, `prompt`, `floorplan_id`
         b. Update floorplan status to "processing" if floorplan_id exists
         c. Extract vectors: `from app.services.pdf_extractor import extract_vectors` → `extraction = extract_vectors(pdf_path)`
         d. Detect walls: `from app.services.wall_detector import detect_walls` → `wall_result = detect_walls(extraction)`
         e. Build SpatialGraph: `SpatialGraph(walls=wall_result.wall_segments, page_dimensions=(extraction.page_width, extraction.page_height))`
         f. Generate layout: `generate_validated_layout(spatial_graph, prompt)` using default settings
         g. If layout success: calculate BOM with `calculate_bom(layout, materials)`, persist via `create_bom()`
         h. Update floorplan status to "processed" on success, "error" on failure
         i. Return `result_ref = {"result_type": "bom", "result_id": bom_id, "success": True}`
         j. On generation failure: raise RuntimeError with error message
       - Follow the exact error handling and logging patterns from `_run_generate_job`

    4. **Add `get_bom_with_data(bom_id)` to `app/repositories/bom_repository.py`**:
       - Query `generated_boms` by id, return dict with `id`, `floorplan_id`, `total_cost_inr`, `bom_data`, `created_at`. Return `None` if not found.
       - Follow the exact pattern of existing repository functions (use session context manager, return typed dict, log warnings on failure).

    5. **Add `enqueue_process_job(job_id)` to `app/workers/queue_worker.py`**:
       - Follow the exact pattern of `enqueue_ingest_job` and `enqueue_generate_job`.

    6. **Create `tests/test_process_endpoint.py`** with basic tests:
       - Test POST /process without file → 400
       - Test POST /process with non-PDF file → 400
       - Test POST /process without prompt → 400
       - Test POST /process happy path → 202 with job_id and status_url
       - Test GET /results with unknown bom_id → 404
       - Use the same test patterns from `tests/test_async_jobs.py` (mock the queue worker).
  </action>
  <verify>
    <automated>pytest tests/test_process_endpoint.py -v && pytest tests/test_async_jobs.py tests/test_status_endpoint.py -q</automated>
  </verify>
  <done>POST /process returns 202 with job_id for valid PDF+prompt uploads. GET /results/<bom_id> returns full BOM data. Existing tests still pass (no regressions). New endpoint tests pass.</done>
</task>

<task type="auto">
  <name>Task 3: TypeScript types + API client + React Router shell</name>
  <files>
    frontend/src/types/api.ts
    frontend/src/api/client.ts
    frontend/src/components/Layout.tsx
    frontend/src/pages/UploadPage.tsx
    frontend/src/pages/ProcessingPage.tsx
    frontend/src/pages/ResultsPage.tsx
    frontend/src/App.tsx
  </files>
  <action>
    1. **Create `frontend/src/types/api.ts`** — TypeScript interfaces matching the backend API responses:
       ```typescript
       // POST /api/v1/process response (202)
       export interface ProcessResponse {
         job_id: number;
         status: string;
         status_url: string;
       }

       // GET /api/v1/jobs/:id response
       export type JobStatus = "queued" | "running" | "succeeded" | "failed";
       export interface JobStatusResponse {
         job_id: number;
         job_type: string;
         status: JobStatus;
         result_ref: ResultRef | null;
         error_message: string | null;
         created_at: string;
         started_at: string | null;
         finished_at: string | null;
       }
       export interface ResultRef {
         result_type: string;
         result_id?: number;
         success?: boolean;
         iterations_used?: number;
       }

       // GET /api/v1/results/:bomId response
       export interface BOMLineItem {
         material_name: string;
         category: string;
         quantity: number;
         unit: string;
         rate_inr: number;
         amount_inr: number;
         room_name: string | null;
         notes: string | null;
       }
       export interface BOMResultResponse {
         id: number;
         floorplan_id: number;
         total_cost_inr: number;
         bom_data: {
           layout: Record<string, unknown>;
           line_items: BOMLineItem[];
           room_count: number;
           total_area_sqm: number;
           prompt: string;
         };
         created_at: string;
       }

       // Error response shape
       export interface ApiError {
         error: string;
       }
       ```

    2. **Create `frontend/src/api/client.ts`** — typed API functions using fetch:
       ```typescript
       const API_BASE = "/api/v1";

       export async function uploadAndProcess(file: File, prompt: string): Promise<ProcessResponse> {
         const formData = new FormData();
         formData.append("file", file);
         formData.append("prompt", prompt);
         const res = await fetch(`${API_BASE}/process`, { method: "POST", body: formData });
         if (!res.ok) {
           const err = await res.json();
           throw new Error(err.error || `Upload failed (${res.status})`);
         }
         return res.json();
       }

       export async function getJobStatus(jobId: number): Promise<JobStatusResponse> { ... }
       export async function getResults(bomId: number): Promise<BOMResultResponse> { ... }
       ```
       - No hardcoded API key (MVP decision: no auth in frontend)
       - All functions throw Error with user-friendly messages on non-2xx responses
       - Use relative URLs (Vite proxy handles routing to backend)

    3. **Create `frontend/src/components/Layout.tsx`** — app shell:
       - Header bar with "ArchAI BOM" logo text (text-xl font-bold) and tagline "Floorplan → BOM in minutes"
       - Use Tailwind: white header with bottom border, main content area with gray-50 background, max-w-4xl centered container
       - Renders `<Outlet />` from react-router-dom for nested route content

    4. **Update `frontend/src/App.tsx`** with React Router:
       ```tsx
       <BrowserRouter>
         <Routes>
           <Route element={<Layout />}>
             <Route path="/" element={<UploadPage />} />
             <Route path="/processing/:jobId" element={<ProcessingPage />} />
             <Route path="/results/:bomId" element={<ResultsPage />} />
           </Route>
         </Routes>
       </BrowserRouter>
       ```

    5. **Create stub pages** (minimal, just enough to verify routing):
       - `UploadPage.tsx`: Render heading "Upload Floorplan" + placeholder text. Will be fully built in Plan 02.
       - `ProcessingPage.tsx`: Read `jobId` from `useParams()`, render "Processing job {jobId}...". Will be fully built in Plan 02.
       - `ResultsPage.tsx`: Read `bomId` from `useParams()`, render "Results for BOM {bomId}". Will be fully built in Plan 02.
  </action>
  <verify>
    <automated>cd frontend && npx tsc --noEmit && npm run build && echo "TYPES + BUILD OK"</automated>
  </verify>
  <done>TypeScript compiles with zero errors. All 3 routes render their stub content. API client exports typed functions. Build succeeds.</done>
</task>

</tasks>

<verification>
1. `cd frontend && npm run build` completes without errors
2. `cd frontend && npx tsc --noEmit` — zero TypeScript errors
3. `pytest tests/test_process_endpoint.py -v` — all process endpoint tests pass
4. `pytest tests/ -q` — no regressions in existing test suite
5. Frontend dev server starts: `cd frontend && npm run dev` serves at localhost:5173
</verification>

<success_criteria>
- Vite + React + TypeScript + Tailwind project exists in `frontend/` and builds cleanly
- POST /api/v1/process accepts multipart PDF+prompt, returns 202 with job_id
- GET /api/v1/results/:bomId returns full BOM data with layout and line items
- React Router serves 3 routes: /, /processing/:jobId, /results/:bomId
- API client has typed functions: uploadAndProcess, getJobStatus, getResults
- All backend tests pass (new + existing)
</success_criteria>

<output>
After completion, create `.planning/phases/03-react-spa-shell-upload/03-react-spa-shell-upload-01-SUMMARY.md`
</output>
