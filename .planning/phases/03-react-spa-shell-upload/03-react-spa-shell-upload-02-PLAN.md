---
phase: 03-react-spa-shell-upload
plan: 02
type: execute
wave: 2
depends_on: ["03-react-spa-shell-upload-01"]
files_modified:
  - frontend/src/pages/UploadPage.tsx
  - frontend/src/pages/ProcessingPage.tsx
  - frontend/src/pages/ResultsPage.tsx
  - frontend/src/hooks/useJobPolling.ts
  - frontend/src/components/FileDropZone.tsx
  - frontend/src/components/ErrorBanner.tsx
  - frontend/src/components/Layout.tsx
autonomous: false
requirements: [FE-01, FE-02, FE-06]

must_haves:
  truths:
    - "User sees a drag-drop PDF zone and a text prompt input on the upload page"
    - "Submitting a valid PDF + prompt navigates to a processing page with live status updates"
    - "Processing page polls GET /api/v1/jobs/:id every 2 seconds and shows current status"
    - "When job succeeds, user is auto-redirected to results page showing BOM data"
    - "When job fails, user sees a clear error message with a retry option"
    - "API errors (no file, bad format, server error) show user-friendly error banners"
    - "Results page shows the prompt, room count, total area, and grand total cost"
  artifacts:
    - path: "frontend/src/pages/UploadPage.tsx"
      provides: "PDF upload form with drag-drop and prompt input"
      min_lines: 80
    - path: "frontend/src/pages/ProcessingPage.tsx"
      provides: "Job polling with status display and auto-redirect"
      min_lines: 60
    - path: "frontend/src/pages/ResultsPage.tsx"
      provides: "BOM results summary page (layout viz deferred to Phase 4)"
      min_lines: 50
    - path: "frontend/src/hooks/useJobPolling.ts"
      provides: "Custom hook for polling job status with cleanup"
      exports: ["useJobPolling"]
    - path: "frontend/src/components/FileDropZone.tsx"
      provides: "Reusable drag-drop file input component"
      exports: ["FileDropZone"]
    - path: "frontend/src/components/ErrorBanner.tsx"
      provides: "Dismissible error message banner"
      exports: ["ErrorBanner"]
  key_links:
    - from: "frontend/src/pages/UploadPage.tsx"
      to: "frontend/src/api/client.ts"
      via: "uploadAndProcess() call on form submit"
      pattern: "uploadAndProcess"
    - from: "frontend/src/hooks/useJobPolling.ts"
      to: "frontend/src/api/client.ts"
      via: "getJobStatus() called on interval"
      pattern: "getJobStatus"
    - from: "frontend/src/pages/ProcessingPage.tsx"
      to: "frontend/src/hooks/useJobPolling.ts"
      via: "useJobPolling(jobId) hook consumption"
      pattern: "useJobPolling"
    - from: "frontend/src/pages/ProcessingPage.tsx"
      to: "/results/:bomId"
      via: "navigate() on job success"
      pattern: "navigate.*results"
    - from: "frontend/src/pages/ResultsPage.tsx"
      to: "frontend/src/api/client.ts"
      via: "getResults(bomId) on mount"
      pattern: "getResults"
---

<objective>
Build the three user-facing pages: Upload (drag-drop PDF + prompt), Processing (live job polling with status), and Results (BOM summary with data from backend).

Purpose: This is the core user flow — the contractor opens the browser, uploads a PDF, types a description, watches it process, and sees the results. This plan delivers the Phase 3 success criteria.

Output: Fully functional upload → process → results flow with error handling and live polling.
</objective>

<execution_context>
@/Users/harshit/.config/opencode/get-shit-done/workflows/execute-plan.md
@/Users/harshit/.config/opencode/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/STATE.md
@.planning/phases/03-react-spa-shell-upload/03-react-spa-shell-upload-01-SUMMARY.md

<interfaces>
<!-- From Plan 01 — types and API client this plan depends on -->

From frontend/src/types/api.ts:
```typescript
export interface ProcessResponse {
  job_id: number;
  status: string;
  status_url: string;
}

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

export interface ApiError {
  error: string;
}
```

From frontend/src/api/client.ts:
```typescript
export async function uploadAndProcess(file: File, prompt: string): Promise<ProcessResponse>;
export async function getJobStatus(jobId: number): Promise<JobStatusResponse>;
export async function getResults(bomId: number): Promise<BOMResultResponse>;
```

From frontend/src/App.tsx (routing):
```tsx
<Route path="/" element={<UploadPage />} />
<Route path="/processing/:jobId" element={<ProcessingPage />} />
<Route path="/results/:bomId" element={<ResultsPage />} />
```
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Shared components + polling hook</name>
  <files>
    frontend/src/components/FileDropZone.tsx
    frontend/src/components/ErrorBanner.tsx
    frontend/src/hooks/useJobPolling.ts
  </files>
  <action>
    1. **Create `frontend/src/components/FileDropZone.tsx`** — drag-drop PDF upload zone:
       - Props: `onFileSelect: (file: File) => void`, `selectedFile: File | null`, `disabled?: boolean`
       - Render a dashed border drop zone (border-2 border-dashed border-gray-300 rounded-lg p-8)
       - On drag over: change border color to blue-500, bg-blue-50
       - On drop: validate file is PDF (check `.pdf` extension and `application/pdf` MIME type), call `onFileSelect`
       - Also include a hidden `<input type="file" accept=".pdf">` with a clickable "Browse files" link
       - When `selectedFile` is set, show the filename with a green checkmark icon and file size
       - When `disabled` is true, reduce opacity and prevent interactions
       - Reject non-PDF files with inline error text "Only PDF files are accepted"
       - Use Tailwind classes for all styling (no CSS files)

    2. **Create `frontend/src/components/ErrorBanner.tsx`** — dismissible error display:
       - Props: `message: string | null`, `onDismiss: () => void`
       - If `message` is null, render nothing
       - Render a red-tinted banner (bg-red-50 border border-red-200 text-red-700 rounded-lg p-4)
       - Include an X button to dismiss (calls `onDismiss`)
       - Use `role="alert"` for accessibility

    3. **Create `frontend/src/hooks/useJobPolling.ts`** — custom hook for job status polling:
       - Signature: `useJobPolling(jobId: number | null, intervalMs?: number)`
       - Returns: `{ status: JobStatus | null, jobData: JobStatusResponse | null, error: string | null, isPolling: boolean }`
       - Uses `useEffect` with `setInterval` to poll `getJobStatus(jobId)` every `intervalMs` (default 2000ms)
       - **Stops polling** when status reaches "succeeded" or "failed" (terminal states)
       - **Cleanup**: clears interval on unmount or when jobId changes (prevents memory leaks)
       - Sets `error` string if the fetch throws or if job status is "failed" (use job's `error_message`)
       - If `jobId` is null, does nothing (returns all nulls)
       - Import `getJobStatus` from `../api/client` and `JobStatusResponse`, `JobStatus` from `../types/api`
  </action>
  <verify>
    <automated>cd frontend && npx tsc --noEmit && echo "TYPES OK"</automated>
  </verify>
  <done>FileDropZone accepts drag-drop PDF, ErrorBanner shows/hides errors, useJobPolling hook polls and auto-stops on terminal states. TypeScript compiles.</done>
</task>

<task type="auto">
  <name>Task 2: Upload, Processing, and Results pages</name>
  <files>
    frontend/src/pages/UploadPage.tsx
    frontend/src/pages/ProcessingPage.tsx
    frontend/src/pages/ResultsPage.tsx
    frontend/src/components/Layout.tsx
  </files>
  <action>
    1. **Build `frontend/src/pages/UploadPage.tsx`** — the landing page:
       - State: `file: File | null`, `prompt: string`, `isSubmitting: boolean`, `error: string | null`
       - Layout (max-w-2xl mx-auto centered card):
         a. Heading: "Upload Floorplan" (text-2xl font-bold)
         b. Subheading: "Upload a PDF floorplan and describe the space to get a priced Bill of Materials" (text-gray-500)
         c. `<FileDropZone>` component for PDF selection
         d. Prompt textarea: `<textarea>` with placeholder "Describe the space (e.g., '6-room dental clinic with reception, 3 operatories, sterilization room, X-ray room, and staff break room')"
            - 4 rows, full width, border border-gray-300, rounded-lg, p-3
            - Label: "Space Description"
         e. Submit button: "Generate BOM" — bg-blue-600 text-white px-6 py-3 rounded-lg font-medium
            - Disabled when: no file selected OR prompt is empty OR isSubmitting
            - While submitting: show "Processing..." text and disable button
         f. `<ErrorBanner>` component at top for API errors
       - On submit:
         a. Set `isSubmitting = true`, clear error
         b. Call `uploadAndProcess(file, prompt)` from api/client
         c. On success: `navigate(`/processing/${response.job_id}`)` using react-router `useNavigate()`
         d. On error: Set error message from caught Error, set `isSubmitting = false`

    2. **Build `frontend/src/pages/ProcessingPage.tsx`** — job polling page:
       - Read `jobId` from `useParams()`, parse to number
       - If `jobId` is invalid/NaN, show error and link back to "/"
       - Use `useJobPolling(jobId)` hook
       - Layout (max-w-lg mx-auto centered):
         a. Animated spinner (CSS `animate-spin` on a circular SVG) — show while polling
         b. Status text based on `status`:
            - "queued": "Your floorplan is in the queue..."
            - "running": "Analyzing floorplan and generating BOM..."
            - "succeeded": "Complete! Redirecting..." (brief flash before redirect)
            - "failed": "Processing failed" + error message
         c. Progress indicator: show which stage (queued → running → done) with step dots or simple text
         d. Job ID shown in small muted text: "Job #{jobId}"
       - On status === "succeeded":
         - Extract `result_ref.result_id` (the bom_id) from `jobData`
         - `navigate(`/results/${bomId}`, { replace: true })` — replace so back button goes to upload, not processing
         - If `result_ref` or `result_id` is missing, show error: "Job completed but no results found"
       - On status === "failed":
         - Show `jobData.error_message` in an ErrorBanner
         - Show a "Try Again" button that navigates back to "/"

    3. **Build `frontend/src/pages/ResultsPage.tsx`** — BOM results display:
       - Read `bomId` from `useParams()`, parse to number
       - Use `useEffect` to call `getResults(bomId)` on mount
       - State: `result: BOMResultResponse | null`, `loading: boolean`, `error: string | null`
       - Layout (max-w-4xl mx-auto):
         a. Loading state: show skeleton/spinner while fetching
         b. Error state: show ErrorBanner with "Back to Upload" link
         c. Success state:
            - Header card with: prompt text (what the user described), room count, total area (sqm), grand total (₹ formatted with Indian numbering — use `toLocaleString("en-IN")`)
            - Summary stats row: 3 stat cards side-by-side:
              - "Rooms": `bom_data.room_count`
              - "Total Area": `bom_data.total_area_sqm.toFixed(1)` sqm
              - "Estimated Cost": ₹ `total_cost_inr.toLocaleString("en-IN")`
            - BOM line items table (basic HTML table with Tailwind — Phase 4 will make this editable):
              - Columns: Material, Category, Qty, Unit, Rate (₹), Amount (₹), Room
              - Rows from `bom_data.line_items`
              - Format currency with ₹ and Indian locale
              - Footer row with "Grand Total" spanning to Amount column
            - "Upload Another" button that links back to "/"
            - Placeholder text: "Layout visualization coming in next update" (Phase 4 delivers this)

    4. **Update `frontend/src/components/Layout.tsx`** if needed:
       - Add a "New Upload" link/button in the header that navigates to "/" — only show when NOT on the upload page (check `useLocation().pathname !== "/"`)
       - Keep the header clean and minimal
  </action>
  <verify>
    <automated>cd frontend && npx tsc --noEmit && npm run build && echo "BUILD OK"</automated>
  </verify>
  <done>All three pages render correctly. Upload accepts PDF + prompt and calls API. Processing polls and redirects. Results displays BOM data. TypeScript compiles. Build succeeds.</done>
</task>

<task type="checkpoint:human-verify" gate="blocking">
  <name>Task 3: Verify end-to-end upload → process → results flow</name>
  <files>
    frontend/src/pages/UploadPage.tsx
    frontend/src/pages/ProcessingPage.tsx
    frontend/src/pages/ResultsPage.tsx
  </files>
  <action>
    Human verification of the complete upload → processing → results flow built in Tasks 1-2.

    **What was built:**
    1. Upload page with drag-drop PDF zone and prompt textarea
    2. Processing page with live job polling and auto-redirect
    3. Results page with BOM summary table and formatted costs
    4. Error handling for all failure modes (no file, bad format, server errors, job failures)

    **How to verify:**
    1. Start the backend: `python app/main.py` (or `flask run`)
    2. Start Redis + RQ worker (for async jobs to process)
    3. Start the frontend: `cd frontend && npm run dev`
    4. Open http://localhost:5173 in a browser

    **Test the upload flow:**
    - Verify the upload page shows a drag-drop zone and prompt textarea
    - Try submitting without a file → should show error
    - Try submitting without a prompt → button should be disabled
    - Upload a PDF from `sample_pdfs/` and type a prompt like "dental clinic with 4 operatories"
    - Click "Generate BOM" → should navigate to /processing/:jobId

    **Test the processing flow:**
    - Verify the processing page shows a spinner and status text
    - Watch it poll (status should change from queued → running → succeeded)
    - On success, verify auto-redirect to /results/:bomId

    **Test the results flow:**
    - Verify results page shows prompt, room count, total area, grand total
    - Verify BOM line items table has materials with ₹ prices
    - Click "Upload Another" → should go back to upload page

    **Test error handling:**
    - Upload a non-PDF file → should show error
    - Navigate to /processing/99999 → should handle gracefully
    - Navigate to /results/99999 → should show "not found" error
  </action>
  <verify>User confirms the full flow works visually and functionally</verify>
  <done>User types "approved" — all pages render correctly, upload → process → results flow works end-to-end, errors display cleanly</done>
</task>

</tasks>

<verification>
1. `cd frontend && npm run build` — production build succeeds
2. `cd frontend && npx tsc --noEmit` — zero TypeScript errors
3. `pytest tests/ -q` — all backend tests pass including new /process endpoint tests
4. Manual: Upload PDF → Processing → Results flow works end-to-end
5. Manual: Error states show user-friendly messages (no raw traces)
</verification>

<success_criteria>
- Opening localhost:5173 shows a clean upload page with drag-drop PDF zone and text prompt input
- Submitting an upload hits the backend API and shows a processing state with live status polling
- When the job completes, the user lands on a results page showing BOM data with formatted costs
- API errors (no file, bad format, server error) show clear user-facing messages
- All Phase 3 success criteria from ROADMAP.md are met
</success_criteria>

<output>
After completion, create `.planning/phases/03-react-spa-shell-upload/03-react-spa-shell-upload-02-SUMMARY.md`
</output>
