{
    "title": "That one with the go runtime bug",
    "description": "A matryoshka of bugs ends up in golang itself",
    "summary": "A matryoshka of bugs ends up in golang itself",
    "tags": ["Golang", "Post-mortem", "Runtime", "Compilers"],
    "date": "2021-01-07T00:48:29-03:00",
    "lastmod": "2021-01-07T00:48:29-03:00",
    "categories": ["Programming", "Backend", "Advanced", "Bug", "Open Source"],
    "type": "",
    "weight": 0,
    "publishdate": null,
    "expirydate": null,
    "aliases": null,
    "slug": "that-one-with-the-go-runtime-bug",
    "url": ""
}

Once every couple of months or years, a developer tend to stumble upon a bug that leaves them completely stumped. The cause is not apparent, it cannot be reproduced consistently (or the conditions are rather bizarre and seemingly unrelated to the bug itself) and weeks or months can be spent tinkering on it (because your boss obviously only cares about the business: if fixing it doesn't generate money it isn't broken).

In 2019 I had the opportunity to enjoy this experience with what appeared to be a bug in one of the backend services I was developing at that time. Eventually, it unfolded into a month-long journey for me just to pinpoint the issue on my end and a three-month adventure for the compiler engineer/scientist from google who took upon the task of fixing the issue. Happily, the fix landed on go 1.14 (no backports sadly, as it would break literally a lot of stuff otherwise).

## Where's the issue? 
In my backend service I introduced a nil dereference but that was far from being the worse thing. There was something odd about it: the stacktrace showed a place where it could not happen. Not locally, at least. Neither remotely, after adding logs, but at which point the offending line was correctly displayed, so I immediately suspected of my code (why would I do otherwise?).

```go
func (ls LocationsMutableService) getChildren(ctx context.Context, tree string, loc string) (LocationChildren, error) {
	url := ""
	var children LocationChildren
//line 59 - panic not possible here
	response, err := ls.Client.GET(url, &children)
//line 61 - actual place of panic (response == nil)
	if response.StatusCode == http.StatusNotFound { 
		return nil, errors.New("")
	}
	if err != nil {
		return nil, err
	}
	return children, err
}
```

### Rabbit's Hole B1 - Our panic handler 
We had a rather simplistic fork of the gingonic [recovery handler](https://github.com/gin-gonic/gin/blob/master/recovery.go) (which does some questionable things like inadvertently inducing file I/O per request, although locally) but the main stack frame handling code remained intact from the gin codebase, so I looked up there. At the same time, we also used the newrelic agent for stacktrace and error noticing, so the behavior was the same (both were informing the wrong line, under this specific code).

I proceeded to rewrite the recovery handler using the `debug.Stack()` method instead of doing the manual traversal of PCs (program counters) via `runtime.Callers()`, to see if anything changed. To my surprise, it solved the issue. I wasn't the best way (since `debug.Stack()` spits a byte slice with the already formed stacktrace) but as long as the correctness was assured, I was willing to do the sacrifice. The bug was surely in gingonic's code... right.

Oh, there was also another point of failure now

### Rabbit's Hole B2 - NewRelic's agent 
Enter https://github.com/newrelic/go-agent/issues/100

They were using the same mechanism, `runtime.Callers()`, to construct the stacktrace, so I had to open an issue there and discuss it. Some comment arouse about the runtime, but it still couldn't be an issue on their end... right?

### Rabbit's Hole B3 - The go runtime 
Enter https://github.com/golang/go/issues/34123

Diagnosing a runtime issue of this nature can be an herculean work. For starters, the following factors limited reproducibility:
* The inner workings of the go compiler and runtime at work with this bug were (and still are) beyond my capabilities, so I was completely unable to isolate the issue to a piece a code I could share with the golang team
* Any change to the affected package had a great change of making the bug disappear, so I was stuck with a git commit
* Due to how the web APIs were designed at the company, these couldn't be executed out of the box, so I was stuck with a docker image of said git commit (docker became my friend here)

So, with the bug and starting assumption that `debug.Stack()` and `runtime.Callers()` PCs differ on panic trigger sites, the following steps were taken:
1. Get to the bottom and the source of the [debug.Stack()](https://github.com/golang/go/blob/38c2c12bc1b3da40e1b33cac9268b7df9fa49a7e/src/runtime/mprof.go#L813-L828) and [runtime.Callers()](https://github.com/golang/go/blob/6b37b15d9520f9fa2b819e66a37fac4b2d08da78/src/runtime/traceback.go#L799-L801) functions. Interestingly, they share some common code but in completely different ways of use (please note that these revisions are up to date, so implementation details from the original issue may be lost). On the botton there is the [traceback](https://github.com/golang/go/blob/6b37b15d9520f9fa2b819e66a37fac4b2d08da78/src/runtime/traceback.go#L794-L803) runtime helper, which provides the internal stack frame buildup and traversal utilities
2. Provide the output of -S on the LocationsMutableService.getChildren function. Build with go build -gcflags=-S, save the output, find the code for that function, and paste it here.
3. Disassemble the binary (objdump -d), find the same function above, and paste the disassembly of it.
4. Show the actual values in the runtime.Callers return slice. Make sure to use the exact binary used in the previous 2 points (i.e. if you hack in some print statements for this, use the binary with the print statements in it when disassembling).

## Off-by-one 
After some experimentation and log hacking into the runtime code, I found out some additional clues:
* This issue happens from go 1.12 and afterwards, including 1.13. 1.11 seems unaffected (can't say anything about prior versions)
* It can be observed that PCs are being decremented on traceback.callers when it shouldn't, but only in some specific cases like the repro I have at work (and, for the time being, only on the panic site as we don't any any proof that suggests otherwise)
* After some history bisection I discovered that some heavy work was done on tracebacks for 1.12 which involves how PCs are manipulated, this particular PR stood out since the description and changes seem directly related to this issue: [232c979#diff-5a2de8a1053d4e11fbc71407a5361e93](https://github.com/golang/go/commit/232c9793092115870a430ef3c9ef9ae04f9e25c9#diff-5a2de8a1053d4e11fbc71407a5361e93)

With all of this info together, one of the compiler engineers of the golang team started to identify and work on the issue (it was an off-by-one bug with program counter handling, but not a trivial one). And another one came with a simple (?) and isolated repro:

```go
// run
// Copyright 2019 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.
// Make sure that the line number is reported correctly
// for faulting instructions.
package main

import (
	"fmt"
	"runtime"
)

var x byte
var p *byte

//go:noinline
func f() {
	q := p
	x = 11  // line 23
	*q = 12 // line 24
}

func main() {
	println(runtime.Version())
	defer func() {
		recover()
		var pcs [10]uintptr
		n := runtime.Callers(1, pcs[:])
		frames := runtime.CallersFrames(pcs[:n])
		for {
			f, more := frames.Next()
			if f.Function == "main.f" && f.Line != 24 {
				panic(fmt.Errorf("expected line 24, got line %d", f.Line))
			}
			if !more {
				break
			}
		}
	}()
	f()
}

```

And the long awaited fix: https://go-review.googlesource.com/c/go/+/196962/2#message-1b08a694cb1ee65964a69e436361f75eb5c8fe67.

It was a wild ride. I liked dwelling into this (and I just love the fact that you can just hack away into your local go code base) but I can live another year or two before having to hope on another ride like this one.

## Conclusion 
I will gladly quote the cigarette man from the X-Files on this: "trust no one".

Your code will always be the first suspect but always remember that every link of the chain has and __will__ have bugs, and if things go awry and you code seems completely out of the loop on the issue maybe it's not the first place to look at.
