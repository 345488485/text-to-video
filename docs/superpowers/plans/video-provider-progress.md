# Video provider configuration

User request: fixed platform endpoints, API-key-only setup for mainstream video providers.

- Backend owner: provider_backend; config persistence, fixed registry, adapters, API routes, task routing and mocked tests.
- Frontend owner: provider_frontend; settings route, key actions, capabilities-aware studio and build verification.
- Root: integration review, isolated browser acceptance and runtime handoff.
- First supported platforms: Volcengine Ark, Alibaba Wan Beijing, Runway, Luma. Official API references checked before implementation.
- Credentials must never be returned to browsers, stored in localStorage, or printed in tool output.
- Test calls must be read-only; all generation validation uses mocked upstream HTTP.
- Provider config persistence must derive from database directory or explicit path to isolate smoke tests from production.
- Maintain existing environment Ark configuration as fallback and existing default mock behavior in tests.
- Complete: four fixed-endpoint adapters, DPAPI credentials, local administration API, task ownership routing and capability-aware Vben configuration/studio pages.
- Review corrections: serialize paid submit/config mutation and adapter polling/cancellation; preserve terminal states; Wan2.6 requires audio; non-Ark quota estimates explicitly local accounting.
- Verification: 31 Python tests, 13 frontend tests, frontend typecheck and production build pass. Isolated browser save/redaction/delete/mobile checks pass with synthetic key; no real paid calls.
- Runtime: production restarted on 4173, original volcengine default preserved; isolated4174 test process closed.
