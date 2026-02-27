# YAMLStar Gloat Build — CPU Profile Analysis

**Date**: 2027-02-27
**Branch**: `gloat`
**Input**: `"foo: 42"` (simplest useful YAML)
**Iterations**: 101 (1 warmup + 100 profiled)
**Total wall time**: 4m 23s (~2.6s per `yaml/load` call)
**Comparison**: GraalVM native-image does the same work in ~28ms total

## How the Profile Was Collected

A standalone Go binary was built from the YAMLStar Clojure sources
using `gloat` (Glojure AOT compiler).
Go's `runtime/pprof` CPU profiler was added to the generated
`main.go` around the `-main` call.

```
gloat [all yamlstar .clj files] main.clj -o /tmp/yamlstar-profile/
# Edit main.go to add pprof, fix pkg/main package name
go build -o /tmp/yamlstar-bench .
/tmp/yamlstar-bench "foo: 42"   # writes /tmp/yamlstar-cpu.prof
```

## Flame Graph

The interactive flame graph is in
[yamlstar-flamegraph.html](yamlstar-flamegraph.html)
(download and open in a browser — it's self-contained).

## Call Graph (SVG)

Full call graph:

![Call Graph](yamlstar-callgraph.svg)

Focused view (yamlstar + Glojure runtime only):

![Focused Call Graph](yamlstar-focused.svg)

## Top Functions by Self Time (flat)

```
      flat  flat%   sum%        cum   cum%
    37.15s 10.89% 10.89%     37.40s 10.96%  lang.(*SubVector).Nth
    23.24s  6.81% 17.70%     24.97s  7.32%  runtime.tryDeferToSpanScan
    16.17s  4.74% 22.43%     54.96s 16.10%  reflect.implements
    15.90s  4.66% 27.09%     42.75s 12.53%  runtime.mallocgcSmallScanNoHeader
    12.99s  3.81% 30.90%     34.63s 10.15%  lang.(*SubVector).AssocN
    12.40s  3.63% 34.53%     12.40s  3.63%  internal/abi.Name.Name
    11.28s  3.31% 37.84%    186.57s 54.67%  reflect.Value.call
    11.04s  3.23% 41.07%     29.85s  8.75%  runtime.scanObjectsSmall
     9.71s  2.85% 43.92%     15.28s  4.48%  runtime.resolveTypeOff
     6.89s  2.02% 45.94%      6.89s  2.02%  runtime.(*mspan).writeHeapBitsSmall
     4.71s  1.38% 54.27%    186.69s 54.70%  lang.Apply
     4.57s  1.34% 56.99%     15.43s  4.52%  lang.(*Var).IsMacro
     3.92s  1.15% 60.48%     49.92s 14.63%  runtime.mallocgc
```

## Top Functions by Cumulative Time

```
      flat  flat%   sum%        cum   cum%
     4.71s  1.38%  1.38%    186.69s 54.70%  lang.Apply
    11.28s  3.31%  4.69%    186.57s 54.67%  reflect.Value.call
     1.22s  0.36%  5.12%    186.47s 54.64%  lang.FnFunc.Invoke
     0.85s  0.25%  5.37%    185.66s 54.40%  parser.LoadNS.func8          ← call()
     0.06s 0.018%  5.41%    182.50s 53.48%  prelude.LoadNS.func18.1      ← name*()
     0.05s 0.015%  5.43%    133.32s 39.07%  parser.LoadNS.func4.1
     0.14s 0.041%  5.47%    116.99s 34.28%  parser.LoadNS.func28         ← combinator
     1.05s  0.31%  5.78%    113.75s 33.33%  parser.LoadNS.func23         ← combinator
     0.04s 0.012%  5.81%        68s 19.93%  runtime.gcBgMarkWorker
     1.59s  0.47%  6.34%     58.12s 17.03%  runtime.gcDrain
    16.17s  4.74% 11.08%     54.96s 16.10%  reflect.implements
     3.92s  1.15% 12.22%     49.92s 14.63%  runtime.mallocgc
    37.15s 10.89% 28.93%     37.40s 10.96%  lang.(*SubVector).Nth
    12.99s  3.81% 32.74%     34.63s 10.15%  lang.(*SubVector).AssocN
     1.05s  0.31% 36.88%     28.99s  8.49%  lang.coerceGoValue
```

## Root Cause Analysis

### 1. Reflection-based function dispatch — 54.7% of CPU

Every Clojure function call in Glojure goes through:

```
lang.Apply → lang.FnFunc.Invoke → reflect.Value.Call → reflect.Value.call
```

This is the single largest cost.
The parser's `call` function (the central dispatch for grammar rules)
generates 15–20 `lang.Apply` invocations per grammar rule.
For `"foo: 42"`, thousands of grammar rules fire, each going through
this reflection chain.

**This is intrinsic to the Glojure runtime.**
Short of rewriting hot paths in native Go or adding a fast-path
in Glojure for known function signatures, this cannot be reduced
from the Clojure side.

### 2. GC pressure — 17–20% of CPU

`runtime.gcDrain` + `mallocgc` + `scanSpan` + `scanObjectsSmall`
together consume ~20% of CPU.
Root causes:

- Every `lang.Apply` allocates a `[]any{...}` slice
- Persistent vector operations (`conj`, `pop`, `assoc`) allocate
  new `SubVector` nodes on every state-push/pop
- Closure creation for grammar rules (each `name*` call wraps a fn)

### 3. SubVector.Nth — 10.9% flat (!) of CPU

`SubVector.Nth` is the **#1 function by self-time**.
This comes from two sources:

**a) String character access via `nth`**

The `chr` combinator does:
```clojure
(nth @(:input p) @(:pos p))
```

In Glojure, `nth` on a string does `[]rune(x)[n]` — converting the
**entire input string to a rune slice** on every character access.
This is O(n) per character, called thousands of times per parse.
See: `glojure/pkg/lang/iteration.go:46`

**b) State vector access**

`state-curr`, `state-prev`, `state-pop` all index into the parser's
state vector, which becomes a `SubVector` chain after repeated
`conj`/`pop` operations.

### 4. `func-name` try/catch on every `call` — contributes to 53%

The `func-name` function uses Go `defer/recover` (the Glojure
translation of `try/catch go/any`) to safely probe whether a
function supports the name-sentinel protocol.
This is called **twice** per `call` invocation (lines 138 and 147
of parser.clj).
Go's `defer` has non-trivial overhead even on the happy path.

### 5. `rng` regex compilation — per-character overhead

The `rng` combinator compiles a new regex on every invocation:
```clojure
(re-pattern (str "^[" low "-" high "]"))
```

This creates and compiles a new RE2 regex for every character-range
check.
Additionally, `(subs input pos)` allocates a new substring from the
current position to end-of-input on every call.

### 6. `make-receivers` string operations — per-rule overhead

Called once per grammar rule invocation to build callback routing
keys.
Walks the state stack doing `re-matches`, `str/includes?`,
`str/replace`, and `str/join` to produce keys like
`"try__c_flow_sequence__all__x5b"`.
Most rules have no registered callbacks, making this work wasted.

## Cost Breakdown Summary

| Category | % of CPU | Nature |
|----------|----------|--------|
| `reflect.Value.call` (via `lang.Apply`) | ~55% | Glojure runtime |
| GC (`gcDrain`, `mallocgc`, scan) | ~20% | Allocation pressure |
| `SubVector.Nth` + `AssocN` | ~15% | Data structure overhead |
| `reflect.implements` | ~5% | Type assertion overhead |
| `lang.coerceGoValue` | ~5% | Go↔Clojure conversion |

## Optimization Opportunities

### Easy wins (Clojure-side fixes)

| Fix | Expected Impact |
|-----|----------------|
| Pre-compile `rng` regex at rule-definition time | Medium — eliminates thousands of regex compilations |
| Replace `nth` on input string with `subs`+`first` or direct rune extraction | High — eliminates O(n) `[]rune` allocation per char |
| Cache `make-receivers` result or skip when no callbacks | Medium — eliminates string ops on every rule |
| Pre-resolve `func-name` at `name*` creation; avoid try/catch in `call` | Medium — eliminates 2 defer/recover per rule |

### Requires Glojure runtime changes

| Fix | Expected Impact |
|-----|----------------|
| Fast-path `lang.Apply` for known arities (avoid reflection) | Very High — could halve total CPU |
| O(1) string indexing (byte-offset + `utf8.DecodeRuneInString`) | High — eliminates `[]rune` copies |
| Reduce `[]any{}` allocation in `lang.Apply` (pool or stack-alloc) | Medium — reduces GC pressure |

## Reproducing This Profile

```bash
cd /home/ingy/src/yamlstar
git checkout gloat

# 1. Build Go project directory
SRCS="core/src/yamlstar/parser/prelude.clj \
  core/src/yamlstar/parser/parser.clj \
  core/src/yamlstar/parser/receiver.clj \
  core/src/yamlstar/parser/grammar.clj \
  core/src/yamlstar/parser.clj \
  core/src/yamlstar/composer.clj \
  core/src/yamlstar/resolver.clj \
  core/src/yamlstar/constructor.clj \
  core/src/yamlstar/core.clj \
  main.clj"
make shell cmd="gloat $SRCS -o /tmp/yamlstar-profile/ --force"

# 2. Fix Go package name conflict
sed -i 's/^package main$/package mainpkg/' \
  /tmp/yamlstar-profile/pkg/main/loader.go

# 3. Add pprof to main.go (import runtime/pprof, wrap myMain.Invoke)

# 4. Build without stripping debug symbols
make shell cmd='cd /tmp/yamlstar-profile && go mod tidy && \
  go build -o /tmp/yamlstar-bench .'

# 5. Run
/tmp/yamlstar-bench "foo: 42"

# 6. Analyze
go tool pprof -http=:8080 /tmp/yamlstar-bench /tmp/yamlstar-cpu.prof
go tool pprof -top -cum /tmp/yamlstar-bench /tmp/yamlstar-cpu.prof
go tool pprof -svg /tmp/yamlstar-bench /tmp/yamlstar-cpu.prof > graph.svg
```
