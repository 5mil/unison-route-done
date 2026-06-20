# Route.Done: Typed Early Exit for Unison Route Handlers

## The problem

Unison route handlers are plain functions. A handler that calls a response helper — whether that is a function that writes an HTTP response, sends a message, or returns a value — does not automatically stop executing. The function continues running past the response call unless the programmer structures the code to prevent it.

The typical workaround is the `Abort` ability from Unison base. `Abort` lets a computation exit early by raising an abort signal, which a top-level handler catches with `toDefault`. This works, but it has three meaningful costs:

1. **It loses the response value.** `Abort` carries no payload. The handler that catches it receives `()` — the actual HTTP response that triggered the abort is gone. The caller has to construct a fallback response from nothing, which means the failure reason travels out-of-band.

2. **It does not communicate intent.** An aborted computation looks the same whether it failed due to bad input, a missing record, an auth violation, or an internal panic. Callers have to infer from context what the abort meant.

3. **The wiring is mechanical and leaky.** Every function that may abort must declare `{Abort}` in its type. That is correct and unavoidable — effect rows always propagate — but since `Abort` is a generic abort with no payload, there is no type-level indication of *why* the function might abort or *what it sends back*.

## The solution

`Route.Done` is a single-operation algebraic ability:

```unison
structural ability Route.Done where
  done : response ->{Route.Done} r
```

It carries the committed response value. When `done` is called, the continuation is discarded, and the response travels directly to the `Route.handle` boundary where it is returned to the caller. Nothing is lost.

The return type `r` is universally polymorphic. This is the key to ergonomics: `done` typechecks in any return position — a match arm, an if-branch, a sequential line — regardless of the surrounding expected type. The typechecker accepts it everywhere because `r` unifies with any type.

## What changes and what does not

Effect row propagation does not change. Any function that calls `done` — directly or through a helper — must declare `{Route.Done}` in its effect row. This is not a cost of `Route.Done` specifically; it is Unison's ability system working correctly. `Abort` requires the same discipline.

What changes is everything else:

- The response value is carried to the handler intact, as a typed `HttpResponse`.
- The handler constructs the HTTP response at the `Route.handle` boundary from a real value, not a default.
- The effect name `Route.Done` communicates a specific intent: this function may commit a route response and exit. That is different from a generic abort.
- Guard helpers return `()` on the happy path and only exit on failure, making top-to-bottom sequential handler code natural without wrapping.

## Handler structure

```
Route.handle                    -- eliminates Route.Done, returns HttpResponse
  do
    Route.guard.* ...           -- returns () or exits with response
    Route.require ...           -- unwraps Optional or exits with response
    -- business logic
    Route.ok "result"           -- commits response, exits
```

Each line after a guard is only reached if all previous guards passed. The structure maps directly to the logical flow of the handler.

## Relationship to other approaches

**Nested match chains:** Correct but noisy. Each optional field or validation check adds a level of nesting. A handler with five validation steps has five levels of nesting before it reaches the business logic.

**`Abort` with `toDefault`:** Correct. Works. Loses the response value and communicates no intent. Usable as a blunt instrument but not as a structured response layer.

**`Either` threading:** Correct and explicit but requires every step to be wrapped in `Either` and every caller to pattern-match on the result. The noise is proportional to the number of fallible steps.

**`Route.Done`:** Correct. Carries the response value. Communicates intent. Requires the same effect-row discipline as `Abort` but nothing more.

## Correctness note on `when`

Unison's `when` combinator has type `Boolean -> '{e} () -> {e} ()`. Response helpers have return type `{Route.Done} r`, not `{Route.Done} ()`. They do not typecheck inside `when`. This library provides `Route.guard.*` combinators explicitly to cover this use case without the type mismatch.

## Conclusion

`Route.Done` is a small, precise tool. It does one thing: carry a committed response value out of a handler computation at a well-typed boundary. It does not attempt to be a full HTTP framework, a middleware system, or a routing table. It is a control-flow primitive for the common pattern of "respond and stop" in Unison route handler functions.
