---
id: go-selectors
title: Go SDK Selectors
sidebar_label: Selectors
---

## Overview

In Go, the `select` statement lets a goroutine wait on multiple communication operations.
A `select` **blocks until one of its cases can run**, then it executes that case.
It chooses one at random if multiple are ready.

However, a normal Go select statement can not be used inside of Workflows directly because of the random nature.
Temporal's Go SDK `Selector`s are similar and act as a replacement.
They can block on sending and receiving from Channels but as a bonus can listen on Future deferred work.
Usage of Selectors to defer and process work (in place of Go's `select`) are necessary in order to ensure deterministic Workflow code execution (though using `select` in Activity code is fine).

## Full API Example

The API is sufficiently different from `select` that it bears documenting:

```go
func SampleWorkflow(ctx workflow.Context) error {
	// standard Workflow setup code omitted...

	// API Example: declare a new selector
	selector := workflow.NewSelector(ctx)

	// API Example: defer code execution until the Future that represents Activity result is ready
	work := workflow.ExecuteActivity(ctx, ExampleActivity)
	selector.AddFuture(work, func(f workflow.Future) {
		// deferred code omitted...
	})

	// more parallel timers and activities initiated...

	// API Example: receive information from a Channel
	var signalVal string
	channel := workflow.GetSignalChannel(ctx, channelName)
	selector.AddReceive(channel, func(c workflow.ReceiveChannel, more bool) {
		// matching on the channel doesn't consume the message.
	 	// So it has to be explicitly consumed here
		c.Receive(ctx, &signalVal)
		// do something with received information
	})

	// API Example: block until the next Future is ready to run
	// important! none of the deferred code runs until you call selector.Select
	selector.Select(ctx)

	// Todo: document selector.HasPending
}
```

## Using Selectors with Futures

You usually add `Future`s after `Activities`:

```go
	// API Example: defer code execution until after an activity is done
	work := workflow.ExecuteActivity(ctx, ExampleActivity)
	selector.AddFuture(work, func(f workflow.Future) {
		// deferred code omitted...
	})
```

`selector.Select(ctx)` is the primary mechanism which blocks on and executes `Future` work.
It is intentionally flexible; you may call it conditionally or multiple times:

```go
	// API Example: blocking conditionally
  if somecondition != nil {
		selector.Select(ctx)
  }

	// API Example: popping off all remaining Futures
  for i := 0; i < len(someArray); i++ {
		selector.Select(ctx) // this will wait for one branch
		// you can interrupt execution here
	}
```

A Future matches only once per Selector instance even if Select is called multiple times.
If multiple items are available, the order of matching is not defined.

## Using Selectors with Channels

`selector.AddReceive(channel, func(c workflow.ReceiveChannel, more bool) {})` is the primary mechanism which receives messages from `Channels`.

```go
	// API Example: receive information from a Channel
	var signalVal string
	channel := workflow.GetSignalChannel(ctx, channelName)
	selector.AddReceive(channel, func(c workflow.ReceiveChannel, more bool) {
		c.Receive(ctx, &signalVal)
		// do something with received information
	})
```

Merely matching on the channel doesn't consume the message; it has to be explicitly consumed with a `c.Receive(ctx, &signalVal)` call.

## Querying Selector State

You can use the `selector.HasPending` API to ensure that signals are not lost when a Workflow is closed (e.g. by `ContinueAsNew`).

## Learn More

Usage of Selectors is best learned by example:

- Setting up a race condition between an Activity and a Timer, and conditionally execute ([Timer example](https://github.com/temporalio/samples-go/blob/14980b3792cc3a8447318fefe9a73fe0a580d4b9/timer/workflow.go))
- Receiving information in a Channel ([Mutex example](https://github.com/temporalio/samples-go/blob/14980b3792cc3a8447318fefe9a73fe0a580d4b9/mutex/mutex_workflow.go))
- Looping through a list of work and scheduling them all in parallel ([DSL example](https://github.com/temporalio/samples-go/blob/14980b3792cc3a8447318fefe9a73fe0a580d4b9/dsl/workflow.go))
- Executing activities in parallel, pick the first result, cancel remainder ([Pick First example](https://github.com/temporalio/samples-go/blob/14980b3792cc3a8447318fefe9a73fe0a580d4b9/pickfirst/pickfirst_workflow.go))
