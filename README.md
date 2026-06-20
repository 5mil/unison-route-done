# unison-route-done

A small Unison library that gives route handlers a typed early-exit primitive.

Once you call `Route.ok`, `Route.badRequest`, or any other response helper, the handler **stops executing immediately**. No threading `Abort` through every function. No accidental fallthrough past a response. No nebulous effect that bleeds into unrelated code.

```unison
myHandler : Request ->{Route.Done, Knowledge} HttpResponse
myHandler req =
  Route.guard.badRequest
    (Request.queryParam req "q" |> Optional.isNone)
    "Missing required param: q"

  q = Route.require (Request.queryParam req "q") "Missing q"
  Route.ok (Knowledge.search q)
```

---

## Why this exists

In a Unison HTTP handler, nothing stops you from writing:

```unison
when badInput do respond 400 "Bad input"  -- does NOT exit
respond 200 "Success"                      -- also executes
```

The handler keeps executing past the first response because there is nothing to stop it. The common workaround is to thread an `Abort` ability through the entire call graph, which is verbose, leaks into helpers, and loses the response value on exit.

`unison-route-done` solves this with a single `Route.Done` algebraic ability. Calling `done` carries the committed `HttpResponse` value out of the handler immediately. The continuation is discarded. The response is returned to the caller.

---

## Core API

### Ability

```unison
structural ability Route.Done where
  done : response ->{Route.Done} r
```

`done` is polymorphic in its return type `r`, meaning it typechecks in any position — a match arm, an if-branch, a sequential line — regardless of the surrounding expected type.

### Runner

```unison
Route.handle : '{Route.Done, e} response ->{e} response
```

Executes a handler thunk. If `done` is called anywhere inside, execution stops and the committed response is returned. If the thunk runs to completion, its return value is used. The `e` row passes every other ability through untouched.

`Route.handle` is the only place `Route.Done` is consumed. Every function below it in the call stack simply carries `{Route.Done}` in its type row — no special wiring required.

### Response helpers

All of these commit a response and exit immediately:

| Helper | Status |
|---|---|
| `Route.ok text` | 200 |
| `Route.badRequest text` | 400 |
| `Route.unauthorized text` | 401 |
| `Route.forbidden text` | 403 |
| `Route.notFound text` | 404 |
| `Route.conflict text` | 409 |
| `Route.serverError text` | 500 |
| `Route.okJson text` | 200 + `Content-Type: application/json` |
| `Route.respond status text` | any status |

### Guards

Guards return `()` on the happy path so they compose sequentially:

```unison
Route.guard.badRequest  : Boolean -> Text ->{Route.Done} ()
Route.guard.unauthorized: Boolean -> Text ->{Route.Done} ()
Route.guard.forbidden   : Boolean -> Text ->{Route.Done} ()
Route.guard.notFound    : Boolean -> Text ->{Route.Done} ()
Route.guard.conflict    : Boolean -> Text ->{Route.Done} ()
Route.guard             : Boolean -> Nat -> Text ->{Route.Done} ()
```

### Optional unwrappers

```unison
Route.require      : Optional a -> Text ->{Route.Done} a  -- exits 400 on None
Route.requireFound : Optional a -> Text ->{Route.Done} a  -- exits 404 on None
```

---

## Installation

This library is a Unison source file. Add `Route.u` to your project and `pull` it into your UCM namespace:

```
pull https://github.com/5mil/unison-route-done:.Route .myproject.Route
```

---

## Relationship to `Abort`

| | `Abort` | `Route.Done` |
|---|---|---|
| Carries response value | No — collapses to `()` | Yes — typed `HttpResponse` |
| Effect row propagation | Same — bleeds upward | Same — bleeds upward |
| Handler recovers full response | No | Yes |
| Works in any return position | No — type mismatch risk | Yes — `r` is polymorphic |
| `when` compatible without wrapper | No | No — use `guard` instead |

Both `Abort` and `Route.Done` require the effect to appear in every function signature on the call stack between the call site and the handler. The difference is that `Route.Done` carries the actual response value all the way out, whereas `Abort` discards it.

---

## Effect row discipline

Anytime a function calls `Route.done`, `Route.ok`, `Route.require`, or any guard, it must declare `{Route.Done}` in its effect row:

```unison
-- correct
myHelper : Text ->{Route.Done, Knowledge} Data

-- will not compile
myHelper : Text ->{Knowledge} Data
```

This is not a bug — it is the type system correctly enforcing that the early exit is visible at every level of the call stack. `Route.handle` eliminates the ability at the dispatch boundary.

---

## Examples

See the [`examples/`](./examples/) directory:

- `validation.u` — query parameter validation
- `auth_gate.u` — token verification and role enforcement
- `multi_step.u` — validate, persist, respond in sequence
- `shared_helpers.u` — reusable helpers that own their failure responses

---

## License

MIT
