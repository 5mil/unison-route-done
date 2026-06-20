# unison-route-done — Project Log

## 2026-06-19 | Initial publish

**What was built:**

- `Route.u` — core library: `Route.Done` ability, `Route.handle` runner, `HttpResponse` type, response helpers (ok, badRequest, unauthorized, forbidden, notFound, conflict, serverError, okJson, respond), guard combinators (badRequest, unauthorized, forbidden, notFound, conflict, generic), Optional unwrappers (require, requireFound)
- `examples/validation.u` — query parameter validation pattern
- `examples/auth_gate.u` — token verification and role enforcement pattern
- `examples/multi_step.u` — validate / persist / respond in sequence
- `examples/shared_helpers.u` — reusable helpers that own their failure responses
- `README.md` — full API reference with comparison table and installation
- `USAGE.md` — canonical usage patterns for guards, optionals, helpers, JSON, and full handler skeleton
- `whitepaper.md` — design rationale: why Route.Done over Abort, effect row discipline, correctness note on `when`
- `LICENSE` — MIT

**Key design decisions:**

- `structural` (not `unique`) for the ability — avoids hash-mismatch on library upgrade
- Continuation `_k` explicitly bound and discarded in `Route.handle` — correct Unison handler syntax
- Guards return `()` not `r` — makes sequential composition typecheck correctly
- `Route.require` / `Route.requireFound` added as Optional unwrappers — most common pattern in practice
- `HttpResponse` defined in the library rather than imported — keeps the library self-contained
- `when` incompatibility documented explicitly in README, USAGE, and whitepaper

**Open questions:**
- `Route.json` guard variant (for structured error bodies)
- Streaming response support (depends on Unison runtime)
- Integration example directly against Grim.Handlers.Local and Grim.Handlers.Cloud
