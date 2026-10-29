# FrameFlow Vben 5 Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the legacy static frontend with a production-ready Vben Admin 5 Ant Design Vue application while preserving every existing FastAPI-backed FrameFlow workflow.

**Architecture:** Add the official Vben 5.7 monorepo under `frontend/`, customize its Ant Design Vue application, and isolate FrameFlow code in focused API, store, component, and view modules. Vite proxies `/api` during development; FastAPI serves the built SPA in production and preserves all current API routes.

**Tech Stack:** Vben Admin 5.7, Vue 3, TypeScript, Vite, Ant Design Vue, Pinia, Vue Router, Vitest, FastAPI, Python unittest

**Spec:** `docs/superpowers/specs/2026-10-29-vben-migration-design.md`

## Global Constraints

- Use Vben Admin 5.7 with the official `apps/web-antd` application.
- Use Node 24 and pnpm 11; do not introduce npm or yarn lockfiles.
- Preserve every current `/api/*` request and response field.
- Preserve the current FastAPI provider, SQLite database, and Seedance integration.
- Keep the low-cost default: 720p, Fast, 5 seconds, audio disabled.
- Keep legacy `public/` and the verified migration ZIP; do not delete them.
- Never place `.env`, API keys, SQLite files, build output, or backups in Git.
- Use Vben preferences for theme persistence and semantic CSS variables for brand colors.
- Every write operation must show loading, success, and actionable failure feedback.

---

## Planned File Structure

```text
frontend/
├─ apps/web-antd/
│  ├─ src/
│  │  ├─ api/frameflow/
│  │  │  ├─ client.ts              # JSON transport and error normalization
│  │  │  ├─ contracts.ts           # API request and response types
│  │  │  └─ index.ts               # Typed endpoint functions
│  │  ├─ components/frameflow/
│  │  │  ├─ MetricCard.vue
│  │  │  ├─ StudioComposer.vue
│  │  │  ├─ StudioPreview.vue
│  │  │  └─ WorkCard.vue
│  │  ├─ stores/frameflow/
│  │  │  ├─ account.ts
│  │  │  ├─ generation.ts
│  │  │  └─ works.ts
│  │  ├─ views/frameflow/
│  │  │  ├─ dashboard/index.vue
│  │  │  ├─ studio/index.vue
│  │  │  ├─ works/index.vue
│  │  │  ├─ billing/index.vue
│  │  │  ├─ templates/index.vue
│  │  │  └─ assets/index.vue
│  │  ├─ router/routes/modules/frameflow.ts
│  │  ├─ preferences.ts             # Vben preference and theme defaults
│  │  └─ styles/frameflow.css       # Brand tokens and studio layout
│  ├─ vite.config.mts                # `/api` development proxy
│  └─ .env.development
├─ package.json                      # Official Vben scripts
└─ pnpm-lock.yaml                    # Fixed dependency graph

backend/main.py                      # SPA static hosting and route fallback
tests/test_api.py                    # Production frontend hosting contract
docs/DEVELOPMENT.md                  # Two-process development and build steps
README.md                            # Vben architecture and run commands
```

---

### Task 1: Establish Recovery Baseline and Official Vben Scaffold

**Files:**
- Modify: `.gitignore`
- Create: `frontend/**` from official `vbenjs/vue-vben-admin` tag `v5.7.0`
- Verify: `backups/video_project-before-vben-20261029.zip`

**Interfaces:**
- Consumes: The verified pre-migration ZIP and the existing FastAPI app.
- Produces: A clean official Vben monorepo where `pnpm run build:antd` succeeds.

- [ ] **Step 1: Verify the recovery archive before changing the tree**

Run:

```powershell
Get-FileHash backups/video_project-before-vben-20261029.zip -Algorithm SHA256
```

Expected SHA-256:

```text
C658D491B9903A395188AE05F7865C871F8C803BA64E388651B43C29EFA60A7A
```

- [ ] **Step 2: Expand ignore rules for generated and private files**

Add exactly these entries to `.gitignore`:

```gitignore
backups/
frontend/node_modules/
frontend/**/node_modules/
frontend/**/dist/
frontend/.turbo/
frontend/**/.vite/
frontend/**/coverage/
```

- [ ] **Step 3: Initialize a local Git recovery baseline**

Run:

```powershell
git init
git add .
git commit -m "chore: preserve pre-vben application baseline"
```

Expected: `.env`, `data/*.db*`, `backups/`, `.venv/`, and logs do not appear in `git status --short`.

- [ ] **Step 4: Import the official Vben source without nested Git metadata**

Run in a temporary directory, then copy the checked-out files into `frontend/`:

```powershell
git clone --depth 1 --branch v5.7.0 https://github.com/vbenjs/vue-vben-admin.git $env:TEMP\frameflow-vben-5.7.0
robocopy $env:TEMP\frameflow-vben-5.7.0 frontend /E /XD .git
```

Expected: `frontend/apps/web-antd/package.json` and `frontend/pnpm-lock.yaml` exist; `frontend/.git` does not exist.

- [ ] **Step 5: Install the fixed dependency graph**

Run:

```powershell
cd frontend
pnpm install --frozen-lockfile
```

Expected: install completes without changing `pnpm-lock.yaml`.

- [ ] **Step 6: Verify the untouched Vben Ant Design application**

Run:

```powershell
pnpm run build:antd
```

Expected: exit code 0 and a Vben Ant Design `dist` directory.

- [ ] **Step 7: Commit the scaffold checkpoint**

```powershell
git add .gitignore frontend
git commit -m "chore: add vben 5 ant design frontend"
```

---

### Task 2: Build the Typed FrameFlow API Layer

**Files:**
- Create: `frontend/apps/web-antd/src/api/frameflow/contracts.ts`
- Create: `frontend/apps/web-antd/src/api/frameflow/client.ts`
- Create: `frontend/apps/web-antd/src/api/frameflow/index.ts`
- Create: `frontend/apps/web-antd/src/api/frameflow/client.test.ts`
- Modify: `frontend/apps/web-antd/vite.config.mts`

**Interfaces:**
- Consumes: Existing FastAPI JSON routes under `/api`.
- Produces: `frameflowRequest<T>()`, `frameflowApi`, `GenerationInput`, `VideoTask`, `Usage`, `Billing`, and `Work` types.

- [ ] **Step 1: Write failing transport tests**

Create `client.test.ts` with cases that assert JSON parsing, error normalization, and custom headers:

```ts
import { afterEach, describe, expect, it, vi } from 'vitest';

import { FrameFlowApiError, frameflowRequest } from './client';

describe('frameflowRequest', () => {
  afterEach(() => vi.restoreAllMocks());

  it('returns parsed JSON', async () => {
    vi.spyOn(globalThis, 'fetch').mockResolvedValue(
      new Response(JSON.stringify({ status: 'ok' }), { status: 200 }),
    );
    await expect(frameflowRequest('/api/health')).resolves.toEqual({ status: 'ok' });
  });

  it('normalizes backend errors', async () => {
    vi.spyOn(globalThis, 'fetch').mockResolvedValue(
      new Response(JSON.stringify({ error: { code: 'NO_CREDIT', message: '额度不足', retryable: false } }), { status: 402 }),
    );
    await expect(frameflowRequest('/api/generations')).rejects.toEqual(
      expect.objectContaining<Partial<FrameFlowApiError>>({ code: 'NO_CREDIT', message: '额度不足', retryable: false }),
    );
  });
});
```

- [ ] **Step 2: Run the focused test and verify failure**

Run from `frontend/`:

```powershell
pnpm exec vitest run apps/web-antd/src/api/frameflow/client.test.ts
```

Expected: FAIL because `client.ts` does not exist.

- [ ] **Step 3: Define exact API contracts**

`contracts.ts` must export literal unions and interfaces matching Pydantic fields:

```ts
export type Ratio = '1:1' | '16:9' | '9:16';
export type Resolution = '1080p' | '480p' | '720p';
export type TaskStatus = 'cancelled' | 'failed' | 'generating' | 'post_processing' | 'queued' | 'succeeded';

export interface GenerationInput {
  duration: 5 | 8 | 10 | 15;
  generateAudio: boolean;
  prompt: string;
  quality: 'fast' | 'pro' | 'standard';
  quantity: 1;
  ratio: Ratio;
  resolution: Resolution;
}

export interface VideoTask {
  actionUrl: null | string;
  cancelledAt: null | number;
  completedAt: null | number;
  createdAt: number;
  errorCode: null | string;
  errorMessage: null | string;
  id: string;
  input: GenerationInput;
  progress: number;
  resultUrl: null | string;
  stage: string;
  status: TaskStatus;
}
```

Also define `CostEstimate`, `Usage`, `Plan`, `BillingRecord`, `BillingEvent`, `Billing`, and `Work` from responses currently rendered in `public/app.js` and returned by `backend/store.py`.

- [ ] **Step 4: Implement transport and typed endpoints**

`client.ts` must export:

```ts
export class FrameFlowApiError extends Error {
  constructor(
    message: string,
    public readonly code = 'HTTP_ERROR',
    public readonly retryable = false,
    public readonly status = 500,
  ) {
    super(message);
    this.name = 'FrameFlowApiError';
  }
}

export async function frameflowRequest<T>(path: string, init: RequestInit = {}): Promise<T> {
  const headers = new Headers(init.headers);
  if (init.body && !headers.has('Content-Type')) headers.set('Content-Type', 'application/json');
  const response = await fetch(path, { ...init, headers });
  const payload = await response.json().catch(() => ({}));
  if (!response.ok) {
    const detail = payload?.error;
    throw new FrameFlowApiError(detail?.message ?? '请求失败', detail?.code, detail?.retryable, response.status);
  }
  return payload as T;
}
```

`index.ts` exposes one function per endpoint, including an `Idempotency-Key` header for generation creation.

- [ ] **Step 5: Add Vite API proxy**

Merge this server block into `vite.config.mts` without removing Vben plugins:

```ts
server: {
  proxy: {
    '/api': {
      changeOrigin: true,
      target: 'http://127.0.0.1:4173',
    },
  },
},
```

- [ ] **Step 6: Run tests and type checking**

```powershell
pnpm exec vitest run apps/web-antd/src/api/frameflow/client.test.ts
pnpm run check:type
```

Expected: both commands pass.

- [ ] **Step 7: Commit the API checkpoint**

```powershell
git add frontend/apps/web-antd
git commit -m "feat: add typed frameflow api client"
```

---

### Task 3: Implement Pinia Business Stores

**Files:**
- Create: `frontend/apps/web-antd/src/stores/frameflow/account.ts`
- Create: `frontend/apps/web-antd/src/stores/frameflow/generation.ts`
- Create: `frontend/apps/web-antd/src/stores/frameflow/works.ts`
- Create: `frontend/apps/web-antd/src/stores/frameflow/generation.test.ts`

**Interfaces:**
- Consumes: `frameflowApi` and the contracts from Task 2.
- Produces: `useFrameFlowAccountStore`, `useGenerationStore`, and `useWorksStore`.

- [ ] **Step 1: Write a failing generation lifecycle test**

Mock `frameflowApi` and assert that `submit()` stores the task, `pollOnce()` updates it, terminal states stop scheduling, and `cancel()` replaces current state.

```ts
it('submits and reaches a terminal task state', async () => {
  vi.mocked(frameflowApi.createGeneration).mockResolvedValue({ task: queuedTask, estimate });
  vi.mocked(frameflowApi.getGeneration).mockResolvedValue({ task: succeededTask });
  const store = useGenerationStore();
  await store.submit(input);
  expect(store.task?.status).toBe('queued');
  await store.pollOnce();
  expect(store.task?.status).toBe('succeeded');
  expect(store.isPolling).toBe(false);
});
```

- [ ] **Step 2: Run the test and verify failure**

```powershell
pnpm exec vitest run apps/web-antd/src/stores/frameflow/generation.test.ts
```

Expected: FAIL because the store does not exist.

- [ ] **Step 3: Implement account and works stores**

`account.ts` exposes `usage`, `billing`, `loading`, `loadUsage()`, `loadBilling()`, and `changePlan(planId)`.

`works.ts` exposes `items`, `loading`, `load()`, and `remove(taskId)`; successful deletion removes the item locally after the server confirms it.

- [ ] **Step 4: Implement the generation store**

The store must expose:

```ts
interface GenerationState {
  estimate: CostEstimate | null;
  estimateLoading: boolean;
  isPolling: boolean;
  submitting: boolean;
  task: null | VideoTask;
}
```

Actions: `estimateCost(input)`, `enhance(input)`, `submit(input)`, `pollOnce()`, `startPolling()`, `stopPolling()`, `cancel()`, and `reset()`. Poll after 700 ms, then 900 ms; retry transient request failures after 2200 ms. Always clear timers on terminal state and `$dispose()`.

- [ ] **Step 5: Run store tests and type checks**

```powershell
pnpm exec vitest run apps/web-antd/src/stores/frameflow/generation.test.ts
pnpm run check:type
```

Expected: PASS.

- [ ] **Step 6: Commit the store checkpoint**

```powershell
git add frontend/apps/web-antd/src/stores
git commit -m "feat: add frameflow business stores"
```

---

### Task 4: Configure Vben Navigation, Branding, and Themes

**Files:**
- Create: `frontend/apps/web-antd/src/router/routes/modules/frameflow.ts`
- Create: `frontend/apps/web-antd/src/styles/frameflow.css`
- Modify: `frontend/apps/web-antd/src/preferences.ts`
- Modify: `frontend/apps/web-antd/src/app.ts`
- Modify: `frontend/apps/web-antd/.env`
- Remove from route registration: Vben demo routes that are not used by FrameFlow

**Interfaces:**
- Consumes: Vben route meta and preference APIs.
- Produces: Six navigable FrameFlow routes and three persistent brand themes.

- [ ] **Step 1: Add a failing route metadata test**

Assert that the exported route tree contains `/dashboard`, `/studio`, `/works`, `/billing`, `/templates`, and `/assets`, each with a Chinese title and icon.

- [ ] **Step 2: Implement the route module**

Use lazy page imports and stable route names:

```ts
const routes: RouteRecordRaw[] = [
  { path: '/', redirect: '/dashboard' },
  { component: () => import('#/views/frameflow/dashboard/index.vue'), meta: { icon: 'lucide:layout-dashboard', order: 1, title: '工作台' }, name: 'FrameFlowDashboard', path: '/dashboard' },
  { component: () => import('#/views/frameflow/studio/index.vue'), meta: { icon: 'lucide:clapperboard', order: 2, title: '视频创作' }, name: 'FrameFlowStudio', path: '/studio' },
  { component: () => import('#/views/frameflow/works/index.vue'), meta: { icon: 'lucide:library', order: 3, title: '我的作品' }, name: 'FrameFlowWorks', path: '/works' },
  { component: () => import('#/views/frameflow/templates/index.vue'), meta: { icon: 'lucide:sparkles', order: 4, title: '灵感模板' }, name: 'FrameFlowTemplates', path: '/templates' },
  { component: () => import('#/views/frameflow/assets/index.vue'), meta: { icon: 'lucide:images', order: 5, title: '品牌素材' }, name: 'FrameFlowAssets', path: '/assets' },
  { component: () => import('#/views/frameflow/billing/index.vue'), meta: { icon: 'lucide:wallet-cards', order: 6, title: '套餐与账单' }, name: 'FrameFlowBilling', path: '/billing' },
];
```

- [ ] **Step 3: Configure the FrameFlow application identity**

Set title and namespace to `FrameFlow`, disable demo login routing, and use the local application flow without pretending an authentication backend exists. The application must open on `/dashboard`.

- [ ] **Step 4: Define three semantic theme presets**

`frameflow.css` defines semantic variables for `data-frameflow-theme="obsidian"`, `cloud`, and `oriental`, including surface, muted surface, border, text, accent, success, warning, and danger colors. `preferences.ts` keeps Vben dark/light mode synchronized with the selected preset and persists the preset as `frameflow-theme`.

- [ ] **Step 5: Run route tests, type checking, and build**

```powershell
pnpm exec vitest run apps/web-antd/src/router/routes/modules/frameflow.test.ts
pnpm run check:type
pnpm run build:antd
```

Expected: all pass and the build contains FrameFlow route chunks.

- [ ] **Step 6: Commit navigation and theme checkpoint**

```powershell
git add frontend/apps/web-antd
git commit -m "feat: configure frameflow vben shell and themes"
```

---

### Task 5: Build the Dashboard

**Files:**
- Create: `frontend/apps/web-antd/src/components/frameflow/MetricCard.vue`
- Create: `frontend/apps/web-antd/src/views/frameflow/dashboard/index.vue`
- Create: `frontend/apps/web-antd/src/views/frameflow/dashboard/index.test.ts`

**Interfaces:**
- Consumes: account and works stores from Task 3.
- Produces: `/dashboard` with real usage, plan, service, and recent work data.

- [ ] **Step 1: Write a failing dashboard component test**

Mount with a test Pinia and assert the rendered labels `可用额度`, `本月创作额度`, `当前套餐`, and `最近创作`, plus a `开始创作` link to `/studio`.

- [ ] **Step 2: Implement accessible metric cards**

`MetricCard.vue` accepts `label`, `value`, `description`, `icon`, and `tone`; it uses semantic text and does not make an entire card clickable unless an explicit link is present.

- [ ] **Step 3: Implement dashboard data loading and states**

On mount, load usage, billing, works, and health in parallel. Use Ant Design Vue skeletons while loading, a retry result on failure, an accurate monthly progress bar, and at most six recent works.

- [ ] **Step 4: Run focused test, type check, and build**

```powershell
pnpm exec vitest run apps/web-antd/src/views/frameflow/dashboard/index.test.ts
pnpm run check:type
pnpm run build:antd
```

- [ ] **Step 5: Commit dashboard checkpoint**

```powershell
git add frontend/apps/web-antd/src/components frontend/apps/web-antd/src/views/frameflow/dashboard
git commit -m "feat: add frameflow dashboard"
```

---

### Task 6: Migrate the Video Creation Studio

**Files:**
- Create: `frontend/apps/web-antd/src/components/frameflow/StudioComposer.vue`
- Create: `frontend/apps/web-antd/src/components/frameflow/StudioPreview.vue`
- Create: `frontend/apps/web-antd/src/views/frameflow/studio/index.vue`
- Create: `frontend/apps/web-antd/src/views/frameflow/studio/index.test.ts`

**Interfaces:**
- Consumes: generation, account, and works stores.
- Produces: Complete `/studio` generation workflow.

- [ ] **Step 1: Write failing interaction tests**

Tests must cover:

```ts
it('uses the low-cost defaults', () => {
  expect(wrapper.get('[data-test="duration"]').text()).toContain('5 秒');
  expect(wrapper.get('[data-test="resolution"]').text()).toContain('720p');
  expect(wrapper.get<HTMLInputElement>('[data-test="audio"]').element.checked).toBe(false);
});

it('maps 1080p to standard quality before submit', async () => {
  await wrapper.get('[data-test="resolution-1080p"]').trigger('click');
  await wrapper.get('[data-test="submit"]').trigger('click');
  expect(mockSubmit).toHaveBeenCalledWith(expect.objectContaining({ quality: 'standard', resolution: '1080p' }));
});
```

Also test prompt validation, enhancement, estimate display, submit loading, cancel, failed task action, result video, and download link.

- [ ] **Step 2: Implement `StudioComposer.vue`**

Use Ant Design Vue controls and emit typed events. Keep local form state as:

```ts
const input = reactive<GenerationInput>({
  duration: 5,
  generateAudio: false,
  prompt: '',
  quality: 'fast',
  quantity: 1,
  ratio: '9:16',
  resolution: '720p',
});
```

Debounce estimates by 220 ms. Changing resolution to `1080p` sets `quality = 'standard'`; `480p` and `720p` set `quality = 'fast'`. Disable submit when prompt is shorter than four trimmed characters, submission is active, or estimated cost exceeds available credits.

- [ ] **Step 3: Implement `StudioPreview.vue`**

Render explicit empty, queued/generating/post-processing, succeeded, failed, and cancelled states. Use the task progress as the progress bar value, sanitize text through Vue bindings, use `controls` on result video, and provide download, retry, cancel, and variant events.

- [ ] **Step 4: Compose the studio page**

The page coordinates store actions and refreshes account and works after terminal success. It stops polling on route leave and calls `generationStore.reset()` only for an explicit new creation.

- [ ] **Step 5: Run interaction tests and build**

```powershell
pnpm exec vitest run apps/web-antd/src/views/frameflow/studio/index.test.ts
pnpm run check:type
pnpm run build:antd
```

- [ ] **Step 6: Commit studio checkpoint**

```powershell
git add frontend/apps/web-antd/src/components/frameflow frontend/apps/web-antd/src/views/frameflow/studio
git commit -m "feat: migrate video creation studio to vben"
```

---

### Task 7: Implement Works, Billing, Templates, and Assets Pages

**Files:**
- Create: `frontend/apps/web-antd/src/components/frameflow/WorkCard.vue`
- Create: `frontend/apps/web-antd/src/views/frameflow/works/index.vue`
- Create: `frontend/apps/web-antd/src/views/frameflow/billing/index.vue`
- Create: `frontend/apps/web-antd/src/views/frameflow/templates/index.vue`
- Create: `frontend/apps/web-antd/src/views/frameflow/assets/index.vue`
- Create: focused `*.test.ts` files beside each page

**Interfaces:**
- Consumes: account and works stores, router query parameters, existing endpoint contracts.
- Produces: Four fully usable navigation destinations with no dead buttons.

- [ ] **Step 1: Write failing page behavior tests**

Cover these exact behaviors:

- Works: empty state, video preview, download, first-click confirmation plus confirmed delete.
- Billing: current plan disabled, other plans switch through store, records and events render.
- Templates: choosing a template routes to `{ path: '/studio', query: { prompt } }`.
- Assets: reject non-image or files over 10 MB; show object URL preview for a valid image; revoke the old object URL on replacement and unmount.

- [ ] **Step 2: Implement the works page**

Use responsive cards, not a dense administrative table. Every work shows title, ratio, duration, resolution, creation date, status, preview, download, and delete. Use `Modal.confirm` for deletion.

- [ ] **Step 3: Implement billing**

Use plan cards and two record sections. Display credits as platform credits, never as yuan. Keep the external supplier-cost link clearly labeled as a separate cost reference.

- [ ] **Step 4: Implement templates**

Move the three existing prompts into a typed local template array with categories and descriptions. A template selection navigates to the studio; the studio reads `route.query.prompt` once and fills the form.

- [ ] **Step 5: Implement session-only assets**

Use `Upload.Dragger` with `beforeUpload={() => false}` and local preview only. Show the exact copy `素材仅保存在当前浏览器会话，尚未上传云端`.

- [ ] **Step 6: Run page tests, type check, and build**

```powershell
pnpm exec vitest run apps/web-antd/src/views/frameflow
pnpm run check:type
pnpm run build:antd
```

- [ ] **Step 7: Commit business page checkpoint**

```powershell
git add frontend/apps/web-antd/src
git commit -m "feat: add frameflow works billing templates and assets"
```

---

### Task 8: Serve the Built Vben SPA from FastAPI

**Files:**
- Modify: `backend/main.py`
- Modify: `tests/test_api.py`
- Modify: `.env.example`

**Interfaces:**
- Consumes: Vben production build directory.
- Produces: Same-origin production application with SPA route fallback and untouched API behavior.

- [ ] **Step 1: Write failing production hosting tests**

Update `test_frontend_and_health_are_served_by_python` and add direct-route coverage:

```python
def test_spa_routes_fall_back_to_vben_index(self) -> None:
    for path in ("/", "/dashboard", "/studio", "/works", "/billing"):
        response = self.client.get(path)
        self.assertEqual(response.status_code, 200)
        self.assertIn("FrameFlow", response.text)

def test_missing_api_route_never_falls_back_to_spa(self) -> None:
    response = self.client.get("/api/does-not-exist")
    self.assertEqual(response.status_code, 404)
    self.assertEqual(response.headers["content-type"].split(";")[0], "application/json")
```

- [ ] **Step 2: Run the focused backend tests and verify the new route test fails**

```powershell
.venv\Scripts\python.exe -m unittest tests.test_api.ApiContractTests.test_spa_routes_fall_back_to_vben_index -v
```

Expected: FAIL because the current static mount has no Vue history fallback.

- [ ] **Step 3: Implement explicit asset serving and SPA fallback**

Resolve the frontend directory in this order:

1. `FRAMEFLOW_FRONTEND_DIR` environment variable when set.
2. `frontend/apps/web-antd/dist` when it contains `index.html`.
3. Legacy `public/` as development fallback.

Mount Vben asset directories before the catch-all route. Add `GET /{full_path:path}` after every API route; reject `full_path.startswith('api/')` with JSON 404 and otherwise return the chosen `index.html`. Do not use a root `StaticFiles` mount that intercepts application routes.

- [ ] **Step 4: Build frontend and run all backend tests**

```powershell
cd frontend
pnpm run build:antd
cd ..
.venv\Scripts\python.exe -m unittest discover -s tests -v
```

Expected: frontend build and all Python tests pass.

- [ ] **Step 5: Commit production hosting checkpoint**

```powershell
git add backend/main.py tests/test_api.py .env.example
git commit -m "feat: serve vben spa from fastapi"
```

---

### Task 9: End-to-End Verification and Documentation

**Files:**
- Modify: `docs/DEVELOPMENT.md`
- Modify: `docs/API_CONFIGURATION.md`
- Modify: `README.md`
- Create: `frontend/apps/web-antd/e2e/frameflow.spec.ts` if Vben's Playwright setup is available; otherwise create an equivalent app-level browser test in the repository's supported E2E directory.

**Interfaces:**
- Consumes: Complete Vben build and running FastAPI service.
- Produces: Reproducible run instructions and verified browser behavior.

- [ ] **Step 1: Document exact development commands**

Document two terminals:

```powershell
# Terminal 1
.venv\Scripts\python.exe run.py

# Terminal 2
cd frontend
pnpm run dev:antd
```

Document the Vite URL emitted by the command and production commands:

```powershell
cd frontend
pnpm run build:antd
cd ..
.venv\Scripts\python.exe run.py
```

- [ ] **Step 2: Add browser acceptance coverage**

The E2E test must navigate every sidebar route; verify the dashboard metrics; verify theme persistence after reload; create a mock-provider generation; observe a terminal result; open, download, and delete the work; and switch a plan. Use the mock provider in automated tests so the test cannot create a paid supplier task.

- [ ] **Step 3: Run the full automated verification suite**

```powershell
cd frontend
pnpm run check:type
pnpm run test:unit
pnpm run build:antd
cd ..
.venv\Scripts\python.exe -m compileall -q backend tests run.py
.venv\Scripts\python.exe -m unittest discover -s tests -v
```

Expected: every command exits 0.

- [ ] **Step 4: Run browser verification against the local app**

Verify at desktop 1440×900 and mobile 390×844:

- No horizontal overflow.
- Every sidebar item opens its page.
- All buttons have visible hover/focus/disabled states.
- The studio textarea, segmented parameters, audio switch, estimate, submit, cancel, preview, and download work.
- Dashboard, usage, billing, and works data match API responses.
- Direct reload on `/studio`, `/works`, and `/billing` returns the SPA.
- Browser console contains no uncaught error.

- [ ] **Step 5: Inspect secrets and generated artifacts before final commit**

```powershell
git status --short
git grep -n "ARK_API_KEY=" -- ':!docs/**'
```

Expected: no `.env`, database, API key, `node_modules`, or `dist` file is tracked.

- [ ] **Step 6: Commit documentation and acceptance tests**

```powershell
git add README.md docs frontend/apps/web-antd
git commit -m "docs: document and verify vben migration"
```

- [ ] **Step 7: Produce the final verification report**

Report the Vben version, changed routes, exact commands run, unit/backend/E2E counts, browser sizes checked, local URL, known non-goals, and the backup path. Do not claim success for any check that was not executed.
