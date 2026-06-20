# Usage Guide — unison-route-done

## The pattern in one sentence

Wrap your handler in `Route.handle`, call any `Route.*` response helper inside it, and execution stops at that point.

---

## 1. Wrapping a handler

Every handler must be called inside `Route.handle` for the early exit to work:

```unison
dispatch : Request ->{IO, Auth} HttpResponse
dispatch req =
  Route.handle do myHandler req
```

The `do` keyword thunks the handler so `Route.handle` controls when it runs.

---

## 2. Guard patterns

Guards are the primary control-flow tool. They return `()` on success and exit the handler on failure:

```unison
myHandler : Request ->{Route.Done, Auth} HttpResponse
myHandler req =
  Route.guard.badRequest (Text.isEmpty (Request.path req)) "Empty path"
  Route.guard.unauthorized (not (Auth.verify req)) "Not authenticated"
  Route.ok "All checks passed"
```

Each guard line is a checkpoint. If it fails, the response is committed and no subsequent line runs.

---

## 3. Optional unwrapping

`Route.require` and `Route.requireFound` are the cleanest way to handle missing values:

```unison
myHandler req =
  userId = Route.require (Request.queryParam req "id") "Missing id param"
  entity = Route.requireFound (Database.lookup userId) "User not found"
  Route.ok entity.name
```

Compared to a manual match chain, this removes two levels of nesting for every optional field.

---

## 4. Reusable helpers

Helpers can own their own failure responses. They just carry `{Route.Done}` in their type:

```unison
requireAuth : Request ->{Route.Done, Auth} User
requireAuth req =
  token = Route.require (Request.header req "Authorization") "Missing token"
  Route.guard.unauthorized (not (Auth.verify token)) "Bad token"
  Auth.userFromToken token

-- Call it in any handler — no extra wiring needed
myHandler req =
  user = requireAuth req
  Route.ok ("Hello, " ++ user.name)
```

---

## 5. Ability row discipline

`Route.Done` must appear in the type of every function that calls a `Route.*` helper:

```unison
-- This compiles:
validate : Text ->{Route.Done} ()
validate t = Route.guard.badRequest (Text.isEmpty t) "Empty value"

-- This does NOT compile — missing {Route.Done}:
validate : Text -> ()
validate t = Route.guard.badRequest (Text.isEmpty t) "Empty value"
```

`Route.handle` eliminates the ability at the dispatch boundary, so the dispatcher itself does not need `{Route.Done}` in its type.

---

## 6. Why not `when`

`when` in Unison has type `Boolean -> '{e} () -> {e} ()`. The response helpers have return type `{Route.Done} r` (not `()`), so they do not typecheck inside `when`. Use `Route.guard.*` instead:

```unison
-- Does NOT typecheck:
when badInput do Route.badRequest "Bad input"

-- Correct:
Route.guard.badRequest badInput "Bad input"
```

---

## 7. JSON responses

For JSON APIs, use `Route.okJson` which sets the Content-Type header automatically:

```unison
myHandler req =
  data = Knowledge.serialize (Knowledge.getAll)
  Route.okJson data
```

For error responses with JSON bodies, use `Route.respond` with the status you need and include a serialised body manually, or extend the library with your own JSON-specific guard.

---

## 8. Full handler skeleton

```unison
myRoute : Request ->{Route.Done, Auth, Knowledge, Vault} HttpResponse
myRoute req =
  -- 1. Auth
  _user = requireAuth req

  -- 2. Input validation
  name = Route.require (Request.bodyField req "name") "Missing name"
  Route.guard.conflict (Knowledge.exists name) "Already exists"

  -- 3. Business logic
  entity = Knowledge.create name
  Vault.recordProvenance entity.id "created"

  -- 4. Response
  Route.ok ("Created: " ++ name)

myDispatch : Request ->{Auth, Knowledge, Vault, IO} HttpResponse
myDispatch req = Route.handle do myRoute req
```
