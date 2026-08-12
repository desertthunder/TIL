---
title: Cancellation Signals
description: Cooperative cancellation for asynchronous and long-running work
tags:
  - concurrency
  - asynchronous-programming
  - software-design
---

A cancellation signal is a request for work to stop that does not forcibly terminate the
work. The operation must observe the request, release resources, and return or throw at a
safe point. Calling a cancel function also does not wait for the operation to finish.[^1]

Most cancellation APIs separate authority from observation. The caller owns a controller
or cancel function and callees receive a read-only signal or token. This lets a request
pass down a call tree without giving every function the power to cancel unrelated work.

Go's `Context`[^1], .NET's `CancellationToken`[^2], and the web's `AbortSignal`[^3] all
follow this broad pattern.

A good cancellation signal is:

- **sticky:** once cancelled, it stays cancelled
- **composable:** a timeout, parent request, or explicit action can share one path
- **propagated:** every child operation receives the signal
- **observable:** code can poll it, await it, or register a callback

Consumers must check both before starting and while blocked or looping. A JavaScript API,
for example, should reject immediately if its `AbortSignal` is already aborted, then listen
for its `abort` event while work is pending.[^3] CPU-bound work needs explicit checkpoints,
as an operation stuck in non-cancellable code cannot respond promptly.

Cancellation is also a result worth preserving. Returning the signal's reason distinguishes
a deadline or user abort from an I/O or programming failure. Cleanup belongs in `finally`,
`defer`, or the language's equivalent, because cancellation can arrive at any suspension or
checkpoint.

[^1]: Go, [`context` package](https://pkg.go.dev/context).

[^2]: Microsoft, [Cancellation in managed threads](https://learn.microsoft.com/en-us/dotnet/standard/threading/cancellation-in-managed-threads).

[^3]: MDN, [`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal).
