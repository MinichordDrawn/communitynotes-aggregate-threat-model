# Aggregate Threat Model: finagle, twitter-server, finatra, and Community Notes

One record of every finding from the four per-system assessments and the three
cross-boundary assessments of X's service stack and its Community Notes scoring pipeline.
It covers the four systems, how the three Scala layers stack and where the pipeline sits
beside them, the threats inside each, the threats at the boundaries between them, the
chains that join a gap in one system to a gap in another, and the candidates tested and
rejected. Every finding is code-confirmed against the source. Each threat is recorded
once, with every position an actor can start from.

One boundary is different in kind. The three Scala layers share code: finatra imports
twitter-server and finagle, and twitter-server imports finagle. Community Notes shares no
code with any of them. Its seam to the stack is a set of contracts (a column order pinned
to a Scala schema class, Thrift enrollment codes, an HTTP API, a daily data export) whose
far end has no source in any tree. Findings on that seam are marked as resting on inferred
deployment facts wherever they do, and nothing inferred is counted in the impact ranking.

## The model in brief

The assessments measure each system against a model of self-observing systems. The model
has four constraints: no central controller exists; an undetected failure carries a
catastrophic cost; the environment is adversarial; no external maintenance arrives. A system
that survives under those constraints satisfies eight properties, and every observer in the
system satisfies each of them:

1. **Heartbeat.** Every observer emits a continuous liveness signal.
2. **Substrate Dependency.** An observer shares the failure substrate of what it watches.
3. **Continuous State Verification.** State is re-verified continuously, not once.
4. **System-Wide Response.** A detection anywhere reaches the whole system.
5. **Self-Defense.** An observer resists removal, replacement, and reconfiguration.
6. **Death Signaling.** A peer notices and records an observer's death.
7. **Automatic Integration.** A new component is observed by construction, not by convention.
8. **Reflexive Response.** A detected fault produces an immediate halt, not a continue.

**Structure.** The smallest structure that satisfies all eight is a ring of at least three
mutually observing observers plus one coordinator, written 3+1. The coordinator
surface-checks every member, takes validation back from each, and holds the primary halt.
Rings nest: coordinators form rings of their own, and the chain runs upward until the whole
system is covered. Halt on fault: when any node detects a fault it alerts and halts, the
halt propagates to at least two other nodes, and no actor passes a critical observer without
an alert and a halt. Execution-observation coupling: the execution thread checks the
observer's heartbeat and stops the moment it stops.

**Tiers.** Required structure scales with consequence. Bounded: a scheduled check and an
alarm. Significant: a heartbeat and automatic restart. Severe: redundant observers.
Catastrophic: the full 3+1 ring. Customer-data floor: any subsystem that carries customer
data sits at Catastrophic whatever else is true of it.

**Trust boundaries.** B1 is the observer layer itself; no human belongs inside it. B2 is a
privileged insider with code, config, flag, or deploy access. B3 is an external actor with
no foothold. The model treats B2 and B3 both as adversaries, so a finding that needs an
insider position is not discounted.

**Judging within the model.** All three systems keep serving through a detected fault; the
model halts. The findings measure the systems against the model, not against their own
design goals. The model is not anti-availability: structural checks run faster than
procedural ones, so a structural design can improve availability and performance as well as
safety. The assessments do not settle which design is more available; they report the
safety findings only.

## The four systems

### finagle

finagle is an open-source RPC system written in Scala on top of Netty. It is the data
plane that services are built on. It manages connections, load balancing, retries,
timeouts, failure detection, and telemetry between services, protocol-agnostic across HTTP,
Thrift, and Mux. Its self-observation is a set of observers: a per-session failure detector,
a circuit breaker (failure accrual), a fail-fast factory, a load balancer with panic mode, a
per-request timeout, an expiring service, a channel stats handler, service discovery over
ZooKeeper (serversets), a metrics registry with an HTTP exporter, and a sampling tracer.
Customer request and response payloads ride finagle sessions on every call. Its goal is
availability and throughput: it keeps serving through partial failure.

### twitter-server

twitter-server is the common server template for internal X services. A service author
extends the `TwitterServer` trait and writes a `main`; the trait mixes in the rest and
stands up an admin HTTP server on port 9990 by default. That admin plane exposes lint
results, client and server registries, metrics and histograms, build and process info,
thread and contention dumps, heap and CPU profiles, and lifecycle controls. No admin route
carries authentication. Its goal is operator observability and orchestration: it answers
health and readiness, exposes state on request, and keeps the service running.

### finatra

finatra is a request-serving framework layered on twitter-server and finagle. A service
author writes controllers and wires filters; finatra supplies the HTTP and Thrift request
path, routing, JSON, validation, exception mapping, and dependency injection through inject
and Guice. Its own source holds 24 observation functions across the two paths: request
filters for stats, exception mapping, access logging, nack handling, response shaping, and
MDC; an exception manager with framework mappers; and two startup observers. Customer
request bodies cross its filters directly, and its unhandled-exception path builds a 500
body from the throwable. Its goal is developer productivity and request handling that keeps
serving.

### Community Notes

Community Notes is a Python batch pipeline that reads contributor notes, ratings, the
previous run's note status history, and user enrollment, trains matrix-factorization and
Gaussian models, and writes one status per note plus one score per contributor. It runs as
four phases in separate processes joined by files: post-selection similarity, prescoring,
final scoring, and contributor scoring. The README states the code is developed in an
internal repository and exported on deploy, and public users download the daily data and
run it themselves. Three sibling projects sit beside it: an API note writer that drafts
notes with an LLM and submits them through the X API, a live-note generator, and a
downstream post scorer. Customer data is note text, author and rater identifiers,
enrollment states, and the public status of every note. Its goal is one status per note,
aligned with prior runs, and the run keeps serving through every detection: it filters,
demotes, holds, or logs, and proceeds to a public status.

### How they stack

finatra sits on twitter-server, which sits on finagle, and all three run inside one JVM per
service.

- **A customer request** enters on the service's external HTTP or Thrift port, passes
  finagle's server stack, then finatra's filter chain and controller. Outbound calls leave
  through finagle clients, whose endpoints come from serversets in ZooKeeper.
- **finatra on twitter-server.** finatra's `HttpServer` and `ThriftServer` extend
  twitter-server's base and its lifecycle hooks. finatra's `AdminHttpRouter` adds
  application routes to the admin plane. finatra's route table is written into the global
  registry that twitter-server serves. finatra's `Awaiter` and twitter-server's shutdown
  timer share one timer thread.
- **finatra on finagle.** finatra's filters are finagle `Filter`s over finagle `Service`s.
  finatra binds one injected `StatsReceiver` and one `ResponseClassifier` to both its own
  filters and finagle's server stack. Its servers are finagle `Http.server` and
  `ThriftMux.server`.
- **twitter-server on finagle.** The admin handlers read finagle's client, server, endpoint,
  balancer, and stats registries. The admin plane itself is hosted on a finagle HTTP server
  inside the same JVM as finagle's clients and servers.

### Where Community Notes sits

Community Notes runs beside the stack, not on it. No file in finatra, twitter-server, or
finagle names Community Notes, and no Community Notes file imports or calls any of the
three. Nine crossings exist, each with exactly one endpoint in a tree:

- **Into the pipeline.** The daily data export supplies its four input TSVs. An
  enrollment producer named only in a docstring supplies enrollment states as strings. An
  abstract notes-data client with no implementation supplies the live-note generator's
  per-note status. Two optional internal Python modules, absent from every tree, configure
  child-process logging.
- **Out of the pipeline.** The three note outputs go to a combine job and a Scala schema
  class named only in comments, with column order pinned by index. The contributor output
  carries enrollment states as integer Thrift codes to an importer named only in a
  docstring. A not-helpful tag order is pinned to a Scala accessor by comment. The
  generator writes a classification string to a store described as thrift-side.
- **Request and response.** The note writer calls two X API paths through an external CLI
  absent from every tree, and sees only an exit code and stdout.

Whatever serves the API and consumes the outputs is, by X's own architecture, some
service on the Scala stack. No tree shows which, so every finding that depends on that is
marked inferred.

### Combined structure

| Layer | Own structure | Rings |
|---|---|---|
| finagle | Directed acyclic graph of 14 observers. Each watches a resource below it and reports up the stack or into a passive sink. The one round trip, the failure-detector ping, is answered by a component that echoes and checks nothing. | 0 |
| twitter-server | Request-driven star of 41 nodes. Every observation component is a `Service` that runs when an HTTP request arrives and answers that requester. Every path ends at the requester, a passive store, or a halt call. | 0 |
| finatra | Linear filter chain. The HTTP and Thrift chains are one-direction folds; every observer watches everything below it and nothing watches it back. The one cycle is a bounded two-node exception re-dispatch carrying a throwable, not a health signal. | 0 |
| finagle and twitter-server seam | 22 one-directional edges. 21 point from twitter-server into finagle stores or from finagle into its own stores. One points from finagle into a twitter-server logger and returns `false`. | 0 |
| finatra on the two lower layers | 23 one-directional edges. 21 point from finatra into lower-layer stores, hosts, or sinks. Two point from a lower layer into finatra's request path and end at a metrics sink, a null monitor, or a log. | 0 |
| Community Notes | Directed acyclic graph with a linear rule chain at its core, across four phases joined by files. 50 observers. The one cycle is the cross-run loop through the status-history file, which carries labels and timestamps, never observer health, and does not close by filename in source. | 0 |
| Community Notes and the Scala stack | 9 crossings, each with exactly one endpoint in a tree. Four into the pipeline, four out, one request and response. No code edge in any of the four trees. | 0 |

The union of three DAGs joined at one-directional seams is a DAG. Adding Community Notes
adds a second, disjoint DAG: no seam edge has both endpoints in a tree, so no seam edge
lies on any cycle in source. Joined by deployment, the two form data loops through
endpoints with no source, and no loop has three mutually observing nodes. Zero rings at
any layer, zero coordinators, zero super-rings, on either side and across the seam.

Neither upper layer is a coordinator over the layers beneath it. twitter-server belongs to
no ring, removes itself from finagle's observation by running the admin server with null
stats, a null tracer, and admission control off, reads live status from one of finagle's 14
observers (balancer status through `Metadata.status`), receives no validation back, and
halts only on an inbound POST. finatra belongs to no ring, checks the closed state of its
hosting servers and nothing about the lower layers' observers, receives no validation back,
and halts only on its own startup assertions while passing every lower-layer runtime fault
through as a response. The closest thing to a heartbeat and a halt in the union is finatra's
`Awaiter`, a dead-man's switch that reads a closed flag, is itself unobserved, and runs on
the finagle timer it would need to outlive.

Community Notes has four hub candidates of its own: the rule applier, the post-scoring
merge, the scorer runner, and the phase runner. Each fails the test. None belongs to a
ring, none takes validation back from what it collects, and the one cross-run halt is off
by default. Across the seam, a Scala node coordinating the pipeline, or the reverse, fails
every clause by absence: zero edges in either direction, no health signal crossing, and no
node on either side able to halt the other. The pipeline's outbound signals are TSVs, an
HTTP payload through a CLI, and a string to a store; none is addressed to a Scala
observer and none carries the pipeline's health.

**Tier.** Every layer and every seam sits at Catastrophic. finagle: twelve observer groups
carry or govern customer data. twitter-server: the admin plane dumps process memory of a
request-serving process, holds a one-POST halt, and ships inside every service.
finatra: customer request bodies pass through its filters. Community Notes: note text,
author and rater identifiers, and enrollment flow through every phase, and the note
status it writes is public. The seams share the failure
substrate of the data path, since the admin plane runs inside the JVM that finagle routes
payloads through and finatra serves requests in. At Catastrophic every missing property
counts as a real gap.

**Missing structural components across the stack.** Descriptive only.

- **Mutual-observation cycle: absent** at every layer and across every seam.
- **Coordinator: absent.** The nearest candidates, twitter-server's `AdminHttpServer` and
  finatra's `StatsFilter`, host or take the results of everything below them, take no
  validation back, and substitute a log line or a 500 for a halt.
- **Check and report separation: absent.** Every detection ends in a passive sink: a
  `StatsReceiver`, a logger, MDC, a registry, a span, or the HTTP requester.
- **Halt path with propagation: absent.** Three halts exist in the stack, and the pipeline
  adds a worker exception that ends one process and a cross-run flip check that ships off
  by default. twitter-server's fires only on
  an inbound POST. finatra's fire only at startup and unwind into the inject lifecycle.
  `Awaiter` fires only on server closure. None fires on a detected runtime fault, and none
  propagates to a peer.
- **Upward chain: absent.** `/health` is a constant flipped by phase order, the presence
  record carries an empty map, and no node reports health upward.

## Keys

**Actor.** B1 the observer layer. B2 a privileged insider. B3 an external actor. The admin
plane carries no authentication on any route, so a B3 with network reach to port 9990 acts
with B2 reach. That rule applies to every finding that names the admin port.

**Entry points.** Every threat and chain below names the positions an actor starts from.

| Code | Entry point | Who starts here |
|---|---|---|
| EP1 | Admin port. Network reach to the admin HTTP server, port 9990 by default, no credentials. Includes the metrics exporter served on the same plane. | B3, acting with B2 reach |
| EP2 | Service port. The external HTTP or Thrift port that customer requests arrive on. | B3 as a caller |
| EP3 | Mesh position. A peer the service connects to, a downstream endpoint that answers its requests, a peer that answers its pings, or a position on the path between them. | B3 with a foothold in the mesh, or B2 |
| EP4 | ZooKeeper. Write access to, or influence over, serverset membership and resolution. | B3 into ZooKeeper, or B2 with write |
| EP5 | Code, config, flags, deploy. A change to source, a flag, a stack param, a module binding, or an injected override. | B2 |
| EP6 | No actor entry. The gap is reached by a fault in the observer layer itself. | B1 |
| EP7 | Telemetry collector. Reach to the span collector that tracing ships to. | B2 or B3 at the collector |
| EP8 | Input data. The notes, ratings, enrollment, and status-history TSVs the pipeline reads. | B3 contributors and raters; B2 with file access |
| EP9 | Previous outputs. The prior run's status history, scored notes, and intermediates under the pipeline's output directory. | B2 with file access; B1 fault |
| EP10 | Pipeline CLI. Flags and arguments of the scoring runner. | B2 |
| EP11 | Sibling. The note writer, the live-note generator, or the post scorer. | B3 post author; a third-party provider |
| EP12 | Export. The daily data export that produces the pipeline's inputs, off-tree. | B2 on the far side; B3 whose data populates it |
| EP13 | API. The X API paths the note writer calls, served off-tree. | B3 shaping API traffic; B1 fault on the API side |

**Evidence paths.** `F/` is finagle (`finagle-core/src/main/scala/com/twitter/finagle` and
its sibling modules). `T/` is twitter-server (`server/src/main/scala/com/twitter/server/`).
finatra paths use module shorthands: `http-server` is
`http-server/src/main/scala/com/twitter/finatra/http`, `http-core` is
`http-core/src/main/scala/com/twitter/finatra/http`, `thrift` is
`thrift/src/main/scala/com/twitter/finatra/thrift`, `inject-server` is
`inject/inject-server/src/main/scala/com/twitter/inject/server`. The finagle tables use
file abbreviations: TFD `ThresholdFailureDetector`, SPM `ServerPingManager`, MCS
`MuxClientSession`, MSS `MuxServerSession`, FAF `FailureAccrualFactory`, FailFast
`FailFastFactory`, FD `FailureDetector`, ZK `Zk2Resolver`, STB `Stabilizer`, HS
`HealthStabilizer`, MET `Metrics`, JX `JsonExporter`, CSH `ChannelStatsHandler`, RZT
`RawZipkinTracer`, ST `SamplingTracer`, SMP `Sampler`. Community Notes paths are relative
to that repository: a file named without a directory lives in `scoring/src/scoring/`, and
sibling-project files are named by their project.

**Halt on fault.** The required control for every finding is the same: the observer alerts
and the system halts until the fault clears or the actor leaves. The column records what
the code does instead.

## Part A: Threats inside each service

### A1. finagle

#### Spoofing

| Component | Actor | Entry | Threat | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|---|
| ThresholdFailureDetector, ServerPingManager | B3 | EP3 | Any `Rping` resolves the outstanding ping with no tag match, and the server echoes `Rping` for any `Tping` without checking service health. | A peer or on-path actor answers pings and holds the detector Open while the service is dead; the client keeps dispatching customer requests to it. | TFD:70-77, MCS:254-261, SPM:20-25 | Continues. |
| FailureAccrualFactory | B3 | EP3 | An endpoint whose responses the classifier marks Successful is never suspected, whatever the content. | A compromised endpoint returning well-formed replies stays Alive and keeps taking customer requests. | FAF:404-406, :383-384 | Continues. |
| Balancer, PanicMode | B3 | EP3 | `bestOf` status lets one Open node mask a degraded set; after `maxEffort` misses the verified pick is discarded for an unchecked `dist.pick()`. | Customer requests route to unhealthy nodes while the balancer reports healthy. | Balancer:96, :235 | Continues. |
| FailFastFactory | B3 | EP3 | An endpoint that accepts one probe connection flips to Ok; an exhausted backoff also flips to Ok with no successful check. | An endpoint that accepts a bare connect but serves errors is marked healthy and receives customer requests. | FailFast:190-192, :199-201 | Continues. |
| Serversets | B3 into ZooKeeper, or B2 with write | EP4 | The observer serves ZooKeeper-supplied addresses it cannot verify; the Stabilizer returns the last good state indefinitely. | An actor who influences ZooKeeper membership redirects customer traffic; stale membership persists unverified. | ZK:320-325, STB:10-16, STB:58 | Continues. |

#### Tampering

| Component | Actor | Entry | Threat | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|---|
| FailureDetector | B2 | EP5 | Setting `sessionFailureDetector` to `none` or any unparsable value silently installs `NullFailureDetector`, always Open, never closes. | The liveness observer is removed for every session with one flag; dead sessions read healthy. | FD:60-65, :180-185, :31-34 | Continues. |
| FailureAccrualFactory | B2 | EP5 | `Param.Disabled` drops the module, `Param.Replaced` swaps it, a lenient `ResponseClassifier` redefines failure. | One param removes or blinds the circuit breaker. | FAF:226-227, :223-224, :197 | Continues. |
| Balancer | B2 | EP5 | `update(newFactories)` replaces the entire node set with no caller check; `dist` is a `protected var`; `close` strips the gauges. | An actor swaps the routed endpoint set or removes the observer in one call. | Balancer:167-210, :104, :256-259 | Continues. |
| FailFastFactory | B2 | EP5 | `FailFast(false)` removes the factory entirely. | One flag removes connection-failure observation per endpoint. | FailFast:82-83 | Continues. |
| TimeoutFilter | B2 | EP5 | Default `Duration.Top` omits the filter; `DeadlineOnlyToggle.unsafeOverride` is a public global mutator; the timeout is a per-request `Tunable`. | The request-time observer is disabled globally, or its threshold changed per request. | TimeoutFilter:91, :185-186, :32, :68 | Continues. |
| StatsFilter | B2 | EP5 | A null `StatsReceiver` removes the filter; a failure flagged `Ignorable` is excluded from every stat. | The observer is removed with one param; flagged failures leave no record. | StatsFilter:73-74, :299-302 | Continues. |
| Metrics | B2 | EP5 | A same-name gauge registration silently replaces the existing gauge; `unregisterGauge` removes the gauge, its schema, and its reserved name in one call. | An actor overwrites or deletes a metric with one call and no signal. | MET:293-300, :309-314 | Continues. |
| ExpiringService | B2 | EP5 | Default `Param(Duration.Top, Duration.Top)` leaves the module unwrapped; one `close` cancels both timers. | Idle and lifetime observation is off by default and removable in one call. | ExpiringService:24, :44-45, :191-195 | Continues. |
| ChannelStatsHandler | B2 | EP5 | `TcpStatsUpdater.cancel()` disables the poll in one call; no `handlerRemoved` override, so pipeline removal is unobserved. | Channel observation is removed with no resistance and no signal. | CSH:57-60; no `handlerRemoved` in :90-236 | Continues. |

#### Repudiation

| Component | Actor | Entry | Threat | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|---|
| MuxClientSession, MuxServerSession | B1 | EP6 | A drained or cleanly shut session logs nothing on the clean path and waits for an external `close`. | An observer's death leaves no record that crosses a boundary. | MCS:223-226, :241; MSS:172-174 | Continues. |
| FailureAccrualFactory | B2 | EP5 | Removal via `Disabled` emits no signal; the base `didMarkDead` hook is empty. | Disabling the breaker produces no trace. | FAF:226-227, :357 | Continues. |
| StatsFilter, Metrics, Zipkin | B1 | EP6 | Detections end in a passive sink; a throwing gauge is dropped with a log; error annotations ride a span that may never ship. | No detection reaches an independent boundary that acts on it. | StatsFilter:315; MET:327-330; JX:204; RZT:59-72, ST:48-52 | Continues. |
| Serversets | B2 | EP4, EP5 | The Stabilizer rewrites `Failed` to the prior `Bound` or to `Neg` and never emits `Addr.Failed`. | A failure signal never leaves the Stabilizer. | STB:58-60 | Continues. |

#### Information disclosure

| Component | Actor | Entry | Threat | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|---|
| MuxClientSession | B3 | EP3 | `handleDispatch` admits requests reading only `isDraining`, so dispatches continue over a session the detector marked Closed. | Customer payloads reach a peer the observer has flagged dead. | MCS:166-169 vs :208 | Continues. |
| FailureAccrualFactory | B3 | EP3 | Fail-open sends requests, customer payloads included, to an endpoint already marked dead. | Customer data reaches a dead or compromised endpoint. | FAF:263-264, :442-443 | Continues. |
| Balancer, PanicMode | B3 | EP3 | Panic mode dispatches to a node already judged unhealthy. | Customer requests and their data reach an unhealthy node. | Balancer:235, :246 | Continues. |
| Metrics, JsonExporter | B3 | EP1 | The exporter is an HTTP service that returns internal metrics to any caller reaching its bind address, and logs the remote address of malformed period requests. | Internal telemetry reaches an external actor. | JX:89-90, :140-170, :158-161 | Continues. |
| Zipkin tracing | B2 or B3 at the collector | EP7 | Error annotations carry request detail into spans shipped to an external collector. | Diagnostic data, with possible customer detail, egresses. | RZT:59-72, :185 | Continues. |

#### Denial of service

| Component | Actor | Entry | Threat | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|---|
| Balancer, PanicMode | B3 | EP3 | `pick` returns null when every node is non-Open, then `apply` dispatches to an unchecked pick anyway; Paranoid mode (`maxEffort` 0) skips the check on every request. | The balancer serves through the fault; unhealthy nodes take live customer traffic. | Balancer:218-236, :246; PanicMode:51, :63 | Continues. |
| FailureAccrualFactory | B3 | EP3 | An endpoint that goes silent is unnoticed until traffic is sent; the breaker fails open and revives on one probe. | Silent endpoints escape detection; dead endpoints keep taking traffic. | FAF:399-419, :263-264, :345 | Continues. |
| MuxClientSession | B3 | EP3 | One `Tdrain`, or one `Tlease` with a past deadline, from the peer takes the session out of service. | A peer removes the session with a single message. | MCS:270-273, :278-281, :215 | Continues. |
| MuxServerSession | B3 | EP3 | The server never pings, so a silent client is never detected, and a peer that withholds `Rdrain` holds the session until the drain deadline. | A client stalls the server session with no server-side liveness check. | MSS:104-105, :145-160 | Continues. |
| Serversets | B3 or B2 | EP4, EP5 | The Stabilizer serves the last good state indefinitely; HealthStabilizer hides unhealthy status for at least one probation period. | Stale routing persists through a fault; unhealthy status is masked from callers. | STB:14-16, :58; HS:15-16, :43-49 | Continues. |
| ChannelStatsHandler | B2 or B3 | EP3, EP5 | On a caught exception the handler counts, logs, and forwards; the TCP poll swallows `ChannelException`. | Channel faults produce a counter and a log, never a halt. | CSH:193-206, :74-78 | Continues. |

#### Elevation of privilege

| Component | Actor | Entry | Threat | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|---|
| TimeoutFilter | B3 | EP2 | With `preferDeadlineOverTimeout`, a caller-supplied `Deadline` replaces the configured timeout. | An upstream caller sets the observer's own threshold. | TimeoutFilter:412, :440 | Continues. |
| All Catastrophic observers | B2 | EP5 | A privileged actor disables or replaces any observer (detector `none`, `FailFast(false)`, FailureAccrual `Disabled`, null `StatsReceiver`, `setSampleRate(0)`, balancer `update` or `close`) and passes with a warning at most. | A privileged actor passes every critical observer with no alert and no halt. | FD:180-185; FailFast:82-83; FAF:226-227; StatsFilter:73-74; SMP:45-58; Balancer:167, :256 | Continues. |

### A2. twitter-server

#### Spoofing

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| `/health`, `ReadinessHandler` | B3, B2 | EP1, EP5 | A new component is attested healthy without being looked at. `/health` returns a constant from trait construction, the same for every component. | `T/Lifecycle.scala:33-39`, `:36`; `T/handler/ReplyHandler.scala:9` | Continues. Constant OK. |
| `NumberOfStatsReceiversRule`, collision and gauge rules | B2 | EP5 | A stats receiver loaded after premain gets no rule. The rule closes over a snapshot taken once at premain. | `T/lint/NumberOfStatsReceiversRule.scala:8-9`, `:23`; `T/lint/MetricsCollisionsRules.scala:12-18`; `T/lint/TooManyCumulativeGaugesRules.scala:13-22` | Continues. |
| `ClientRegistryHandler`, `ServerRegistryHandler` | B2, B3 | EP5 to register, EP1 to read | A client or server registered with an empty name is skipped by both handlers and by the index, so an unlabeled component is invisible. | `T/handler/ClientRegistryHandler.scala:104`; `T/handler/ServerRegistryHandler.scala:86`; `T/AdminHttpServer.scala:309`, `:320` | Continues. |

#### Tampering

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| `/ready`, `Lifecycle.Warmup` | B2 | EP5 | State is verified once and never re-checked. `/ready` returns a boolean fixed at construction; `Warmup` swaps handlers on lifecycle events and after the swap nothing re-verifies. | `T/handler/ReadinessHandler.scala:12-14`; `T/Lifecycle.scala:89-92`, `:103-106` | Continues. |
| Lint rules, `LoggingRules` | B2 | EP5 | A bad state introduced after the last pull persists until the next pull. Two logging rules are constants decided once at premain and never re-evaluated. | `T/handler/LintHandler.scala:23-26`; `T/lint/LoggingRules.scala:42-60`; `T/AdminHttpServer.scala:223`, `:230` | Continues. |
| `ThreadsHandler` | B2, B3 | EP1 | The JSON path hides deadlocks. Deadlock ids are computed only on the HTML path; a machine client polling `threads.json` never sees a deadlock. | `T/handler/ThreadsHandler.scala:43-57`, `:78`, `:108-111` | Continues. |
| `AdminHttpServer` admin route table | B2 | EP5 | A route replaced through `addAdminRoutes` or the global `HttpMuxer` is not noticed; later routes override earlier ones and `combine` picks the longest pattern per request. | `T/AdminHttpServer.scala:254-257`, `:235-236`, `:157`, `:299` | Continues. |

#### Repudiation

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| `AdminHttpServer` bind path | B2 | EP5 | If the plane never binds, `adminHttpServer` stays `NullServer` and the muxer stays a 404 stub. Nothing in the package records the absence. | `T/AdminHttpServer.scala:216`, `:207-211`, `:245-249` | Continues. |
| `LintHandler`, `FailedLintRuleHandler` | B2 | EP5 to remove, EP1 to read | Zero rules run renders as a normal page; an empty failed-lint set renders an empty string. Absence of the observer looks like a clean bill. | `T/handler/LintHandler.scala:82`, `:133-134`; `T/handler/FailedLintRuleHandler.scala:39` | Continues. |
| Registry and metric handlers | B2, B3 | EP5, EP1 | An empty registry folds to an empty page with 200 OK. Total loss of what is watched is indistinguishable from nothing yet. | `T/handler/ClientRegistryHandler.scala:130-131`; `T/util/MetricSource.scala:35-38`; `T/handler/MetricQueryHandler.scala:57` | Continues. |
| `AdminHttpServer.loggingMonitor` | B2 | EP1, EP5 | A routed handler fault becomes a single log line and a `false` return. Nothing reaches another observer. | `T/AdminHttpServer.scala:359-364`, `:371` | Continues. |
| `ContentionHandler` | B2 | EP5 | If a `SecurityManager` denies the detector, the endpoint returns 200 OK with an explanatory string. A policy change removes the detector while the endpoint stays green. | `T/handler/ContentionHandler.scala:15-26`, `:49-53` | Continues. |

#### Information disclosure

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| `ServerInfoHandler` | B3 crossing into B2 | EP1 | Every system property and environment variable is dumped, unauthenticated. Environment variables commonly carry secrets. | `T/handler/ServerInfoHandler.scala:33-51`, `:67-74` | Continues. |
| `HeapResourceHandler`, `ThreadsHandler` | B3 crossing into B2 | EP1 | Heap and thread dumps expose process memory, unauthenticated. A heap dump of a request-serving process contains in-flight customer data. | `T/Admin.scala:120-121`; `T/handler/ThreadsHandler.scala:44`, `:62` | Continues. |

#### Denial of service

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| `ShutdownHandler`, `AbortHandler` | B3 crossing into B2 | EP1 | One unauthenticated POST halts the process. `app.close` on shutdown, `Runtime.halt(0)` after 10 ms on abort. | `T/handler/ShutdownHandler.scala:25`, `:56`; `T/handler/AbortHandler.scala:23`, `:26` | Halts on inbound POST, wired to no detection. |
| Admin plane, `AdminThreadPoolFilter` | B2 | EP5 | The admin plane runs on a dedicated pool with admission control off, so it keeps reporting all clear while the work substrate stalls. The observation plane does not share the failure substrate of what it watches. | `T/filters/AdminThreadPoolFilter.scala:12-31`; `T/Admin.scala:261`; `T/AdminHttpServer.scala:373-375` | Continues, by stated design. |
| `/health`, admin plane | B2, B3 | EP5 | One `HttpMuxer.addHandler` at the health pattern replaces the handler; one override of `disableAdminHttpServer` removes the whole plane. | `T/Lifecycle.scala:90`; `T/AdminHttpServer.scala:198`, `:390-394` | Continues. |
| `MetricSource`, `MetricSchemaSource` | B2 | EP5 | Deny the `StatsRegistry` load once and every metric query throws or empties. The registry is a `lazy val` filled once from `LoadService`. | `T/util/MetricSource.scala:13`, `:47-52`; `T/util/MetricSchemaSource.scala:10` | Continues. |

#### Elevation of privilege

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| Every admin route | B3 crossing into B2 | EP1 | No route carries authentication. A network-reachable actor acts with the rights of a privileged insider. `remoteAddress` is logged and nothing else. | no auth in package; `T/handler/ShutdownHandler.scala:32`, `:45`, `:67`; `T/handler/AbortHandler.scala:24`, `:31` | Continues. Access is logged, not halted. |
| `AdminHttpServer`, all handlers | B2, B3 | EP1, EP5 | No reflexive response. On a handler fault the server logs and returns `false` and keeps serving, so an attacker acts in the log-and-continue window. | `T/AdminHttpServer.scala:359-364`; `:373-375` | Continues. |
| Admin server, `NullStatsReceiversRule` | B2 | EP5 | The observation plane carves itself out of its own check. The admin server runs with `NullStatsReceiver`, and the rule that would flag an unobserved server exempts the entry named `adminhttp`. | `T/AdminHttpServer.scala:369`; `T/lint/NullStatsReceiversRule.scala:16`, `:31` | Continues. |

### A3. finatra

Scored across four groups (the HTTP filter chain; the HTTP `ExceptionManager` and mappers;
`HttpWarmup` and `AdminHttpRouter` at startup; the Thrift chain and its exception parallel)
against the eight properties: 25 gaps, 7 satisfied. Substrate Dependency holds in all four
groups; Automatic Integration holds in three and fails at startup; every other property
fails in every group. The rows below record the gaps by STRIDE category.

| STRIDE category | Gaps | Property source |
|---|---|---|
| Tampering | 8 | Continuous State Verification (4), Self-Defense (4) |
| Elevation of privilege | 8 | System-Wide Response (4), Reflexive Response (4) |
| Denial of service | 4 | Heartbeat (4) |
| Repudiation | 4 | Death Signaling (4) |
| Spoofing | 1 | Automatic Integration (startup group) |

B2 is the actor for every gap: one code or config edit removes, reorders, or replaces any
observer, and no runtime check resists it. B3 triggers the runtime faults that the observers
convert to responses and benefits from continue-on-fault, but cannot remove an observer.

#### Tampering

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| HTTP filter chain order | B2 | EP5 | Filters are accepted in any order; `CommonFilters` fixes an order only by convention, so a reorder or omission changes what is observed with no runtime check. | `http-server/routing/HttpRouter.scala:171-175`; `http-server/filters/CommonFilters.scala:18-20` | Continues. |
| ExceptionManager mapper registry | B2 | EP5 | `add` overwrites any mapper, including the root, with no remove and no guard, so a replaced mapper redefines how faults are classified. | `http-core/exceptions/ExceptionManager.scala:136-139`, `:96-118` | Continues. |
| Per-request state | B2, B3 | EP5, EP2 | Observers verify nothing between requests; a state changed after the last request is not re-checked. | `http-server/filters/StatsFilter.scala:232-238` | Continues. |
| Thrift chain and exception parallel | B2 | EP5 | The Thrift path carries the same reorder-and-replace exposure. | `thrift/routing/routers.scala`; `thrift/exceptions/ExceptionManager.scala` | Continues. |

#### Repudiation

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| All runtime observers | B2 | EP5, EP6 | Every detection ends in a passive sink (StatsReceiver, Logger, MDC). No detection crosses an independent boundary that acts on it. | `http-server/filters/StatsFilter.scala:257-258`; `http-server/filters/AccessLoggingFilter.scala` | Continues. |
| Thrift ExceptionManager | B2 | EP5, EP6 | The thrift exception manager takes a `StatsReceiver` and never references it, so the thrift exception path emits no finatra metric at all. | `thrift/exceptions/ExceptionManager.scala:25`, `:27-85` | Continues. |

#### Denial of service

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| Runtime observers (Heartbeat) | B2 removes or never installs; B3 benefits | EP5, EP2 | Nothing runs between requests, so a silent failure is unnoticed until traffic arrives, and a removed observer leaves no liveness signal. | `http-server/filters/StatsFilter.scala:257-258`; `http-server/routing/HttpWarmup.scala:53-59` | Continues. |
| HttpWarmup | B2, B3 | EP5, EP2 | Warmup exercises only caller-constructed requests and no http-server code calls `send`, so it is a startup formality, not a live exercise of the path. | `http-server/routing/HttpWarmup.scala:53-59`, `:79-80` | Halts at startup only, into inject, non-propagating. |

#### Elevation of privilege

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| HTTP filter chain and ExceptionManager | B2 primary; B3 triggers | EP5, EP2 | No system-wide response and no reflexive halt. A detected fault becomes a 500 and the request is answered, so an attacker acts in the continue window. | `http-server/filters/StatsFilter.scala:257-258`; `http-server/filters/ExceptionMappingFilter.scala:17-22`; `http-core/exceptions/ExceptionManager.scala:86-89` | Continues. |
| Thrift chain | B2; B3 triggers | EP5, EP2 | The thrift `ThrowableExceptionMapper` re-raises unchanged; the fault reaches no peer and no halt. | `thrift/exceptions/ThrowableExceptionMapper.scala:18-20` | Continues. |

#### Spoofing

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| HttpWarmup (Automatic Integration, startup group) | B2 | EP5 | A component is attested ready by a warmup that exercises only caller-constructed requests, so a route can pass warmup without a real exercise of its handler. | `http-server/routing/HttpWarmup.scala:53-59` | Halts at startup only. |

#### Information disclosure

No finding scored in finatra's own source. One side observation, boundary-dependent and not
scored: the unhandled 500 body carries `toExceptionDetails(throwable)`
(`http-server/internal/exceptions/ThrowableExceptionMapper.scala:27`), and the
cancelled-request mapper carries elapsed deadline values, so what reaches a caller at EP2 on
an error depends on inject `ExceptionUtils`. The disclosure exposure at finatra's layer is
realized across the seams, recorded as FA2 and FA3 in Part B.

### A4. Community Notes

Scored across eight observer groups (input validation, rater-behavior detectors, model-fit
checks, status-gate rules, cross-run guards, contributor state, run orchestration, and the
siblings) against the eight properties: 61 gaps, 3 satisfied. The rows below carry the
pipeline's own entry vocabulary, which maps to the key as follows: input data is EP8,
previous outputs is EP9, CLI is EP10, sibling is EP11, code is EP5, none is EP6.

#### Default-off and dormant observers

Seven observers exist in the source and do not act. Each is a Self-Defense or Death
Signaling gap: the observer's absence produces no signal.

| Observer | State | Evidence |
|---|---|---|
| Flip check | Off by default. Prescoring hard-codes it off on both entry paths. When on, it returns without checking at or below 200 notes, and two of its churn caps are 1e7 and one is 1.0, with a TODO. | `runner.py:50`, `:361`; `run_scoring.py:2625`; `note_status_history.py:221`, `:237`; `constants.py:38`, `:56-60` |
| Dtype-drift patcher | Never armed. Would check dtype expectations on every concat, merge, join, and apply during execution. No caller in the tree. The `--enforce-types` flag is parsed and read only inside the uncalled patcher. Even when wired, it prints to stderr unless a fail flag is set. | `pandas_utils.py:684-733`, `:713`, `:176-177`; `runner.py:51-63` |
| Final-round train-error check | Commented out. The 0.09 threshold is defined; the call is commented out. The live check runs only in prescoring at 0.16, returns without checking below 200 notes, and raises with no re-fit. The documentation states a loss above 0.09 triggers a re-fit. | `mf_base_scorer.py:193`, `:1184`, `:627-629`, `:324-333`; `documentation/under-the-hood/ranking-notes.md:86` |
| Firm-reject consistency check | Logs with a TODO to make it an assert. A note that Core or Expansion firm-rejected and that still ends CRH appears as value counts in a log line. | `run_scoring.py:1095-1109`, `:1105` |
| Input fingerprints | Computed, never compared. Every one of twelve call sites passes the result into a log f-string. | `pandas_utils.py:41-51`; `mf_base_scorer.py:572`, `:602-608`; `run_scoring.py:1519`, `:1526` |
| URL checker | No caller. A tree-wide search returns only the definition. | `evaluator/url_evaluator.py:11-43` |
| Timing blocks | No threshold. Both log elapsed time in a finally block and act on nothing. | `constants.py:1206-1213`; `scorer.py:61-70` |

#### The lock-and-leave chain

A single sequence, each link verified, by which a note's public status leaves observation.

1. **Lock.** At a run where the note is unlocked, older than 14 days, and decided by a rule
   with locking enabled, the current status is copied into the locked status
   (`note_status_history.py:140-150`; `constants.py:41`). The condition is age plus a
   trusted decider, not 14 days of stable status.
2. **Pin.** The drift guard selects notes whose new label differs from the locked status
   and returns the locked status as the update (`scoring_rules.py:1608-1614`).
3. **Stabilization cleared.** Once locked, both stabilization timestamps are set to -1
   (`note_status_history.py:152-156`).
4. **Flip check excludes it.** The flip check prunes to rows with no lock timestamp
   (`note_status_history.py:223`).
5. **Rescoring excludes it.** The locking-eligible rescoring subset requires no locked
   status (`run_scoring.py:1911-1915`). Earlier subsets can still include a locked note,
   but the pin at step 2 overwrites its label.
6. **Inertia helps it get there.** CRH inertia keeps a note at CRH when its intercept sits
   just below threshold and it was CRH in the previous run, reading the previous status
   from the unverified history file (`scoring_rules.py:1559-1571`).

#### STRIDE

Actor key: **B1** the observer layer, **B2** a privileged insider with code, config, flag,
file, or deploy access, **B3** an external actor: a contributor, a rater, a post author,
an API note writer, or anyone shaping the input data. The model treats B2 and B3 both as
adversaries. Entry names where the actor starts: the input data (notes, ratings,
enrollment, status-history TSVs), the previous run's outputs, code, the CLI, a sibling
project, or none for a fault in the layer itself. The halt-on-fault column records what
the code does instead of alerting and halting.

#### Spoofing

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| Phase runners | B2 with write access to the output directory; B1 fault | Previous outputs | A stale or planted intermediate passes as this phase's input, so final and contributor scoring run on parameters, classifiers, and frames no one produced this run. | `runner.py:337`, `:391-401`, `:468-473` | None. Read as current. |
| Drift guard | B2 supplying the status file | Previous outputs | A forged locked status becomes the note's final status, and the run writes it back into the file it came from. | `scoring_rules.py:1608-1614`; `note_status_history.py:140`, `:148-150`; `run_scoring.py:2423-2430`, `:1104-1109` | None. Override and continue; the later check logs. |
| Rater-behavior detectors and status gates | B3, coordinated raters | Input data | Accounts that mimic independent raters push a chosen note to CRH; every gate on the path filters, demotes, or holds, and the run continues to a public status. | `process_data.py:553-555`; `post_selection_similarity.py:210-215`, `:257-258`; `scoring_rules.py:561-583`, `:606-628`, `:724-738`, `:1559-1571`; `note_status_history.py:140-150` | None. |
| Post-selection similarity phase | B2 | CLI | The skip flag writes an empty similarity frame, so the correlated-rater set is empty and no ratings are dropped, indistinguishable downstream from a clean rater population. | `runner.py:316-320`; `run_scoring.py:2590-2600`; `post_selection_similarity.py:210-215` | None. |
| Note writer (sibling) | B3, post author | Sibling | Post text is interpolated into every prompt and two substring tests are the only gate, so attacker-shaped note text is submitted to the API in test mode under the operator's credentials. | `template-api-note-writer/src/note_writer/write_note.py:58-62`, `:106`, `:145-156`; `main.py:37-44`; `submit_note.py:11`, `:18-27` | None. Submit. |
| Participant ID mapping | B1 fault; B2 with a crafted intermediate | Previous outputs | Integer ID columns skip the forward map and codes absent from the reverse map are left as integers, so a score or state is written against the wrong participant with no error. | `process_data.py:48`, `:52`, `:120-121` | None. Silent. |

#### Tampering

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| Rescoring passthrough | B2 | Previous outputs | Notes outside the rescoring set are copied verbatim from a supplied `scored_notes.tsv`, so most notes' public statuses and scores come from the supplied file. The loader expects a headerless file, which is what a tamperer would write; the runner's own header-bearing output fails loudly at load. | `run_scoring.py:2205-2213`, `:2351-2374`; `runner.py:241-250`; `process_data.py:197-201`, `:247-248` | None. |
| Status history timestamps | B2 | Previous outputs | Edited stabilization, lock, and current-label timestamps skip the stabilization hold, hide notes from the flip check, and drop notes out of rescoring. | `scoring_rules.py:829-837`, `:907-913`, `:999-1007`; `note_status_history.py:223`; `run_scoring.py:1805-1813`, `:1834-1842`, `:1911-1936` | None. |
| Status history author column | B2 | Previous outputs | The author on a row whose note is absent from the notes file is unchecked, because the consistency asserts compare inner-join counts against the notes count, and that column drives per-author CRNH totals and earn-out. | `note_status_history.py:31-37`, `:66-78`; `process_data.py:448-450`, `:477-479`; `note_ratings.py:327-340`; `run_scoring.py:1190-1196`; `contributor_state.py:81-87`, `:564-576` | None. |
| Ratings directory | B2 with filesystem access | Input data | Every `.tsv` in the directory is concatenated, so injected ratings on rater-note pairs not already present reach every scorer. Exact duplicate rows are dropped silently; a conflicting duplicate pair raises. | `process_data.py:279-292`, `:502-508` | Raise only on a duplicated rater-note pair. |
| CLI flags | B2 | CLI | Flags remove the flip check, strict columns, similarity filtering, and full ratings, choose the scorer set, and forge "now", which moves lock eligibility and stabilization. No flag is checked against a known-good set. | `runner.py:44-50`, `:115-120`, `:136-142`, `:195-214`, `:263-265`; `note_status_history.py:142`; `scoring_rules.py:913`; `run_scoring.py:1923`, `:2495-2500` | None. |
| Constants | B2 with code access | Code | Churn caps, the 200-note threshold, and scorer thresholds change what every detector does, with no observer on the values. | `constants.py:38`, `:51-60`; `run_scoring.py:105-272` | None. |
| Enrollment TSV | B2 | Input data | An enrollment state of removed, or an API state, exempts a participant from every transition predicate, freezing them in or out. | `contributor_state.py:14-25`, `:40`, `:61`, `:82`, `:100`, `:521-539`; `run_scoring.py:1338-1341` | None. |
| Intermediate pickles | B2 with write access to the intermediates directory | Previous outputs | The topic, pflip, pcrh, and meta artifacts are deserialized with joblib before any type check, giving code execution inside the final and contributor phases. | `runner.py:397-400`; `process_data.py:900-916` | None. Type asserts run after the load. |
| Shared memory | B1 fault; B2 on the host | None | The explicit close and unlink sit after the blocking result collection, so on a worker exception the run's ratings, enrollment, status history, and note topics stay in named segments. Workers open segments by name and parse with no integrity check. | `run_scoring.py:472-476`, `:488-491`, `:494-546`, `:613`, `:616-621` | None. Cleanup skipped on the error path. |

#### Repudiation

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| Status history writer | B2 | CLI | "Now" comes from the clock or from a flag, and no signature, hash, or provenance marks the file, so a run can backdate or postdate locks and transitions with no record that distinguishes it. | `note_status_history.py:344-350`; `runner.py:263-265`; `constants.py:34-35` | None. |
| Phase runners | B1 fault | None | Completion is a log line and the presence of files, so the next phase cannot tell a finished phase from an aborted one whose earlier files remain. | `runner.py:326`, `:381`, `:458`, `:506`, `:337`, `:391-401`, `:468-473` | None. |
| Flip check | B1 | None | Failure detail exists only in the assertion string, and the metrics dict that would carry the numbers is discarded. | `note_status_history.py:279-297`; `run_scoring.py:2459-2479`, `:1993`, `:2218`; `runner.py:411` | Raise, no durable record. |
| Rater-behavior detectors | B1 | Input data | Which raters were tagged high volume is recorded only in log lines and no output column. Per-rater similarity and clique values are written only into the prescoring rater intermediate, not into the four public outputs. | `process_data.py:575-581`; `constants.py:96`, `:1089`, `:1096`, `:1106-1154`; `post_selection_similarity.py:216-218`; `run_scoring.py:1595-1597` | None. |
| Firm-reject consistency check | B1 | None | A note that Core or Expansion firm-rejected and that still ends CRH appears only as value counts in a log line, with no per-note record. | `run_scoring.py:1095-1109` | None. |
| Note writer (sibling) | B1; B2 | Sibling | The submit response is discarded and the only record is a print, so nothing durable says what was submitted under the account. | `template-api-note-writer/src/cnapi/submit_note.py:38`; `main.py:37-49` | None. |

#### Information disclosure

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| Prescoring start | B2 with log access; B1 | None | Every environment variable is printed to stdout at the start of prescoring, so any secret in the process environment lands in the logs. | `run_scoring.py:1496-1498` | None. |
| Output files | B3, anyone with the download | None | One row per rater carries the viewpoint factor, intercept, enrollment state, earn-out count, state-change and earn-out timestamps, rating counts, top not-helpful tags, and 14-day and 90-day note totals, keyed by public ID. Joined to notes and ratings this profiles pseudonymous participants. No observer bounds what is written; the validators require every listed column. | `constants.py:1106-1154`, `:730-755`, `:881-910`, `:943-1059`, `:640-659`; `runner.py:438-443`, `:497-502`; `run_scoring.py:1429-1450` | None. |
| Prescoring rater intermediate | B2 with read access to the output directory | None | Public rater IDs are restored onto the prescoring rater frame before it is written to the intermediates directory, and that frame carries rater intercepts, factors, reputation, and the per-rater similarity and clique values. | `runner.py:368-374`; `run_scoring.py:1595-1597`; `constants.py:1078-1097` | None. |
| Flip check failure message | B2 with log access | None | The assertion string samples up to 30 status-history rows including the author column, which holds the real participant ID where input IDs were numeric and a per-run code where they were factorized. | `note_status_history.py:274-297`; `process_data.py:103-108`, `:848-856`; `runner.py:438-443` | Raise. |
| Shared memory | B2 on the host | None | Ratings, enrollment, status history, note topics, and the prescoring frames sit in named segments any process with the name can read and parse, and the error path skips cleanup. | `run_scoring.py:472-476`, `:488-491`, `:494-546`, `:616-621` | None. |
| Note writer (sibling) | Third party; B3 reach | Sibling | Post text, image URLs, and the quoted and replied-to context are sent to an external API with the key read from the environment, with no observer on egress. | `template-api-note-writer/src/note_writer/llm_util.py:11-16`; `write_note.py:58-62`, `:116`, `:148`, `:151`; `cnapi/get_api_eligible_posts.py:49-57` | None. |
| Live-note generator (sibling) | B2 with log access | Sibling | Full prompts and responses containing note contents are logged at INFO. | `collaborative-note-generator/live_note_generator.py:686-692`, `:801-808` | None. |
| Entry point | B2 | Code | The root logger is set to DEBUG, so records from libraries and from any logger without its own level are emitted. The module loggers set INFO and their frame summaries carry dtypes and byte counts only. | `scoring/src/main.py:30`; `process_data.py:18`, `:226-228`; `pandas_utils.py:59-88` | None. |

#### Denial of service

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| Scorer runner | B1 fault; B3 with data that slows convergence | Input data; none | A hung worker holds the phase open with no timeout, and the training loop has no wall-clock bound, so no new statuses are produced and nothing alerts. | `run_scoring.py:613`; `matrix_factorization/matrix_factorization.py:495-497` | None. Wait. |
| Flip check | B3, coordinated ratings causing mass flips | Input data | Churn is unmeasured by default. When enabled, two subsets carry caps of 1e7 and one carries 1.0, so a mass status change ships to the public. | `runner.py:50`; `constants.py:51-60`; `note_status_history.py:221`, `:237`, `:277-297` | Only with the flag, and only after scoring completes. |
| Scorers | B3 shaping ratings so a scorer's input empties; B1 | Input data | A scorer whose ratings empty after filtering returns empty output, and meta-scoring proceeds from the NMR default with that model silently absent from the decision. | `scorer.py:318-319`, `:325-327`, `:386-402`, `:416-420`, `:430-432`; `mf_base_scorer.py:592-600`, `:748-751`, `:1155-1156`, `:1493-1498`; `run_scoring.py:836-838`; `scoring_rules.py:669-684` | None. Continue. |
| PCRH and pflip plausibility models | B1 | None | An unfitted PCRH model returns empty predictions with a warning, and pflip drops training labels that lack scoring cutoffs with a warning, so those plausibility gates are absent without a halt. | `pcrh_model.py:696-698`, `:785-792`; `pflip_plus_model.py:1105-1131` | None. |
| Small-input mode | B2 | CLI; input data | At or below 200 notes the high-volume flag is forced false, the similarity rating filter is skipped, the flip check returns without checking, and the train-error check returns without checking, so a sampled or small run carries no rater-behavior observation. | `constants.py:38`; `process_data.py:553-555`; `post_selection_similarity.py:257-258`; `note_status_history.py:221`, `:237`; `mf_base_scorer.py:324-327` | None. |
| Prescoring flip check | B2 with code access | Code | Prescoring passes the flip check off on both entry paths, so the prescoring rescoring pass never runs. | `runner.py:361`; `run_scoring.py:2625`, `:1663-1697` | None. |
| Note writer (sibling) | Third party; B3 | Sibling | No timeout on the HTTP call or the subprocess, so a slow provider holds the run; a tag failure after three retries propagates out of the worker and ends the run. | `template-api-note-writer/src/note_writer/llm_util.py:16`; `xurl_util.py:12`; `misleading_tags.py:14-23`; `main.py:20`, `:78-81` | Exception, no alert. |
| Live-note generator (sibling) | Third party | Sibling | Up to ten retries per LLM gate per post multiply cost against a failing provider; the failure path returns a rejection. | `collaborative-note-generator/live_note_generator.py:88`, `:562`, `:682`, `:797`, `:539-545` | Continue. Fail-closed at the reject gate. |

#### Elevation of privilege

| Component | Actor | Entry | Impact | Evidence | Halt on fault |
|---|---|---|---|---|---|
| Every observer | B2 | CLI | A privileged actor passes every critical observer using flags, with no alert and no halt. | `runner.py:44-50`, `:51-63`, `:136-142`, `:195-214`, `:263-265` | None. |
| Every observer | B2 with code access | Code | Any assert is removed by one edit, and nothing watches the observers; the only ordering check covers declared rule dependencies. | `scoring_rules.py:120-122`, `:1707-1708` | None. |
| Status lock | B3 | Input data | A status held at a run more than 14 days after note creation, decided by a locking-enabled rule, is copied into the locked status; from then the drift guard overwrites any new label with it and the flip check no longer covers the note. | `note_status_history.py:140-150`, `:152-156`, `:223`; `scoring_rules.py:1608-1614`; `run_scoring.py:1911-1915` | None. |
| Live-note generator (sibling) | B3, with the payoff inside B1 | Input data, then sibling | The scorer's per-version status gates generation, so a manipulated status changes whether a post is eligible for a regenerated live note. | `collaborative-note-generator/live_note_generator.py:181-194` | None. |
| Note writer (sibling) | B3 crossing into B2 | Sibling | Post text drives an LLM whose output is submitted under the operator's account; the mode is a literal in the caller, so a one-line change moves attacker-shaped notes out of test mode. | `template-api-note-writer/src/note_writer/write_note.py:58-62`, `:151-168`; `main.py:41`; `cnapi/submit_note.py:11`, `:18-27` | None. |
| Intermediate loading | B2 with file write | Previous outputs | Planted joblib files are unpickled inside the scoring process. | `runner.py:397-400`; `process_data.py:900-916` | None. |
| Participant ID mapping | B2 | Previous outputs | Integer columns bypass the forward map and unknown codes pass through on restore, so scores and states land on chosen participants. | `process_data.py:48`, `:52`, `:120-121` | None. |

## Part B: Threats across the boundaries between services

### B1. finagle and twitter-server

| # | Threat | Actor | Entry | Impact | twitter-server evidence | finagle evidence |
|---|---|---|---|---|---|---|
| CP1 | Topology disclosure | B3 | EP1 | The unauthenticated client registry maps the mesh: every downstream name, dest address, full stack params, and resolved socket addresses. An external actor reads the downstream topology with no foothold. | `T/handler/ClientRegistryHandler.scala:119`, `:145-159`; `T/view/EndpointRegistryView.scala:33-56` | `F/client/ClientRegistry.scala:14`, `:135-137`; `F/client/EndpointRegistry.scala:36-44` |
| CP2 | Balancer health | B3 | EP1 | The unauthenticated balancer readout exposes live per-node health: `numAvailable`, `numBusy`, `numClosed`, `status`, `panicMode`, `totalLoad`. An external actor learns which nodes are healthy and how near each balancer sits to panic. | `T/handler/LoadBalancersHandler.scala:20-26`, `:33-36`; `T/view/BalancersJsonView.scala:20-34` | `F/loadbalancer/BalancerRegistry.scala:25-26`; `F/loadbalancer/Metadata.scala:27-59` |
| CP3 | Attached clients | B3 | EP1 | The unauthenticated attached-clients readout returns each peer's socket address, TLS session id, cipher suite, and peer certificate common name. An external actor enumerates who is connected and the TLS posture of each link. | `T/handler/AttachedClientsHandler.scala:42-64`, `:96-99` | `F/server/ServerRegistry.scala:22-33`, `:76-79` |
| CP4 | Metrics snapshot | B3 | EP1 | The unauthenticated metrics snapshot pulls every finagle counter and gauge, including `loadbalancer/available`, `size`, `load`, and the `panicked` counter. An external actor reads finagle's internal state even without the registry handlers. | `T/util/MetricSource.scala:13`, `:31-41` | `F/stats-core/JsonExporter.scala:82-90`; `F/loadbalancer/Balancer.scala:148-160` |
| CP5 | Observation switched off | B3 | EP1 | twitter-server hosts the disclosure and halt plane on a finagle server, then runs it with `NullStatsReceiver`, `NullTracer`, and admission control off. finagle cannot throttle or record abuse of the admin port. | `T/AdminHttpServer.scala:367-377`, `:373-375` | `F/filter/ServerAdmissionControl.scala:53-56`, `:81-86` |
| CP6 | Fault does not halt | B3 or B2 | EP1, EP5 | The one finagle-observes-twitter-server edge fails to halt. When an admin handler throws, `MonitorFilter` catches it, logs it, and returns `false`. The admin connection keeps serving. A fault in the observation plane never crosses into a halt. | `T/AdminHttpServer.scala:359-363`, `:371` | `F/filter/MonitorFilter.scala:56-70`; `F/util/Monitor.scala:97` |

### B2. finatra on twitter-server and finagle

| # | Threat | Actor | Entry | Impact | Evidence |
|---|---|---|---|---|---|
| FA1 | Application controllers on the unauthenticated admin plane | B3 with admin-port reach, acting with B2 rights | EP1; EP5 for the missing filter | finatra hands user controllers to twitter-server as admin `Route`s or to finagle's global `HttpMuxer` under `/admin/finatra/`. Those handlers run under the admin server's config: null stats and tracer, admission control off. An installed auth filter survives; a missing one is never added. | `http-server/routing/AdminHttpRouter.scala:70-83`, `:109-135`; `http-server/routing/HttpRouter.scala:674` |
| FA2 | Route table and filter chain disclosed on `/admin/registry.json` | B3 | EP1 | `HttpRouter` registers every route's path, method, controller class, request and response class, capture names, and per-route filter, plus the global filter chain, into the util `GlobalRegistry`. `RegistryHandler` returns it to any caller. | `http-server/routing/HttpRouter.scala:672-692`; `http-server/routing/Registrar.scala:54-71`; `inject/inject-core/.../internal/LibraryRegistry.scala:60`, `:81`; `T/handler/RegistryHandler.scala:44-50` |
| FA3 | Per-route metrics name the same surface on `/admin/metrics.json` | B3 | EP1 | finatra's `StatsFilter` scopes counters and latencies as `route/<name>/<METHOD>/status/<code>` into the shared receiver, which twitter-server's `MetricSource` pulls. An external actor reads each route's error rate and latency. | `http-server/filters/StatsFilter.scala:171-191`; `T/util/MetricSource.scala:31-41` |
| FA4 | The dead-man's switch reads closure, not health, and nobody observes it | B1 | EP6 | `Awaiter.any` polls `Await.isReady` every second on finagle's `DefaultTimer`. Ready means closed. No finagle runtime observer closes a server. The latch has no timeout and no observer, and twitter-server's shutdown timer is the same object, so one dead timer thread removes both halts silently. | `inject-server/Awaiter.scala:3`, `:28-34`, `:38` |
| FA5 | The unauthenticated halt now closes the customer-facing servers | B3 with admin-port reach | EP1 | POST `/quitquitquit` calls `app.close`; finatra's `onExit` closes the external HTTP, HTTPS, and Thrift `ListeningServer`s; `Awaiter` sees one closed and releases `main()`. finatra adds no origin check and no second halt node. | `http-server/servers.scala:328-331`, `:350-351`; `thrift/servers.scala:207-208` |
| FA6 | Startup halts and hangs run behind a live admin plane | B3 reader; B2 or B3 into ZooKeeper as cause | EP1 to read; EP4 or EP5 to cause | The admin server starts at `premain` with `/health` at warming up before any `App.main` phase. The client-resolution gate is an untimed `Await.ready`. During a hang every unauthenticated admin route serves. The process never finishes starting and never halts. | `inject-server/TwitterServer.scala:193-205`, `:200`; `T/AdminHttpServer.scala:383-395` |
| FA7 | Health flips on phase order, after the external bind, with no check | B2 for the order; the reader is an orchestrator outside B1 | EP5 | `postWarmup` binds the external servers, then `afterPostWarmup` calls `warmupComplete()`, which swaps `/health` to OK. Nothing between bind and flip verifies the bound service answers. The optional warmup only adds a lint `Issue` when skipped. | `http-server/servers.scala:316-339`; `inject-server/TwitterServer.scala:321-329`; `T/Lifecycle.scala:103-106` |
| FA8 | Lower-layer faults become finatra responses | B3 | EP2, EP3 | A finagle rejection becomes a 503, a `Failure` a 500, an escaped `Throw` an empty 500; the thrift root mapper rethrows unchanged. finagle's server `StatsFilter` then counts the outcome. Three layers see the fault; none halts. | `http-server/filters/HttpNackFilter.scala:33-38`; `http-server/internal/exceptions/HttpNackExceptionMapper.scala:29`; `http-server/internal/exceptions/FailureExceptionMapper.scala:28-35`; `http-server/filters/StatsFilter.scala:251-258`; `thrift/exceptions/ThrowableExceptionMapper.scala:18-20` |
| FA9 | Namespace squat grants package-private reach | B2 (code submitter) | EP5 | `FinagleBuildRevision` lives in package `com.twitter.server.internal` inside finatra's tree; twitter-server has no `internal` directory. Scala `private[server]` resolves by package name, so a class placed beside it reaches twitter-server's `private[server]` members with no check. | `http-server/.../FinagleBuildRevision.scala:1`, `:10-18`; `T/AdminHttpServer.scala:147` |
| FA10 | One binding blinds both layers' server-side telemetry | B2 | EP5 | The same injected `StatsReceiver` and the same `HttpResponseClassifier` instance go to finatra's `StatsFilter` and to finagle's server stack. One override of `statsReceiverModule` or `configureHttpServer` rewrites or removes what both layers count. | `http-server/servers.scala:301-302`, `:404-406` |

### B3. Community Notes and the Scala stack

No code edge exists in any of the four trees. Searches re-run at verification: the names
communitynotes, community notes, and birdwatch over finatra, twitter-server, and finagle
return nothing; finagle, finatra, twitter-server, and `com.twitter` over Community Notes
return nothing; the seven Scala-side names Community Notes mentions return nothing in any
Scala tree and appear in Community Notes only inside comments, docstrings, or executable
code that cannot resolve (two API path literals handed to an absent CLI, an abstract client
with no subclass, two imports guarded by `except ImportError`). The `/2/notes` API path
appears in no Scala tree. No Community Notes file names an admin route, port 9990,
`/health`, or `/ready`. Community Notes has no test files of any kind.

**Seam table.** Every far side is inferred: the endpoint has no source in any tree and its
existence rests on a deployment fact.

| # | From | To | What crosses | Community Notes evidence | Observed by |
|---|---|---|---|---|---|
| Z1 | Daily data export | Pipeline input load | Notes, ratings, status history, enrollment, with participant IDs | `constants.py:64-67`; `runner.py:124-129`, `:267-276`; `process_data.py:279-292`, `:197-201`, `:393-405` | Pipeline: field count, column names, ID type. Nothing on date, signature, hash, or run id. Far side: nothing in any tree. |
| Z2 | Pipeline note outputs | Combine job, then a Scala schema class named in comments | One status per note, intercepts, timestamps, author IDs; column order pinned by index | `runner.py:445-447`; `constants.py:1036-1038`, `:1049-1050`; `run_scoring.py:1429-1440`, `:2490-2500`, `:1401-1410` | Pipeline: column set and list order when strict columns is on. No index assert. Far side: nothing. |
| Z3 | Pipeline contributor output, enrollment as integer Thrift code | An importer named in a docstring | Per-participant enrollment state, scores, counts | `constants.py:789-799`, `:1124`, `:1156`; `contributor_state.py:319-328`, `:586-595`; `runner.py:502` | Pipeline: column set only; the code map passes unknown strings through. Far side: nothing. |
| Z4 | An enrollment producer named in a docstring | Pipeline enrollment input | Enrollment state as a string, per participant | `contributor_state.py:321-324`, `:600-602`; `constants.py:813-822` | Pipeline: column names; rows without a rater id dropped in silence. Far side: nothing. |
| Z5 | Pipeline not-helpful tag list order | A Scala accessor named in a comment | List position of each tag | `constants.py:492-494`, `:526-528` | Nothing on either side. |
| Z6 | Note writer: GET eligible posts, POST a note, through an external CLI | The X API server | Posts with author id, text, media; note text and tags; the POST creates a note under the operator's account | `cnapi/get_api_eligible_posts.py:21-33`, `:49-57`; `cnapi/submit_note.py:9-13`, `:18-38`; `cnapi/xurl_util.py:11-13`, `:23-24`; `main.py:37-48` | Pipeline: exit code, JSON parse of stdout, asserts on referenced-post types. The POST return value is discarded. Far side: nothing. |
| Z7 | Live-note generator classification string | A note store described as thrift-side | Note text and classification | `live_note_generator.py:32-42`, `:818-832` | Pipeline: retries when the string is not canonical. Far side: nothing. |
| Z8 | The note store, through an abstract client | Live-note generator | The scorer's status per note version, note text, user IDs | `notes_data_client.py:14-16`, `:38-48`, `:67-78`; `live_note_generator.py:101-103`, `:183-184`, `:247-255` | Pipeline: a null test before reading status; a hydration failure returns a result and continues. Far side: nothing. |
| Z9 | Two optional internal Python modules | Pipeline child-process logging | Logging configuration; no customer data | `run_scoring.py:415-421`, `:580-589` | Nothing. Absence is swallowed at both sites. |

Direction tally: into the pipeline Z1, Z4, Z8, Z9; out of the pipeline Z2, Z3, Z5, Z7;
request and response Z6. On the pipeline side, eight rows carry a structural check of the
artifact that crosses. None reads the far side's health, date, provenance, or completion.
On the Scala side, nothing in any tree observes any of the nine crossings.

**Cycles traced.** The cross-run loop closes only off-tree: the writer emits
`note_status_history.tsv` and the reader defaults to `noteStatusHistory-00000.tsv`, and
nothing in the tree copies or renames one to the other. The deployment loop (outputs to
services to export to inputs) fails all four ring clauses: fewer than three mutually
observing nodes, input checks that report to the exception path, no coordinator, no
health signal. The note writer's request and response is two nodes and one shot with the
response discarded. The generator loop checks that a scoring result exists and what its
status is, never anything about the scorer. Between any in-tree Scala node and any
pipeline node: zero edges.

**Seam threats.** All eleven verified; far side inferred on every row.

| # | Threat | Seam | Actor | Entry | Impact | Evidence |
|---|---|---|---|---|---|---|
| CN1 | Positional schema contract, no index assert | Z2 | B2 with code access; B1 far-side schema change | EP5, EP9 | Comments pin index 88 and indices 97 and 98 to a Scala schema class. The validators assert set equality and reorder by list, never an index. Deprecated columns are dummy-filled to hold positions. Strict columns is one flag from off. No test file exists in the tree. | `constants.py:1036-1038`, `:1049-1050`; `run_scoring.py:1429-1432`, `:1401-1410`, `:2495-2500`; `runner.py:130-142` |
| CN2 | Unknown enrollment codes pass through in both directions | Z3, Z4 | B1 far-side change to the string set; B2 on the far side | EP8, EP12 | The code map has nine entries and returns any unmapped string unchanged. An unknown string is excluded from no transition predicate, because each excludes only named states. The output column is declared Int64 and the declaration is never applied. The enrollment log counts only codes 0 to 4. | `constants.py:789-799`, `:1124`, `:1156`; `contributor_state.py:21-25`, `:39-48`, `:60-68`, `:81-87`, `:99-107`, `:319-328`, `:586-595`, `:604-620`; `process_data.py:734-746` |
| CN3 | The export as an unverified input; corrupted rows dropped in silence | Z1, Z4 | B2 on the far side; B3 raters and authors; B1 fault | EP12, EP8 | The pipeline checks a field count and column names, then concatenates every TSV in the directory. Nothing dates, signs, hashes, or checks provenance. Rows without a rater id are dropped silently, with a comment about a corrupted dump. Nothing in twitter-server or finagle produces an integrity signal for a TSV. | `process_data.py:279-292`, `:197-201`, `:393-405`; `contributor_state.py:600-602`; `documentation/under-the-hood/download-data.md:32` |
| CN4 | The note-writer API path: exit code only, response discarded | Z6 | B3 post author; B3 shaping API traffic; B1 fault on the API side | EP11, EP13; EP2 inferred | Two API paths through an absent CLI. The POST return value is discarded. Success logs a fixed line and every exception is logged as a likely duplicate, so a far-side 5xx, a rejection, and a duplicate are one event. Test mode is a literal. | `cnapi/xurl_util.py:11-13`, `:23-24`; `main.py:37-48`; `cnapi/submit_note.py:9-13`, `:29-38` |
| CN5 | Public status as a control, read with no observation of the batch that produced it | Z2, Z8 | B3 coordinated raters; B1 | EP8, EP9, EP12 | The status leaves at a comment-pinned index with no marker of the run that produced it. The generator gates on that status through an abstract client, and a missing result counts as not CRNH. | `runner.py:445-446`, `:506`; `live_note_generator.py:183-184`, `:189` |
| CN6 | The batch timestamp written by the consumer and read by nobody | Z2 | B2 on the consumer, inferred; B1 | EP9, EP12 | The pipeline writes NaN into the final-scoring timestamp column with a comment naming the consumer that fills it. The column has no reader in the pipeline, so the only record of when a status was produced is written by the reader. | `run_scoring.py:2490-2491`; `note_status_history.py:101-102`; `constants.py:727`, `:752` |
| CN7 | Optional internal modules: a boundary the code cannot see | Z9 | B2 deploy; B1 | EP5 | Two imports pass on ImportError, so the code behaves identically with or without them and no observer can tell which mode a run is in. The same shape recurs in the generator's absent LLM client and the post scorer's absent constants module. | `run_scoring.py:415-421`, `:580-589`; `live_note_generator.py:56`; `score_posts.py:3` |
| CN8 | The cross-run loop does not close by filename in source | Z1, Z2 | B2 operator supplying the file; B2 on the far side, inferred; B1 | EP9, EP10, EP12 | The writer emits `note_status_history.tsv` and the reader defaults to `noteStatusHistory-00000.tsv`. Nothing in the tree copies or renames one to the other, so the loop closes only through an operator or the export. A third name for the scored-notes artifact is defined and never used. | `runner.py:445-447`, `:124-129`, `:88-93`, `:241-247`; `constants.py:63`, `:67` |
| CN9 | An abstract notes-data client with no implementation | Z8 | B2 whoever supplies it; B1 | EP11, EP5 | The generator raises without a client. No subclass exists, the scoring-result type is never constructed, and the status parser is never called, so the parse that gates generation happens entirely off-tree. A hydration failure returns a result and continues. | `notes_data_client.py:14-16`, `:38-48`, `:67-78`; `live_note_generator.py:101-103`, `:183-184`, `:247-255` |
| CN10 | Tag order pinned by a comment | Z5 | B2 with code access | EP5 | The not-helpful tag list order is pinned to a Scala accessor by comment. No test pins it and no reader in any tree would notice a change. | `constants.py:492-494`, `:526-528` |
| CN11 | The canonical classification set modeled in the generator | Z7 | B1 far-side change; B2 with code access | EP11, EP5 | The generator retries when a classification string is not in its canonical set, the only pipeline check that models a far-side behavior, against a set the far side can change with no signal. | `live_note_generator.py:32-42`, `:818-832` |

## Part C: Chains

Each chain joins a gap in one system to a gap in another. Every chain lists each entry point
an actor can start from, and marks which entries are required and which are alternatives.
Ranked by severity, headline first.

### C1. Route then dump

- **Entry points.**
  - EP1, required. The read step: the unauthenticated thread dump.
  - EP2 or EP3, optional. Either lets the actor trigger the faults the layers serve through
    instead of waiting for them. No position on the data path is required.
- **Step 1, finagle client layer.** finagle continues through a detected fault. Panic-mode
  routing dispatches to an unchecked pick when the verified pick fails, and fail-open accrual
  lets requests flow through a node already judged not open. Customer payloads stay resident
  in process memory instead of the process halting. `F/loadbalancer/Balancer.scala:228-246`;
  `F/liveness/FailureAccrualFactory.scala:262-266`, `:442-443`.
- **Step 2, finatra server layer.** finatra denies the halt again. A nack becomes a 503, a
  `Failure` a 500, an escaped `Throw` an empty 500 (FA8), and `Awaiter` reads only closure
  (FA4). A third keep-serving layer.
- **Step 3, twitter-server.** The unauthenticated dump reads process memory. `ThreadsHandler`
  returns every stack trace with frame detail, routed unconditionally at `/admin/threads`. A
  dump of a request-serving process holds in-flight customer data.
  `T/handler/ThreadsHandler.scala:43-64`; `T/Admin.scala:92-100`. The route table on
  `/admin/registry.json` (FA2) supplies the controller class names that let the actor match
  dump frames to requests.
- **Seam.** The admin plane runs inside the same finagle JVM as the finagle clients and the
  finatra request path (`T/AdminHttpServer.scala:367-377`;
  `F/server/ListeningStackServer.scala:117`).
- **Combined impact.** The actor denies the halt at two layers so customer data keeps
  flowing, then reads that data from the same host through the admin port. Actor: B3,
  starting from admin-port reach only.
- **Caveat.** The thread-dump path is unconditional. The heap-dump variant depends on the
  Heapster native agent, absent from this tree, so `HeapResourceHandler` returns a 500 when
  it is not loaded (`T/handler/HeapResourceHandler.scala:7`, `:18-23`). The heap path holds
  only where an operator has loaded that agent.
- **Composition.** finagle panic and fail-open findings, twitter-server heap and thread dump
  finding, FA8, FA4, FA2.

### C2. Recon then attack

- **Entry points.**
  - EP1, required. The reconnaissance step.
  - EP3, one branch. finagle-level attacks on the peers the recon revealed.
  - EP2, the other branch. Driving the service's own external port at the routes the recon
    revealed. Either branch completes the chain; both can run.
- **Step 1, twitter-server and finatra recon.** The unauthenticated client and endpoint
  registry returns downstream names, dest addresses, and resolved socket addresses (CP1;
  `T/handler/ClientRegistryHandler.scala:119`, `:145-159`). `/admin/registry.json` returns
  the route table, controller classes, and filter classes (FA2). `/admin/metrics.json`
  returns per-route error rates and latencies (FA3). `/admin/servers/<name>` returns the
  finatra server's stack.
- **Step 2, branch EP3, finagle peers.** With exact downstream addresses in hand, the actor
  applies finagle's detection gaps against those peers: fail-open accrual sends payloads to
  an endpoint marked dead; a spoofed ping loop holds a dead endpoint in service.
  `F/liveness/FailureAccrualFactory.scala:262-266`;
  `F/liveness/ThresholdFailureDetector.scala:64-78`.
- **Step 2, branch EP2, the external port.** The actor drives the external port at routes
  the registry shows unfiltered and at routes whose metrics already show 5xx, while finatra
  keeps the process up by converting faults (FA8).
- **Combined impact.** An external actor with no prior knowledge learns the API surface,
  filter posture, downstream mesh, and live per-route health, then attacks with no
  guesswork. The disclosure removes the reconnaissance cost that would otherwise gate the
  finagle findings. Actor: B3.
- **Composition.** CP1, FA2, FA3, finagle fail-open and ping findings, FA8.

### C3. Recon then panic

- **Entry points.**
  - EP1, required. The balancer readout and the live confirmation of panic.
  - EP3, required. The means to push healthy nodes unhealthy.
- **Step 1, twitter-server.** The unauthenticated balancer readout reveals per-node health
  and the panic posture for a target client (CP2; `T/handler/LoadBalancersHandler.scala:20-26`;
  `T/view/BalancersJsonView.scala:20-34`).
- **Step 2, finagle.** The actor pushes healthy nodes unhealthy until `pick(maxEffort)`
  fails, at which point the balancer discards the verified pick and dispatches to an
  unchecked `dist.pick()`. Customer requests route to a node judged not open.
  `F/loadbalancer/Balancer.scala:228-246`; `F/loadbalancer/PanicMode.scala:37-69`.
- **Combined impact.** The readout tells the actor how many nodes to knock out to trip panic
  and confirms through the same endpoint when panic engages. Customer data then routes to a
  chosen node. Actor: B3.
- **Support note.** Step 2 relies on the finagle spoofing and denial findings for the means
  to push nodes unhealthy. The seam contributes the recon loop and the live confirmation.
- **finatra effect.** Unaffected. No finatra edge touches `Balancer` or `PanicMode`;
  finatra's server chain sits above the client and only converts outcomes. FA3's per-route
  5xx counts add a second confirmation signal without changing the mechanism.
- **Composition.** CP2, finagle panic-mode findings, finagle spoofing and denial findings.

### C4. Blind removal

- **Entry points.**
  - EP5, required. A flag, a stack param, or a module override.
  - EP1, the reader. Anyone at the admin port sees the all-clear; no further foothold is
    needed to confirm the blind spot.
- **Step 1, finagle form.** A privileged actor removes a critical finagle observer by config:
  `sessionFailureDetector=none` installs `NullFailureDetector`, `FailFast(false)` drops the
  fail-fast factory, `FailureAccrual.Param.Disabled` drops the breaker.
  `F/liveness/FailureDetector.scala:31-33`, `:180-185`; `F/service/FailFastFactory.scala:82-83`;
  `F/liveness/FailureAccrualFactory.scala:226-227`.
- **Step 1, finatra form.** Override `statsReceiverModule` to `NullStatsReceiver`, or
  `configureHttpServer` with `withStatsReceiver(NullStatsReceiver)`, one def and one line
  (`inject-server/TwitterServer.scala:118`; `http-server/servers.scala:301`, `:404-406`).
  finagle's server stack then drops `StatsFilter` under a null receiver, and finatra's own
  filters write to the same null instance (FA10). One removal point across every finatra
  server at once.
- **Step 2, twitter-server.** The admin plane does not surface the removal as a fault. No
  lint rule covers a removed detector or breaker, and the params dump renders
  `FailureDetector.Param` as `GlobalFlagConfig`, not the resolved `NullFailureDetector`
  (`T/Linters.scala:20-35`). `NullStatsReceiversRule` would flag the null receiver but runs
  only on GET `/admin/lint` and exempts the admin server
  (`T/lint/NullStatsReceiversRule.scala:21-34`, `:29-33`). `ServerRegistryHandler.findScope`
  needs a `<name>/pending` metric, so the blinded server drops out of `/admin/servers`
  instead of showing as faulted (`T/handler/ServerRegistryHandler.scala:73-79`).
- **Combined impact.** An insider removes a critical observer in one or both lower layers,
  and the combined admin plane shows all clear: no lint failure, no status flag, a params
  dump that reads normal, and a server list that omits the blinded server. Death Signaling
  fails across the seam. Actor: B2.
- **Composition.** finagle tampering findings, FA10, twitter-server lint and registry
  findings.

### C5. Resolve-hang behind a live admin plane

- **Entry points.**
  - EP5, one cause. A bad dest in client config keeps resolution pending.
  - EP4, the other cause. An unhealthy or influenced ZooKeeper client moves resolution to
    `Pending`. Either cause suffices.
  - EP1, the read. The half-started process serves every admin route.
- **Step 1, finagle.** A registered client's `Dest` stays `Addr.Pending`
  (`F/finagle-serversets/.../serverset2/Zk2Resolver.scala:285-288`).
- **Step 2, finatra.** `postInjectorStartup` calls `Await.ready` with no timeout on the
  resolution future (`inject-server/TwitterServer.scala:200`), before warmup, the external
  bind, the health flip, and `Awaiter` (FA6).
- **Step 3, twitter-server.** The admin server, up since `premain`, keeps serving:
  `/admin/threads` shows the blocked main thread, `/admin/server_info` dumps environment and
  system properties, quit and abort work (`T/AdminHttpServer.scala:383-395`;
  `T/handler/ThreadsHandler.scala:43-56`; `T/handler/ServerInfoHandler.scala:33-46`).
- **Combined impact.** A half-started process stays open to B3 for as long as resolution
  stays pending. Impact is availability and secrets, not customer data, since no traffic has
  arrived. finatra creates this chain: neither lower layer waits on resolution. Actor: B2 or
  B3 into ZooKeeper for the cause; B3 with admin-port reach for the read.
- **Composition.** FA6, finagle serverset findings, twitter-server disclosure and halt
  findings.

### C6. A failed resolution passes the gate and health reports OK

- **Entry points.**
  - EP4, required. ZooKeeper write, or influence over resolution, so that a serverset
    resolves to nothing.
  - EP2, the consequence. Callers on the service port receive the 5xx that the failed
    downstream produces.
  - EP1, the reader. The orchestrator reads `/health`.
- **Step 1, finagle.** `Stabilizer` never emits `Addr.Failed`; it rewrites a failure to
  `Addr.Neg` when it holds no prior bound set
  (`F/finagle-serversets/.../serverset2/Stabilizer.scala:59-60`).
- **Step 2, finatra.** The gate waits only for a value other than `Pending`, so `Neg` passes
  and finatra logs "resolved to Neg" at info (`F/client/ClientRegistry.scala:38`, `:49-50`).
  finatra then binds the external servers and calls `warmupComplete()`
  (`inject-server/TwitterServer.scala:329`).
- **Step 3, twitter-server.** `warmupComplete()` swaps `/health` to OK
  (`T/Lifecycle.scala:103-106`). The orchestrator reads OK for a server whose downstream
  resolved to nothing. Requests on that client fail and finatra maps them to 5xx (FA8), still
  with no halt.
- **Combined impact.** A server that cannot reach its downstream is attested healthy by a
  gate that looks verified. finatra creates this chain: twitter-server alone flips health
  with no gate, and the gate finatra adds is what makes the OK look verified. Actor: B3 into
  ZooKeeper or B2 with ZooKeeper write.
- **Composition.** finagle Stabilizer findings, FA7, FA8, twitter-server `/health` finding.

### How finatra changes the two-layer chains

| Chain | Effect of adding finatra | Why |
|---|---|---|
| C1 Route then dump | Widens | Adds the server-side keep-serving link (FA8), the inert runtime halt (FA4), and the controller class names that make dump frames readable (FA2). No finatra edge gates any step. |
| C2 Recon then attack | Widens | Adds the server's own route table and filter posture (FA2), per-route health (FA3), the finatra library section of the registry, the finatra server label in the index, and the EP2 branch. |
| C3 Recon then panic | Unaffected | No finatra edge touches the panic mechanism. |
| C4 Blind removal | Widens | The shared `StatsReceiver` and classifier bindings (FA10) turn one param into a removal of both layers' server-side observation on every finatra server, and `ServerRegistryHandler` drops the blinded server instead of flagging it. |
| C5, C6 | New | finatra creates both; neither lower layer waits on client resolution. |

### Chains that stand only with inferred links

No chain joins a Community Notes gap to a stack gap in source. Three join them if a
deployment fact holds that no tree shows. Recorded, marked, and not counted in the impact
ranking.

#### C7. CRH push, lock, serve, disclose (inferred)

- **Entry points.**
  - EP8, required. Coordinated ratings.
  - EP1, inferred. The admin plane of whatever service serves the status.
- **Inferred links, two.** The combine job feeds a serving service; that service is
  finatra.
- **Source-supported half.** Inertia keeps a note at CRH when it was CRH last run and its
  intercept clears a lowered threshold (`scoring_rules.py:1559-1571`). After the lock age a
  status decided by a locking-enabled rule is copied into the locked status
  (`note_status_history.py:140-150`). The drift guard overwrites any new label with it
  (`scoring_rules.py:1608-1614`). The flip check excludes locked rows and ships off
  (`note_status_history.py:221-223`; `runner.py:44-50`). The status leaves at a
  comment-pinned index (`runner.py:437-447`; `constants.py:1036-1038`).
- **Under the inference.** The route table and controller classes reach the registry
  endpoint (FA2) and the same JVM answers a thread dump (C1). The inference adds a readout
  of the serving path, not a second corruption.

#### C8. Note writer against fault-to-response (inferred)

- **Entry points.**
  - EP11, required. A post author shapes the prompt.
  - EP13, required. The API path.
  - EP2, inferred. The service port of the API server.
- **Inferred links, two.** The CLI maps an HTTP error to a non-zero exit; the notes API
  path is served by finatra.
- **Source-supported half.** The POST goes through the CLI with test mode a literal, the
  pipeline sees only an exit code, and the return value is discarded (CN4).
- **Under the inference.** The fault-to-response conversions (FA8) answer the note writer's
  faults with status codes it never reads, and nothing halts on either side. Without the
  first link a 5xx is recorded as a successful submission.

#### C9. Deployment loop re-entering the locked baseline (inferred)

- **Entry points.**
  - EP8, required. Ratings that set the status.
  - EP12, inferred. The return path from the consumer to the export.
  - EP9, the read. The next run's status input.
- **Inferred link, one.** The return path.
- **Source-supported half.** Run N writes the locked status (`note_status_history.py:148-150`;
  `runner.py:446`). Run N+1 reads it as the status input (`runner.py:124-129`), passes it
  to the rules (`run_scoring.py:2423-2430`), the drift guard overwrites
  (`scoring_rules.py:1608-1614`), and the flip check ignores locked rows and ships off.
- **No stack gap participates.** The far side is a placeholder. A privileged actor who
  edits the persisted status is a position, not a stack gap.

### How Community Notes changes the stack chains

Source-supported change first. Inferred reach is listed separately and not counted.

| Chain | Change in source | Why | Inferred reach, not counted |
|---|---|---|---|
| C1 Route then dump | Unaffected | No seam edge lands in any Scala tree. No pipeline node runs in a JVM. | If the notes API is a finatra service, the note writer's post text, author id, and note text are in-flight data in a JVM that dumps threads, and the API path appears in the route table. Would widen. |
| C2 Recon then attack | Unaffected | The pipeline reads no admin route. | The note writer would be a caller on the EP2 branch whose only signal is an exit code, so a 5xx is recorded as a duplicate. Would widen the EP2 branch. |
| C3 Recon then panic | Unaffected | No seam edge touches the balancer. The pipeline makes no finagle client call. | None. |
| C4 Blind removal | Unaffected | No seam edge reads or sets a stack param, flag, or binding. | A blinded API server answers the note writer with the same exit code as a healthy one. No change to the mechanism. |
| C5 Resolve-hang | Unaffected | No seam edge touches client resolution or the admin plane. | A hung API server fails every CLI call and the pipeline records each as a duplicate. A consequence at the caller, not a step. |
| C6 Failed resolution passes the gate | Unaffected | Same. | The note writer would be among the EP2 callers receiving the 5xx. |
| C7, C8, C9 | New, inferred only | The seam creates them; each leaves a tree at a seam edge into a far side with no source. | As stated per chain. |

## Part D: Candidates tested and rejected

Each was tested against source and fails there. None was resurrected.

Between finagle and twitter-server:

1. **Per-host-metrics recon.** `HostMetricsExporter` returns a disabled stub unless the
   `perHostStats` flag is set, and even then per-host stats resolve to `NullStatsReceiver`.
   Conditional on an operator flag, not a default chain.
2. **A finagle fault triggers the twitter-server halt.** The only finagle-to-twitter-server
   edge logs and returns `false`; the halt fires only on an inbound POST.
3. **A twitter-server removal propagates a halt into finagle.** No twitter-server node can
   halt finagle; the seam edges are passive and finagle never reads twitter-server health.
4. **The MonitorFilter edge carries an attack signal into finagle.** It terminates at a log;
   no finagle observer reads that log.
5. **Kill the admin server so finagle notices.** finagle runs no liveness check on the admin
   server, and `disableAdminHttpServer` removes the plane by design.

With finatra:

6. **A finatra admin-handler fault halts the admin server through `MonitorFilter`.**
   `loggingMonitor.handle` logs and returns `false`; no path reaches `close()`
   (`T/AdminHttpServer.scala:359-364`).
7. **Attacker-driven warmup failure as a denial lever.** A throwing callback exits the app;
   the halt fires and propagates, which the model calls the required response, not a gap
   (`http-server/routing/HttpWarmup.scala:44-47`, `:79-80`).
8. **A finagle runtime fault trips `Awaiter`.** No server-stack module closes the
   `ListeningServer` (`F/server/StackServer.scala:126`, `:137`, `:174`); `Awaiter` reads only
   closure.
9. **finatra admin routes shadow `/health` or `/quitquitquit`.** Blocked at startup by the
   duplicate check and the `/admin/finatra/` prefix rule, and `combine` picks the longest
   pattern (`http-server/routing/AdminHttpRouter.scala:35-43`;
   `http-server/routing/HttpRouter.scala:704-717`; `T/AdminHttpServer.scala:157`).
10. **Double classification as a cross-check or attack surface.** Both layers use one
    classifier instance, so the counts cannot disagree and no comparator exists. Folded into
    C4 (`http-server/servers.scala:302`).
11. **`/admin/finatra/` routes as a covert admin surface.** The registry records every admin
    route with its `admin` key and the per-route metrics name them
    (`http-server/routing/Registrar.scala:73-80`). Disclosed, not hidden.
12. **Killing `DefaultTimer` to disable `Awaiter` and the shutdown timer.** No confirmed
    actor edge into the timer thread from B2 or B3 in the three trees. The shared substrate
    is recorded as FA4; the attack step is unsupported.
13. **The thrift `ExceptionManager`'s unused `StatsReceiver` as a cross-layer blind spot.**
    finagle's server `StatsFilter` on the thrift server still counts every outcome
    (`thrift/servers.scala:185`). The gap stays inside finatra; no seam widens it.


Between Community Notes and the stack:

14. **The note writer reaches the admin plane.** No Community Notes file names an admin
    route, port 9990, `/health`, or `/ready`.
15. **A Scala-side node reads a pipeline file, log, or exit code.** No file in any Scala
    tree names Community Notes, birdwatch, or the export.
16. **A far-side fault halts the pipeline, or a pipeline fault halts a service.** Each
    side's halt is local to its own process: a scorer exception re-raises inside the
    pipeline (`run_scoring.py:613`), twitter-server halts on an inbound POST on its own
    process, and finatra's `Awaiter` exits its own wait.
17. **finatra's thrift fault passthrough answers the enrollment importer.** The pipeline
    writes a file and makes no Thrift call.
18. **finagle-thrift's enum decode as evidence of the importer's unknown-code behavior.**
    The decode returns null on an unknown value on finagle's own tracing structs. An
    analog, not a link.
19. **The xAI endpoint as a Scala seam.** A third-party API, out of scope.
20. **The exit code as a heartbeat from the API.** One shot, caught, no response. Scored as
    a gap (CN4), not a threat of its own.

Rejected chains between Community Notes and the stack:

21. **A tampered export from a Scala-side insider.** No stack gap participates. The
    insider's position is file access on a host with no source; the admin plane offers
    reads and a halt, not a file write. The pipeline half stands and is recorded as CN3.
22. **An unknown enrollment code reaching the Thrift consumer.** The consumer is a
    placeholder. The pipeline half stands and is recorded as CN2.
23. **The generator's status gate as a cross-boundary chain.** The link is an abstract
    class with no implementation. No stack gap participates.

## Part E: Impact across the stack

The severity order follows one rule: who can reach customer data, and from how far outside.

**Highest: an external actor with a path to customer data.**

- Route then dump (C1) is the sharpest finding in the stack. From admin-port reach alone, the
  actor reads in-flight customer data out of a process that two layers refused to halt, with
  finatra's route table making the dump readable.
- The admin plane itself, unauthenticated on every route, hands a network-reachable actor a
  heap or thread dump of a request-serving process and every environment variable and system
  property.
- Five finagle findings put customer payloads onto an endpoint the system already knew was
  bad or that an external actor can influence: dispatch over a session marked Closed,
  fail-open accrual, panic-mode routing, a spoofed ping holding a dead endpoint in service,
  and ZooKeeper-influenced membership. Recon then attack (C2) and recon then panic (C3) remove
  the reconnaissance cost that would otherwise gate them.
- The seam hands an external actor a full reconnaissance surface with no foothold: the
  downstream mesh (CP1), live balancer health and panic posture (CP2), attached connections
  and their TLS posture (CP3), finagle's internal metrics (CP4), the application route table
  and filter chain (FA2), and per-route error rates (FA3).

**High: any access to customer data, secrets, or topology at all.** No actor should hold
customer data, so every route to it ranks high even from a privileged position. A privileged
insider disables any observer in any layer with a single flag, param, or binding and operates
behind it with no alert and no halt; the shared finatra binding (FA10) makes one override
blind two layers on every server. The model treats the insider as an adversary, so blind
removal (C4) and the namespace squat (FA9) count here rather than being discounted for being
internal.

**Also present: availability loss, process halt, and a blind observation plane.** One
unauthenticated POST halts the process and, through finatra, closes the customer-facing
servers (FA5). A half-started process stays open through its admin plane for as long as
resolution hangs (C5), and a server whose downstream resolved to nothing reports healthy
(C6). The observation plane runs on its own thread pool with admission control off, with
`NullStatsReceiver`, exempts itself from its own lint rule, and its only runtime halt reads
closure rather than health (FA4). The denial-of-service findings degrade availability by
serving through faults; they matter, but they sit below customer-data exposure.

**Community Notes, highest: an external actor with no foothold.** Note status is public,
so a corrupted status is a customer-facing outcome.

- **Through the public download.** One row per rater carries the viewpoint factor,
  intercept, enrollment state, earn-out count, and rating history, keyed by public ID. No
  observer bounds what is written, and the output validators require every column. The
  producer of the download is off-tree and unsigned (CN3).
- **Through ratings and notes.** A coordinated group pushes a note to CRH. Every observer
  on the path filters, demotes, or holds, and the run continues. After 14 days the status
  locks and the drift guard pins it, and the flip check that would measure the churn ships
  off. The status then leaves the pipeline at a comment-pinned index with no index assert
  (CN1), no marker of the run that produced it (CN5), and its time-of-production field
  left for the reader to write (CN6).
- **Through the note writer.** A post author shapes the prompt, a two-substring test is the
  only gate, and the note is submitted under the operator's credentials. The response, the
  one thing that says what happened to the note, is discarded unread (CN4).
- **Through the export's population.** Raters and authors whose data populates the export
  reach every scorer through a file the pipeline concatenates and field-counts and never
  dates, hashes, or checks for provenance (CN3).

**Community Notes, high: any privileged access on either side.** A supplied status history
sets public statuses directly through the drift guard, and nothing checks that the file
read is the file written (CN8). A supplied scored-notes file passes through for every note
outside the rescoring set. The enrollment file freezes participants. Planted joblib files
execute inside the scoring process. A code edit to any of the four comment-pinned
contracts, the column list (CN1), the code map (CN2), the tag order (CN10), and the
classification set (CN11), is pinned by no test and noticed by no reader in any tree. A
deployment that differs from the tree (CN7, CN9) means the running system is not the one
in source, and no observer can tell which one is running. Flags remove the flip check, the
similarity filter, and output validation, and forge the clock, with no halt and no alert.

**Community Notes, also present.** Every environment variable is printed to stdout at
prescoring start. Stale intermediates are read as current. A hung worker holds a phase
with no timeout. Shared-memory cleanup is skipped on the error path. An unknown enrollment
string is excluded from no transition and ships in a column declared Int64 with no cast
applied (CN2). Each keeps the pipeline serving through the fault.

**Inferred reach, listed and not counted.** If the notes API is a finatra service, the
note writer's post text and note text are in-flight customer data in a JVM whose admin
plane dumps threads, the fault-to-response conversions answer the note writer's faults
with status codes it never reads, and the API path sits in the disclosed route table (C7,
C8). If the combine job feeds such a service, the serving path of every public status is
disclosed on the same plane (C7). If the export and the enrollment importer run on the
stack, an insider there writes the pipeline's inputs and reads its unknown codes. No tree
supports any of these edges.

**The single pattern.** No layer has a ring, joining the layers makes no ring, and every
seam runs one way into passive stores. finagle keeps customer data resident by refusing to
halt, finatra refuses again one layer up, and twitter-server hands an external actor an
unauthenticated read of that same process. Community Notes filters, demotes, holds, or logs
every detection and proceeds to a public status, and its seam to the stack carries no
observer's health in either direction. Every finding in this document continues through
the fault; nothing halts on a detection, and no signal crosses back.

**Caveat.** In production the admin port is likely behind network controls, so an
unauthenticated route in the code is not the same as one reachable from the internet. Within
the code and the model, it is a real external-to-privileged crossing.

## Part F: Entry-point index

Every threat and chain, indexed by where an actor starts. Community Notes rows are named
by their component as in Part A4; seam threats by CN id.

**EP1, admin port.**
- finagle: JsonExporter disclosure.
- twitter-server: `/health` constant; registry handlers skipping unnamed components; the
  deadlock-hiding JSON path; lint, registry, and monitor repudiation; ServerInfoHandler and
  heap and thread dumps; shutdown and abort; every admin route unauthenticated; handler
  fault log-and-continue.
- Cross-boundary: CP1, CP2, CP3, CP4, CP5, CP6, FA1, FA2, FA3, FA5, FA6 (the read).
- Chains: C1 (required), C2 (required), C3 (required), C4 (the reader), C5 (the read), C6
  (the reader), C7 (inferred).

**EP2, service port.**
- finagle: caller-supplied Deadline replacing the timeout.
- finatra: per-request state; runtime heartbeat absence; HttpWarmup formality;
  continue-on-fault on the HTTP and Thrift chains; the 500-body side observation.
- Cross-boundary: FA8; CN4 (inferred).
- Chains: C1 (optional trigger), C2 (one branch), C6 (the consequence), C8 (inferred).

**EP3, mesh position.**
- finagle: ping spoofing; well-formed replies from a compromised endpoint; `bestOf` masking;
  FailFast probe flip; dispatch over a Closed session; fail-open accrual; panic-mode
  dispatch; null-pick dispatch; silent endpoints; `Tdrain` and `Tlease`; server never pings;
  channel faults.
- Cross-boundary: FA8.
- Chains: C1 (optional trigger), C2 (one branch), C3 (required).

**EP4, ZooKeeper.**
- finagle: serverset spoofing; Stabilizer swallowing `Failed`; stale state and probation
  masking.
- Cross-boundary: FA6 (one cause).
- Chains: C5 (one cause), C6 (required).

**EP5, code, config, flags, deploy.**
- finagle: every Tampering row; FailureAccrual removal without signal; Stabilizer
  repudiation; serverset stale state; channel faults; every Catastrophic observer disabled.
- twitter-server: `/health` constant; lint snapshot rules; unnamed registration; `/ready`
  and Warmup; lint constants; route table replacement; bind path; lint absence; registry
  emptiness; monitor; ContentionHandler; AdminThreadPoolFilter; `/health` and plane removal;
  MetricSource; handler fault; NullStatsReceiversRule exemption.
- finatra: filter order; mapper registry; per-request state; Thrift parallel; passive sinks;
  Thrift ExceptionManager; heartbeat absence; HttpWarmup; continue-on-fault; warmup
  attestation.
- Community Notes: constants; prescoring flip check hard-off; every observer removable by
  one edit; the DEBUG root logger.
- Cross-boundary: CP6, FA1 (the missing filter), FA6 (one cause), FA7, FA9, FA10; CN1,
  CN7, CN9, CN10, CN11.
- Chains: C4 (required), C5 (one cause).

**EP6, no actor entry.**
- finagle: session death unrecorded; detections ending in passive sinks.
- finatra: passive sinks; Thrift ExceptionManager dead receiver.
- Community Notes: phase runners reading stale intermediates; participant ID mapping
  faults; shared memory on the error path; phase completion as a log line; flip-check
  detail discarded; firm-reject log; the environment dump at prescoring start; the public
  output files (reached by anyone with the download); the prescoring rater intermediate;
  the flip-check failure message; the scorer runner's untimed wait; unfitted plausibility
  models.
- Cross-boundary: FA4.

**EP7, telemetry collector.**
- finagle: Zipkin error annotations egressing.

**EP8, input data.**
- Community Notes: rater-behavior detectors and status gates; the ratings directory;
  the enrollment TSV; rater detections unrecorded; the scorer runner under slow-converging
  data; the flip check under mass flips; scorers emptied by shaped ratings; small-input
  mode; the status lock; the live-note generator's status gate.
- Cross-boundary: CN2, CN3, CN5.
- Chains: C7 (required, inferred chain), C9 (required, inferred chain).

**EP9, previous outputs.**
- Community Notes: phase runners; the drift guard; participant ID mapping; rescoring
  passthrough; status history timestamps; the status history author column; intermediate
  pickles; intermediate loading.
- Cross-boundary: CN1, CN5, CN6, CN8.
- Chains: C9 (the read, inferred chain).

**EP10, pipeline CLI.**
- Community Notes: the similarity skip flag; every CLI flag; the status history writer's
  forged clock; small-input mode; every observer passed by flag.
- Cross-boundary: CN8.

**EP11, sibling.**
- Community Notes: the note writer (spoofing, repudiation, disclosure, denial, elevation);
  the live-note generator (disclosure, denial, elevation).
- Cross-boundary: CN4, CN9, CN11.
- Chains: C8 (required, inferred chain).

**EP12, export.**
- Cross-boundary: CN2, CN3, CN5, CN6, CN8.
- Chains: C9 (the return path, inferred).

**EP13, API.**
- Cross-boundary: CN4.
- Chains: C8 (required, inferred chain).
