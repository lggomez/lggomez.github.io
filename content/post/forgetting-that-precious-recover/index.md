{
    "title": "Forgetting that precious recover",
    "description": "What happens when you forget a recover on a spawned goroutine?",
    "summary": "What happens when you forget a recover on a spawned goroutine?",
    "tags": ["Golang", "Post-mortem", "Intermediate"],
    "date": "2021-01-02T01:32:06-03:00",
    "categories": ["Programming", "Backend", "Bug"],
    "type": "",
    "weight": 0,
    "publishdate": null,
    "expirydate": null,
    "aliases": null,
    "slug": "forgetting-that-precious-recover",
    "url": ""
}

___***This is a public post-mortem that was intended to be released on the engineering blog of MercadoLibre, but my departure (and the slowness of the editing) prevented that.***___

## The event 
On December 28, 2018, the year's last friday when our shipping calculator triggered an alert. This component is responsible of delivering the estimated delivery time along with the delivery options for a given article.
From our team, in our NewRelic monitors we saw the cause was due to a strong degradation of the API response times. I guess it was time to leave aside the panettone leftovers and start diagnosing the issue.
Our metrics pointed to a specific flow, **SLA Coverage Brazil** (carrier coverage by service level agreement). Under the hood one of its dependencies was the culprit: **service coverage.**

## The diagnosis 
So, right now, our monitors showed that:
* In service coverage, the apdex and the execution/runtime metrics were degraded by a 100%, at the same time on the whole instance pool, during the window of just a couple of minutes
* This service failure triggers retries, adding an extra 300K requests per minute (RPM) and (not so) gracefully becoming a cascading failure
In the meanwhile, we were manually rebooting instances while adding new ones to the pool in an attempt to boost the service health. Additionally, we start observing replaced instances (that is, an instances that dies due to a process error and gets replaced by a new one).

So, from this point, the following events unfold:
1. We identify test request starting before the event
2. We locate the client and stop said traffic
3. The instances slowly recover and the apdex goes green
Among the confusion this created (the traffic was just a couple dozen requests) we retraced our steps and found the real cause of the downtime, something we discarded on the beginning: the panics

After checking the service coverage logs, we had seen panics but quickly omitted them due to their frequency and the fact that they didn't seem related to our particular problem

## What happened? 
> “...panics do not propagate between goroutines, and must be handled independently.”

When observing the panic stacktraces, we found the same pattern:
1. The stack frames did not include any of our recovery mechanisms at all, so this wasn't being handled
2. The offending code came from a goroutine

As you can see in this [issue](https://github.com/golang/go/issues/20161), one design detail of golang that gets easily overlooked at first is that panics do not propagate between goroutines, and must be handled independently.

When a panic happens inside a go application process, and does not recover it, it finalizes in an anomalous way (exit code ≠ 0).
Our web application stack included a recovery middleware for all things happening in the entrant goroutine that gets covered by it, but any new goroutine must have its recovery mechanism defined explicitly.

## Implications (a.k.a panics for dummies) 
If a panic occurs on a gorountine without a recover(), the callee isn't going to handle it, regardless of its recovery stack frame count, and the application will die.
It is a known issue and a non-negotiable design change for go 1.x (fingers crossed for go 2.x)
Compare the following examples, this one has a propagated (unhandled panic):

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	Go(test)
	time.Sleep(time.Second)
	fmt.Printf("HELLO")
}

func test() {
	panic("PANIC")
}

func Go(f func()) {
	defer func() {
		if r := recover(); r != nil {
			fmt.Printf("RECOVERED ERROR %v", r)
		}
	}()
	go f()
}
```

And this one handles the panic. You may notice that this code is in need of some composability:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	Go(test)
	time.Sleep(time.Second)
	fmt.Printf("HELLO")
}

func test() {
	panic("PANIC")
}

func Go(f func()) {
	go func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Printf("RECOVERED ERROR %v", r)
			}
		}()
		f()
	}()
}
```

## Thinking some solutions 
A couple days after the incident, I gathered the team with some alternatives and let the votes decide which was the best one for our situation (checking and guarding a set of N apicalls across several repos).
There were 3 candidates:

### The obvious and hard 
Not generating panics inside of the spawned goroutines. I'd wanted to patent that but I had the feeling Rob Pike already did that.
This is the ideal way but in the real world is not always possible. Most of time the are third-party dependencies or I/O driven packages which include panics as their error signaling mechanism. Getting rid of that is beyond us.

### The idiomatic and ugly
How about adding this lil' gut to the beginning every new goroutine we spawn in code?

```go
defer func() {
	if r := recover(); r != nil {
		fmt.Printf("RECOVERED - %v\r\n", r)
	}
}()
```

It is going to work, with the disadvantage of breaking the DRY rule, becoming an extra point of failure for specialized handlers.
Remember what I said about composability? we can mitigate it with a handler like the following, as long as f is argument of another function that launches the goroutine or we ourselves launch it:

```go
func Wrap(f func()) func() {
	return func() {
			defer func() {
				if r := recover(); r != nil {
					fmt.Printf("RECOVERED - %v\r\n", r)
				}
			}()
		f()
	}
}
```

We'd use it the following way:

```go
func main() {
	go Wrap(test)()
	time.Sleep(time.Second)
	fmt.Println("HELLO")
}
func test() {
	panic("PANIC")
}
//…
```

This way, we centralize the panic handling. And last but not least, in case we want to return something from said handler, we communicate though channels:

```go
package main
import (
	"fmt"
	"time"
)

func main() {
	signal := make(chan bool)
	go WrapWithSignal(test, signal)()
	time.Sleep(time.Second)
	fmt.Println("HELLO")
	select {
	case sig := <-signal:
		if !sig {
			fmt.Println("Could not finish test execution")
		} else {
			fmt.Println("Finished test execution")
		}
	}
}

func test(signal chan bool) {
	panic("PANIC")
	signal <- true
}

func WrapWithSignal(f func(chan bool), signal chan bool) func() {
	return func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Printf("RECOVERED - %v\r\n", r)
				signal <- false
			}
		}()
		f(signal)
	}
}
```

This simple pattern is idiomatic but can accumulate code smells quickly if the code has added complexity or use cases, so use it accordingly

## Bonus track: Infrastructure 
In order to be resilient, a service must be able to respond quickly to failures. Throwing the entire instance and requesting a new one is far from an ideal solution (in this particular case, a simple reboot of the dying webserver with a watchdog script could've been orders of magnitude faster than the default workflow).

## Bonus track II: Spotting unrecovered panics 
This was a dump of our middleware stack (notice the `RecoveryWithWriter` being the recovery middleware):
 
```shell
vendor/[ommited]/gingonic/mlhandlers.Datadog.func1
vendor/github.com/newrelic/go-agent/_integrations/nrgin/v1.Middleware.func1
vendor/[ommited]/gingonic/mlhandlers.CommonAPiFilter.func1
vendor/[ommited]/gingonic/mlhandlers.RecoveryWithWriter.func1
context.AddContext [ommited]/src/api/context.AccessLog
controllers.(*RouteCoverageController).ValidateGet-fm
controllers.(*RouteCoverageController).Get-fm
```

And, the unhandled panic stacktrace on all of its glory:

```shell
goroutine 141 [running]:
[ommited]/src/api/services.(*CoverageService).GetCoverageServices(0xc4205ec1e0, 0x10e4e20, 0xc420073a70, 0xc420e8a41b, 0x3, 0xc420e8a41b, 0x3, 0x0, 0x0, 0x0, ...)
/services/coverage_service.go:35 +0x39
[ommited]/src/api/services.(*CoverageLocationService).GetRouteCoverage.func1(0xc400000008, 0xca6f90)
/services/coverage_locations.go:39 +0xe3
[ommited]/src/api/vendor/golang.org/x/sync/errgroup.(*Group).Go.func1(0xc42041e2c0, 0xc420426180)
/vendor/golang.org/x/sync/errgroup/errgroup.go:57 +0x57
created by [ommited]/src/api/vendor/golang.org/x/sync/errgroup.(*Group).Go
/vendor/golang.org/x/sync/errgroup/errgroup.go:54 +0x66 ```
```

The behavior is given away by 2 things:
   1. The code never passes through the recovery middleware
   2. The panic occurs inside of a goroutine launched by an [errgroup.Group](https://godoc.org/golang.org/x/sync/errgroup) instance
