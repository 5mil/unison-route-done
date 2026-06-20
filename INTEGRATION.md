# Integration Guide

This guide covers integrating unison-route-done with heavier frameworks, lower-level HTTP servers, and external tools outside the Unison ecosystem.

---

## 1. Official @unison/routes

@unison/routes is Unison's high-level HTTP service library. It handles routing and path parsing. unison-route-done handles early-exit control flow inside the handler body. They complement each other.

```unison
lib.install @unison/routes
lib.install https://github.com/5mil/unison-route-done .Route

myHandler : Request ->{Route.Done, Route, Remote} unison.routes.HttpResponse
myHandler req =
  Route.guard.badRequest
    (Request.queryParam req "q" |> Optional.isNone)
    "Missing q param"

  q    = Route.require (Request.queryParam req "q") "Missing q"
  data = Remote.storage.get q
  ok.json (Json.encode data)

myService : unison.routes.HttpRequest ->{Exception, Remote} unison.routes.HttpResponse
myService =
  use unison.routes.Route
  Route.run <|>
  Route.noCapture GET (s "api" / s "data")   myHandler <|>
  Route.noCapture GET (s "api" / s "status") (ok.text "ok")

deploy : '{IO, Exception} ()
deploy =
  Cloud.main.local.serve do
    serviceHash = deployHttp !Environment.default myService
    ServiceName.assign (ServiceName.named "my-service") serviceHash
```

@unison/routes helpers return Result or Exception — they do not short-circuit. Use unison-route-done for early exits inside the handler body, and @unison/routes for routing and serialization.

---

## 2. Lower-level @unison/http

@unison/http exposes raw HttpRequest and HttpResponse types without routing combinators. You handle routing manually. unison-route-done gives you typed early exit so you do not need nested match chains or Abort threading.

```unison
lib.install @unison/http
lib.install https://github.com/5mil/unison-route-done .Route

handleRequest : http.HttpRequest ->{Route.Done, IO} http.HttpResponse
handleRequest req =
  Route.handle do
    match req.path with
      "/api/data?" ++ _ -> handleData req
      "/api/status"     -> Route.ok "ok"
      _                 -> Route.notFound "Not found"

handleData : http.HttpRequest ->{Route.Done, IO} http.HttpResponse
handleData req =
  Route.guard.badRequest
    (req.queryParams |> Map.get "q" |> Optional.isNone)
    "Missing q"
  Route.okJson "data"

server : '{IO} ()
server = Http.serve 8080 handleRequest
```

If @unison/http uses a different HttpResponse type, add a conversion wrapper:

```unison
HttpResponse.toHttp : unison.Route.HttpResponse -> http.HttpResponse
HttpResponse.toHttp (HttpResponse s b _hs) =
  http.HttpResponse.make s b
```

---

## 3. External Integrations via IO

Unison does not have an FFI for invoking code in other languages directly. The preferred mechanism is network-based: expose your external service as an HTTP endpoint or socket, and call it from Unison via the IO ability.

### HTTP client

```unison
callExternalApi : Text ->{Route.Done, IO} Text
callExternalApi userId =
  Route.guard.unauthorized (userId == "") "Missing userId"
  response = IO.http.get ("https://external.com/user/" ++ userId)
  Route.guard.notFound (response.status != 200) "User not found"
  response.body

myHandler req =
  Route.handle do
    userId = Route.require (req.queryParams |> Map.get "id") "Missing id"
    data   = callExternalApi userId
    Route.okJson data
```

### Socket

```unison
connectToSocket : Text -> Text ->{Route.Done, IO} Socket
connectToSocket host port =
  Route.guard.badRequest (host == "") "Missing host"
  Route.guard.badRequest (port == "") "Missing port"
  socket = IO.socket-connect host port
  Route.guard.notFound (socket == None) "Connection failed"
  socket
```

### Webhook

```unison
sendWebhook : Text -> Json.Value ->{Route.Done, IO} ()
sendWebhook url payload =
  Route.guard.badRequest (url == "")           "Missing webhook URL"
  Route.guard.badRequest (Json.isEmpty payload) "Empty payload"
  body     = Json.encode payload
  response = IO.http.post url body (Map.fromList [("Content-Type", "application/json")])
  Route.guard.conflict (response.status != 200) "Webhook delivery failed"
```

---

## 4. Grim Integration

Grim uses Grim.Handlers.Local for self-hosted UCM deployment and Grim.Handlers.Cloud for Unison Cloud. Wire Route.handle at the dispatch boundary in each handler stack.

```unison
-- Grim/Handlers/Local.u

import Route

dispatch : Request ->{Http, Auth, Knowledge, Vault, IO} HttpResponse
dispatch req =
  Route.handle do routeHandler req

routeHandler : Request ->{Route.Done, Auth, Knowledge, Vault} HttpResponse
routeHandler req =
  Route.guard.badRequest (badQuery req) "Invalid query"
  Route.ok "Success"
```

Route.handle eliminates Route.Done at the dispatch boundary so the outer handler stack does not carry it. Every helper that may exit early carries {Route.Done} in its own type row. The pure logic in Grim.Math stays content-addressed and identical across both deployments.

---

## 5. Migration

```
when bad do respond 400 "msg"           ->  Route.guard.badRequest bad "msg"
match opt with Some a -> a | None -> Abort  ->  Route.require opt "msg"
Abort with toDefault ()                 ->  Route.handle do ...
nested match for validation             ->  sequential Route.guard.* lines
```

---

## 6. Performance

Route.Done is an algebraic ability. The overhead is one continuation capture and resume — no reflection, no exception stack, no dynamic dispatch. Early-exit handlers are faster than equivalent nested match chains because they do not build intermediate result types.

---

## 7. Debugging

`{Route.Done} r does not match {e} ()` — you used `when` where you need `Route.guard.*`, or the function is missing `{Route.Done}` in its type row.

Code continues past Route.ok — you forgot `Route.handle do` at the dispatch site.

HttpResponse where Json.Value expected — use `Route.okJson` for JSON bodies, or add a type conversion wrapper.
