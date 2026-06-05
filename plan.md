# Learning Plan for prometheus/client_golang

This plan is for reading the Prometheus Go client library as a Go developer who wants to understand the design, not just the public API. Follow it in order. Each stage has source files to read, questions to answer, and small exercises that force you to verify your understanding.

## 1. Start With the User-Facing Shape

Read:

- `README.md`
- `prometheus/doc.go`
- `prometheus/examples_test.go`
- `examples/simple/main.go`
- `examples/random/main.go`
- `examples/middleware/main.go`

Goals:

- Understand the two main halves of the repository: instrumentation under `prometheus/` and HTTP API clients under `api/`.
- Learn the public vocabulary: `Collector`, `Metric`, `Registerer`, `Gatherer`, `Counter`, `Gauge`, `Histogram`, `Summary`, `Vec`, labels, descriptors.
- Notice how examples are often written as Go tests. They are executable documentation.

Questions:

- What does application code usually create directly?
- What does Prometheus itself call during scraping?
- What is registered: a metric, a collector, or both?

Exercise:

- Write a tiny local program using `Counter`, `GaugeVec`, and `promhttp.Handler`.
- Then find the exact library code that each constructor eventually calls.

## 2. Core Interfaces and Data Flow

Read:

- `prometheus/collector.go`
- `prometheus/metric.go`
- `prometheus/registry.go`
- `prometheus/value.go`
- `prometheus/desc.go`

Goals:

- Understand the central scrape flow:
  - user registers collectors
  - registry gathers collectors
  - collectors emit metrics
  - metrics write protobuf DTOs
  - encoder exposes text/openmetrics output elsewhere
- Separate the roles of `Collector`, `Metric`, and `Desc`.

Questions:

- Why are `Collector` and `Metric` separate interfaces?
- Why does `Metric.Write` write into `dto.Metric` instead of returning text?
- What errors are detected at registration time vs collect time?
- Why does `Desc` store `id`, `dimHash`, const labels, and variable label names?

Exercise:

- Implement a minimal custom collector with `Describe` and `Collect`.
- Use `NewDesc` and `MustNewConstMetric`.
- Intentionally break label cardinality and observe where the error appears.

## 3. Descriptors, Labels, and Identity

Read:

- `prometheus/desc.go`
- `prometheus/labels.go`
- `prometheus/value.go`
- `prometheus/desc_test.go`
- `prometheus/value_test.go`
- `prometheus/vec_test.go`

Goals:

- Understand label names vs label values.
- Understand constant labels vs variable labels.
- Understand constrained variable labels.
- Understand why metric identity depends on metric name plus label values.

Questions:

- Why are const label values included in a descriptor but variable label values are not?
- Why does `NewDesc` hash const label values into `id`?
- Why does `dimHash` use help text, unit, and label names?
- Why are variable label names prefixed with `$` before hashing dimensions?
- What happens if a const label and variable label use the same name?

Exercise:

- Create two descriptors with the same metric name and different const label values.
- Create two descriptors with the same metric name but different variable label names.
- Predict registration behavior before running the tests.

## 4. MetricVec Internals

Read:

- `prometheus/vec.go`
- `prometheus/counter.go`
- `prometheus/gauge.go`
- `prometheus/example_metricvec_test.go`
- `prometheus/vec_test.go`

Goals:

- Understand how `CounterVec`, `GaugeVec`, etc. share the generic `MetricVec`.
- Understand `WithLabelValues`, `With`, `GetMetricWithLabelValues`, `Delete`, `Reset`, and currying.
- Understand how label values are hashed and how hash collisions are handled.

Questions:

- What is stored in `metricMap.metrics`?
- Why are both hash and raw label values stored?
- What is the difference between `WithLabelValues` and `GetMetricWithLabelValues`?
- Why can `WithLabelValues` panic while `GetMetricWithLabelValues` returns an error?
- How do label constraints affect lookup, deletion, and exported labels?

Exercise:

- Trace `counterVec.WithLabelValues("GET", "200").Inc()` from public API to storage.
- Add log statements locally or use the debugger to see when a child metric is created.

## 5. Simple Metric Types

Read:

- `prometheus/counter.go`
- `prometheus/gauge.go`
- `prometheus/untyped.go`
- `prometheus/timer.go`
- `prometheus/counter_test.go`
- `prometheus/gauge_test.go`
- `prometheus/timer_test.go`

Goals:

- Understand how simple metric types wrap descriptors and values.
- Learn the concurrency strategy for metric updates.
- Understand the split between scalar metrics and vector metrics.

Questions:

- Why does `Counter` disallow decrementing?
- How are float values stored atomically?
- How do `CounterFunc` and `GaugeFunc` differ from normal counters/gauges?
- When is `selfCollector` used?

Exercise:

- Follow `NewCounter` and `NewCounterVec` side by side.
- Identify which code is specific to counters and which code is generic vector machinery.

## 6. Histograms, Summaries, and Observers

Read:

- `prometheus/observer.go`
- `prometheus/histogram.go`
- `prometheus/summary.go`
- `prometheus/histogram_test.go`
- `prometheus/summary_test.go`
- `examples/nativehistogram/main.go`

Goals:

- Understand the `Observer` abstraction.
- Understand classic histogram buckets, native histograms, summaries, objectives, and quantiles.
- Learn where complexity and performance concerns enter the library.

Questions:

- Why do histograms and summaries both implement `Observer`?
- What state does a histogram keep per bucket?
- What is the difference between summary quantiles and histogram buckets?
- Why are native histograms more complex?
- What invariants are maintained during `Observe` and `Write`?

Exercise:

- Trace a single `histogram.Observe(1.23)` call.
- Then trace `Write` and see how bucket counts become DTO fields.

## 7. Registry and Gathering Semantics

Read:

- `prometheus/registry.go`
- `prometheus/registry_test.go`
- `prometheus/wrap.go`
- `prometheus/wrap_test.go`

Goals:

- Understand registration rules and duplicate detection.
- Understand checked vs unchecked collectors.
- Understand wrapping registerers with const labels and prefixes.
- Understand how gather errors are handled.

Questions:

- What makes two descriptors compatible?
- What makes two collectors duplicates?
- Why does the registry track descriptor IDs and dimension hashes?
- What is a pedantic registry?
- What is the difference between `Register`, `MustRegister`, and `Unregister`?

Exercise:

- Write a small test that registers conflicting collectors.
- Predict the exact error before running it.

## 8. HTTP Exposure and Instrumentation

Read:

- `prometheus/promhttp/http.go`
- `prometheus/promhttp/option.go`
- `prometheus/promhttp/instrument_server.go`
- `prometheus/promhttp/instrument_client.go`
- `prometheus/promhttp/delegator.go`
- `prometheus/promhttp/http_test.go`
- `prometheus/promhttp/instrument_server_test.go`
- `prometheus/promhttp/instrument_client_test.go`

Goals:

- Understand how gathered metrics become HTTP responses.
- Understand content negotiation, compression, error handling, and timeouts.
- Understand HTTP server/client instrumentation helpers.

Questions:

- How does `promhttp.Handler` choose an output format?
- Where are gather errors written?
- What metrics does HTTP middleware create?
- How does the delegator preserve optional `http.ResponseWriter` interfaces?

Exercise:

- Trace one scrape request through `promhttp.Handler`.
- Then trace one instrumented HTTP handler request through server middleware.

## 9. Auto Registration, Push, and Test Utilities

Read:

- `prometheus/promauto/auto.go`
- `prometheus/push/push.go`
- `prometheus/testutil/testutil.go`
- `prometheus/testutil/lint.go`
- `prometheus/testutil/promlint/`
- `prometheus/promauto/auto_test.go`
- `prometheus/push/push_test.go`
- `prometheus/testutil/testutil_test.go`

Goals:

- Understand convenience APIs built on top of the core.
- Understand when auto-registration is helpful and when it hides errors.
- Learn how this project tests metric output.

Questions:

- What does `promauto` trade away compared with explicit registration?
- How does the push gateway client group metrics?
- How does `testutil` compare expected metric text with gathered output?
- What does the linter consider invalid or suspicious?

Exercise:

- Use `testutil.CollectAndCompare` against one of your own toy collectors.
- Break help text or labels and inspect the failure.

## 10. Built-In Collectors

Read:

- `prometheus/go_collector.go`
- `prometheus/go_collector_latest.go`
- `prometheus/process_collector.go`
- `prometheus/build_info_collector.go`
- `prometheus/expvar_collector.go`
- `prometheus/collectors/`
- `prometheus/internal/go_runtime_metrics.go`

Goals:

- Understand how library-provided collectors adapt runtime, process, database, build, expvar, and version data.
- See examples of collectors that bridge external data into Prometheus metrics.
- Learn how platform-specific files are organized with build tags.

Questions:

- Which collectors are registered by default?
- How does the Go collector handle runtime metric changes across Go versions?
- Why are some collectors under `prometheus/` and others under `prometheus/collectors/`?
- How do platform-specific process collectors share common behavior?

Exercise:

- Trace one Go runtime metric from runtime source to exported Prometheus metric.
- Compare Linux, Darwin, Windows, and unsupported process collector files.

## 11. API Client Package

Read:

- `api/client.go`
- `api/prometheus/v1/api.go`
- `api/prometheus/v1/example_test.go`
- `api/prometheus/v1/api_test.go`

Goals:

- Understand the separate client for querying a Prometheus server.
- Keep this mentally separate from instrumentation.
- Learn the HTTP client design, request construction, context use, and response decoding.

Questions:

- What does the API client consider stable vs experimental?
- How are query endpoints represented in Go?
- How does the package handle warnings, errors, and response statuses?

Exercise:

- Trace `QueryRange` from public method to HTTP request and response decoding.

## 12. Experimental and Tutorial Areas

Read:

- `exp/README.md`
- `exp/api/remote/`
- `tutorials/whatsup/README.md`
- `tutorials/whatsup/main.go`

Goals:

- Understand what is outside the stable API surface.
- See larger tutorial-style usage.
- Learn how the project isolates experimental code with separate modules.

Questions:

- Why is `exp/` a separate Go module?
- Which APIs would you avoid depending on in production?
- What does the tutorial demonstrate that the small examples do not?

Exercise:

- Run the tutorial tests.
- Read the generated remote protobuf code only after understanding the public wrapper.

## 13. Tests, Benchmarks, and Project Practices

Read:

- `CONTRIBUTING.md`
- `Makefile`
- `prometheus/benchmark_test.go`
- `prometheus/*_test.go`
- `.github/workflows/`

Goals:

- Understand how maintainers protect behavior.
- Learn test style, benchmark style, and compatibility expectations.
- Notice how much behavior is specified by tests rather than comments.

Questions:

- Which tests are table-driven?
- Which tests encode public compatibility guarantees?
- Where are benchmarks focused, and why?
- What commands should you run before submitting a change?

Exercise:

- Pick one small behavior, write a failing test for a hypothetical bug, then fix it locally.
- Keep the change private if your goal is learning, but write it as if it were a real PR.

## Suggested Reading Order Summary

1. Examples and docs.
2. `Collector`, `Metric`, `Desc`, `Registry`.
3. Labels and descriptors.
4. `MetricVec`.
5. Counter, gauge, untyped, timer.
6. Histogram and summary.
7. Registry rules and wrapping.
8. `promhttp`.
9. `promauto`, `push`, `testutil`.
10. Built-in collectors.
11. API client.
12. Experimental and tutorial modules.
13. Tests, benchmarks, contribution workflow.

## Habits While Reading

- Read tests next to implementation. In this repository, tests often explain edge cases better than comments.
- Trace one public call all the way down before moving to the next abstraction.
- Keep a notebook of invariants: label cardinality, descriptor compatibility, registry uniqueness, metric write behavior, and concurrency rules.
- Prefer asking “why does this type exist?” before asking “what does this function do?”
- After reading a file, write a five-line summary of its role in the package.
- Reproduce errors intentionally. Registration failures and label cardinality errors teach the design quickly.

## Milestone Projects

Use these as checkpoints.

1. Build a custom collector for fake application state using `NewDesc` and `MustNewConstMetric`.
2. Build HTTP middleware manually, then compare it with `promhttp` helpers.
3. Build a small test suite around your metrics using `testutil`.
4. Add a constrained label to normalize request paths and prove that cardinality drops.
5. Implement a toy `MetricVec`-like map outside the library to understand hashing, currying, and collision handling.
6. Read one non-trivial open issue or PR in the upstream repository and map it to the files in this plan.
