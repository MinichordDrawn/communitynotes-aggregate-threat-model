# Self-Observing Systems: Framework, Gap Analysis, and Method

Three parts. Part 1 is the framework: what a system that watches itself has to look like.
Part 2 is the gap analysis: the attack that opens when each piece is missing. Part 3 is
the method: how the framework was applied to finagle, twitter-server, finatra, and
Community Notes.

The whole effort took one person about eight hours, starting with no prior knowledge of
any of the four codebases. Most of that time was spent waiting on the model passes; the
human work was steering, correcting, and reading the results.

A few words used throughout:

- An **observer** is any function that checks something about another component and
  reports or acts on what it finds. A health check, a circuit breaker, a validator, a
  lint rule.
- A **ring** is a group of observers that watch each other, so none can die unnoticed.
- A **coordinator** is the one extra observer in a ring that surface-checks all the
  others and holds the halt.
- A **halt** is the system stopping on a detected fault, rather than serving through it.

# Part 1. The Framework

A framework for distributed systems that observe themselves. Four sections: the structure
a correct system converges on, the four constraints it rests on, the eight properties
those constraints require, and the rules that scale it to different stakes and trust
levels.

## 1. Emergent Structure

Satisfy the constraints and properties together and one structure falls out. A correct
self-observing system converges on it, so it works as a reference architecture. Compare a
real system to the structure, find where it deviates, and name the missing property.

### The Ring

Observers form a ring of at least three plus one. The three watch each other.

- A ring of two dies when both nodes are killed, one after the other or at once.
- A ring of three is much harder to kill at once.
- Rings inside rings, where a node is itself a ring, are stronger still.

The plus one is the ring's coordinator. It's a different kind of observer from the other
three. It belongs to the ring, not outside it. It checks something about every other
observer, including itself, and the others report and validate back to it. Its check
stays shallow, because its job is coordination, not deep inspection. That shallow vantage
is what lets it act as the ring's primary halt.

Every observer checks one other observer and reports to a different one.

Rings nest upward through their coordinators. A ring's coordinator is a member of the ring
above it and carries its ring's health up. A coordinator two levels up doesn't check the
rings two levels down. Those rings verify themselves and report to the ring above. A
failure climbs the chain of coordinators, so the top sees it without inspecting the
bottom. Deep verification stays local. Only the surface signal travels.

### Triangulation

Three points define a plane. Three edges define a rigid polygon. Three independent
constraints define one solution. Three is the minimum for determinacy. With fewer, the
system has free play: gaps, ambiguity, instability. The plus one uses that determinacy to
act.

### Structure and Information

The framework prefers structure to process: things that are fixed, static, and checkable
over things that are decided. Hash verification over human approval. Cryptographic proof
over trust. A machine overriding a human over a committee. A process-heavy design carries
more human gates, more approvals, more coordination, and it wouldn't satisfy the
constraints. It'd be a workflow. Structure is the core requirement for safety-critical
design.

Satisfying all the constraints pushes a system onto structural checks rather than
procedural ones. A structural check is faster by nature, even when the structure runs
deep. A boolean test resolves in one step. A loop or an evaluation walks its inputs. Depth
doesn't close the gap: chained structural checks still beat chained procedural checks by
a wide margin, because each link resolves in a step rather than a scan. That speed is why
a structural response can keep up with an exponential threat and a procedural one can't.

Because structure is faster, the model can judge availability and performance too, not
only safety. A tradeoff made in the name of availability isn't automatically the better
design. The structural alternative may win on availability as well, and only modeling it
settles the question. Treat an availability claim as a claim, not a justification.

Everything is either structure or information. Structure is spatial, persistent, rigid:
what things are. Information is temporal, dynamic, causal: what things do. Data versus
process.

Three is the smallest count at which structure becomes stable, holding without unresolved
failure modes. Below three there are gaps. Hamming distance two can detect an error but
can't correct it.

The plus one is always:

- Singular. Exactly one, never two.
- A different kind from the three. Information, where they are structure.
- What makes stable structure do something rather than merely exist.

Three is minimum rigidity: a triangle can't deform without changing an edge length, and a
quadrilateral can. The plus one turns rigid structure into an active system.

### Halt on Fault

Byzantine fault tolerance is the wrong model. A system that keeps serving through a
detected fault keeps serving through a compromise. When any observer detects a fault, the
system halts. No actor, however privileged, passes a critical observer without the
observer alerting and the system halting until that actor has left.

Any node can halt the system. Any halt propagates to at least two other nodes. That's the
reflexive response requirement.

A cascading failure grows exponentially, and a sequential observer can't contain it.
Scheduled interventions, periodic reviews, and intelligent evaluation all add latency, and
during that latency the process doubles. Only a reflexive, structural response matches the
curve, because reflexive responses multiply too: one agent's detection triggers every
other agent at once. Procedural evaluation is too slow. The constraints force reflexive
response. Sequential observation of an exponential process guarantees an eventual
undetected failure.

### Execution-Observation Coupling

The observer emits a heartbeat. The execution thread checks that heartbeat and runs only
while it beats. When the heartbeat stops, execution stops.

That couples execution to observation as structure, not policy. Execution can't outlive
its observer, so an attacker who kills the observer also kills the work it was watching.
The check is a dead-man's switch: the default is stopped, and only a live heartbeat
permits work. The usual order is execution runs and observation watches from the side.
Here, observation gates execution.

## 2. The Four Constraints

A system of distributed agents where:

1. **No central controller.** No agent coordinates the others. No agent has authority over
   any other.
2. **Catastrophic cost of undetected failure.** If any agent's observation fails silently,
   the consequences far outweigh the cost of a false alarm.
3. **Adversarial environment.** Threats can mimic legitimate components. The system can't
   assume failures are benign.
4. **No external maintenance.** The system detects, responds to, and recovers from failures
   on its own, with no outside authority to notice or step in.

## 3. The Eight Required Properties

The four constraints require these eight properties. Remove any one and a specific attack
opens (Part 2 gives the attack for each). Every observer satisfies every property. The
ring is what lets a lone observer satisfy the response properties through its peers.

1. **Heartbeat.** A liveness signal whose absence triggers a response.
2. **Substrate Dependency.** The observer shares the failure substrate of what it watches.
3. **Continuous State Verification.** The observer checks current state against a known
   good state, not only events.
4. **System-Wide Response.** One observer's detection reaches the others.
5. **Self-Defense.** The observer resists removal by a single action.
6. **Death Signaling.** When an observer dies, its absence crosses an independent boundary
   and gets acted on.
7. **Automatic Integration.** New components enter observation on their own, with no
   manual-configuration window.
8. **Reflexive Response.** The observer responds on detection, with no evaluation window
   an attacker can act in. Any node can halt the system, and a halt propagates to at
   least two other nodes.

## 4. Consequence-Proportional Complexity and Trust Boundaries

### Consequence-Proportional Complexity

How much observation a subsystem needs scales with what it costs if that observation
fails silently.

| Consequence tier | Required observation |
|---|---|
| Bounded (one service) | A scheduled check and an alarm |
| Significant (multi-team) | A heartbeat and automatic restart |
| Severe (organizational) | Redundant observers with escalation |
| Catastrophic (system-wide) | The full stack: the 3+1 ring |

**Customer data sets the floor.** Any subsystem that carries customer information runs at
the Catastrophic tier, whatever else is true of it, and needs the full stack. The public
comes first. That's a hard requirement, not a tradeoff.

Catastrophic is relative to the observer. A small organization may judge silent compromise
of its critical systems catastrophic to its operations. The constraints apply at any scale
where that judgment holds. The structure is the same at every scale. What varies is the
size: how many rings, how deep the detection chains, how many independent trust
boundaries.

### Trust Boundaries

Three boundaries, nested from the inside out.

1. **The observer ring, the system itself.** The machine-to-machine observer network. No
   human belongs inside it. Any human access is an adversary event the ring detects and
   halts on.
2. **Privileged actors.** People with backend access, code submission, deploy, and config
   rights. They can cause real production impact by mistake or on purpose.
3. **External actors.** Customers and outside adversaries, with no foothold.

The ring treats everyone outside itself as an adversary, privileged insiders included. No
human touches the ring undetected. Halt on fault holds for boundary 2: a privileged actor
passes no critical observer without an alert and a halt.

# Part 2. The Gap Analysis

The eight properties aren't optional features. They're forced by the four constraints, and
here's the practical test: remove any one and a specific, exploitable gap opens where a
compromise can go undetected.

## 1. Heartbeat (missing)

**Gap.** No liveness signal, so nobody knows if the observer is alive. An attacker kills
it silently, because there's no signal whose absence triggers a response. The observer
stops, and the system runs unprotected until a human happens to notice.

**Attack.** Kill the observer. Wait. Operate freely.

## 2. Substrate Dependency (missing)

**Gap.** Picture a security camera on a separate power grid from the building it watches.
A burglar cuts the building's power: the locks go dead, the alarm goes offline, the vault
opens. The camera keeps rolling, pointed at a dark building, recording nothing useful.
It's working perfectly. It just doesn't share the grid, so it can't feel the outage that
enabled the break-in.

**Attack.** Attack the infrastructure the observer doesn't share. The observer stays
healthy and reports all clear, because the problem is somewhere it can't feel.

Substrate dependency means the camera sits on the same grid. When the power goes out, the
camera goes dark too, and that's the signal. "Camera offline" triggers the response.
Shared failure is detection.

## 3. Continuous State Verification (missing)

**Gap.** Without a known good state to check against, the system has no definition of
normal. It sees events (things that happen) but not states (things that exist and
shouldn't). An attacker who plants something persistent, a backdoor that just sits there,
is never detected, because nothing checks whether the current state matches the expected
one.

**Attack.** Make one change. Wait. The observer detects activity but not presence. The
backdoor persists.

## 4. System-Wide Response (missing)

**Gap.** One agent's detection doesn't reach the others. Each agent works alone. An
attacker compromises them one at a time. Each sees its own compromise and can't tell
anyone.

**Attack.** Compromise agents in sequence. By the time a human correlates the failures,
the whole system is gone.

## 5. Self-Defense (missing)

**Gap.** One action removes the observer. An attacker's first move is to kill it, then
operate freely. Without several independent mechanisms preventing removal, the observer
is a single point of failure for the whole security model.

**Attack.** Delete the observer with one API call or one permission. Then do anything.
The thing that would have caught you is gone.

## 6. Death Signaling (missing)

**Gap.** When the observer dies, its signal dies with it. Nothing crosses into an
independent trust boundary. Killing the observer removes the detection and the alerting
in one move. Nobody outside the compromised boundary knows anything happened.

**Attack.** Kill the observer inside its boundary. No signal escapes, so outside systems
stay unaware. The compromise is contained inside the boundary, and so is every trace of
it.

## 7. Automatic Integration (missing)

**Gap.** A new component is unprotected until someone configures the observer to watch
it. Between "deployed" and "observer updated," the component is invisible.

**Attack.** Deploy during the gap. Or add a component the observer doesn't know about. It
runs unwatched until a human updates the configuration. In a fast-scaling environment
that gap can last days, or forever.

## 8. Reflexive Response (missing)

**Gap.** The observer detects a violation and then evaluates before responding. Is it
really an attack? Should it escalate? Check the context, wait for more data. During that
window the exponential threat doubles. The attacker works in the time between detection
and response.

**Attack.** Trigger detection, then act while the observer decides. By the time it
decides, the compromise has spread past containment.

## The Audit

Run your system against this table. Every row you can't check off is an open attack vector.

| Property | If missing, an attacker can |
|---|---|
| Heartbeat | Kill the observer silently and operate freely |
| Substrate Dependency | Attack through infrastructure the observer doesn't share |
| Continuous State Verification | Plant a persistent backdoor that's never detected |
| System-Wide Response | Compromise agents one at a time with nothing propagating |
| Self-Defense | Delete the observer with one action, then do anything |
| Death Signaling | Kill the observer inside its boundary with no signal escaping |
| Automatic Integration | Deploy in the gap between "new component" and "observer updated" |
| Reflexive Response | Act during the window while the observer decides |

# Part 3. The Method

How the framework was applied to four real codebases. The framework is Part 1. The gap
analysis is Part 2. Everything below is the procedure, the passes, the targets, the
numbers, and the rules for combining the results.

This is what worked for analyzing these codebases. A new target may call for a tweak,
and that's expected.

## 1. The procedure

The procedure runs ring-first. The 3+1 ring is what a correct self-observing system
converges on, so any deviation from it points at a missing property. Find where the ring
breaks, then name the gap. Check structure before information: the three mutually
watching nodes are structure, the plus one is information, and information stays rigid
only on structure that already is. So the ring comes first.

The whole procedure is written down here so an agent with no memory of the discussion can
run it from scratch. It has four steps, a recursion rule, a judging rule, and a recording
rule.

### Step 0. Tier by consequence

Before scoring anything, give each observer subsystem a consequence tier. The required
observation scales with what it costs if that observation fails silently. The tiers are
in Part 1, section 4.

Check the customer-data floor first. If customer data flows through the observer or the
thing it watches, the tier is Catastrophic and the ring is mandatory, whatever else is
true.

The tier sets the bar the later steps score against. A missing ring or property counts as
a gap only when the tier requires that much. A bounded observer running on a scheduled
check and an alarm meets its bar, and the absent ring there isn't a gap. Record the tier
and one line of justification for each subsystem.

### Step 1. Look for ring topology

Map every observer in the system. For each one, record two edges: which observer it
checks, and which observer it reports to.

Test the map against the four ring requirements, and mark each present or missing:

- A cycle of at least three observers that watch each other.
- Each observer checks one other observer and reports to a different one.
- One plus-one observer, a different kind, that checks every observer including itself.
- Nested rings, where a node is itself a ring.

A pairwise edge (two nodes, one direction) or a star (many nodes, one hub) counts as a
missing ring.

Then recurse, from the bottom up:

1. **Find the base rings.** List every observer and group them into rings. Any observer
   in no ring is a gap.
2. **Name each ring's coordinator.** The plus one: the node that checks something about
   every other observer in the ring and takes their reports and validations back. It
   checks at the surface, not deep. A ring with no coordinator is a gap.
3. **Find the super-rings.** Treat each coordinator as a node in the layer above and
   group those into rings the same way.
4. **Repeat.** Name each super-ring's coordinator and look for the layer above that. One
   layer at a time.
5. **Stop and record.** Stop when the top layer is a single ring, or when no ring forms at
   the next layer up. The second case is a gap: the structure doesn't close. Write the
   result layer by layer, marking every layer where a ring or a coordinator is missing.

### Step 1.5. Name the missing structural components

Where Step 1 finds no ring, say which components the model requires and which are
absent. Stay descriptive. Report the missing structure; don't design the fix, and don't
say which specific observers should watch which. The system's architects know their
architecture. The audit names the gap, not a build.

For each missing ring, mark each of these present or absent:

- **A mutual-observation cycle.** At least three observers that watch each other and
  close a loop. Absent when every edge runs one way: a chain, a star, a DAG.
- **A coordinator.** A plus one that surface-checks the members and can halt. Absent when
  the only breadth observer is a collection hub or a star center that takes no validation
  back.
- **Check and report separation.** Each observer checks one node and reports to a
  different one. Absent when an observer reports only to itself or to a passive sink.
- **A halt path with propagation.** Any member can halt, and a halt reaches at least two
  others. Absent when a halt stays local.
- **An upward chain.** A coordinator that carries this ring's health into the ring above.
  Absent when the layer doesn't close, or nothing reports upward.

A system with no rings gets a named list of what's missing, not a blank verdict, and the
audit proposes no architecture.

### Step 2. Run the gap analysis on the missing rings

For every missing or broken ring, run the eight-property audit from Part 2 on the
observers involved. A missing ring predicts specific gaps:

- No cycle of three points at System-Wide Response, Self-Defense, and Death Signaling.
- A check target and a report target collapsed into one node points at Self-Defense.
- No plus one means reliance on an outside authority, which breaks the fourth constraint.

Every observer satisfies every property. The ring is what makes that possible. A pure
observer satisfies System-Wide Response, Self-Defense, and Reflexive Response through its
place in the ring, not by carrying its own actuator. It reports to a peer, its peers watch
it for removal, and it reports the moment it detects. No observer is exempt because it
only observes. An observer that can't satisfy those three is showing the missing ring, so
score that as a gap, not an exemption.

Score each property as a gap (code-confirmed) or satisfied. One dial lowers the bar: the
tier from Step 0. Below the tier that requires a property, mark it not applicable and name
the tier. Nothing else exempts a property.

For each gap, record the required response under halt on fault: the observer alerts and
the system halts until the fault clears or the privileged actor leaves. Continuing through
a fault is itself a gap. Graceful degradation, panic-mode routing, and Byzantine
availability all keep serving through a detected fault, which keeps serving through a
compromise. Flag every one.

### Judge only within the model

Report the structure against this model, not against outside standards. A structure that
doesn't close into rings is unsafe under these constraints, whatever its conventional
merits.

Don't credit a design as superior because it serves availability or performance. The
model can judge those too, and a structural design may win on them, so an availability
tradeoff is a claim, not a justification. Settling that claim means modeling the
alternative structure, which is a separate and slower analysis. The audit evaluates
threats only, so leave the availability claim unevaluated rather than conceded, say so,
and give the threat verdict plainly. State the system's own goal where it clarifies the
deviation.

### Recording results

Record findings as a STRIDE threat model. Map each gap to its category: Spoofing,
Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege.
For each finding, record:

- The component.
- The actor, by trust boundary: the ring (B1), a privileged insider (B2), or an external
  actor (B3), naming any crossing the threat needs. The ring treats B2 and B3 as
  adversaries, and halt on fault requires an alert and a halt even for B2.
- The entry point: where the actor starts.
- The threat, and the file and line that prove it.
- The property, ring gap, or constraint it maps to.
- The halt-on-fault control: whether the observer alerts and the system halts, or whether
  it continues through the fault.

Three templates fix the shape of the output: a findings template, a STRIDE threat-model
template, and a writing guide.

## 2. The three-pass method

Every target ran the same three passes. Each pass was a separate agent with no memory of
the discussion, reading only the framework, the previous pass's output, and the source.
Fable and Opus are two Claude models: Fable for the breadth passes, Opus for the check.

| Pass | Model | Job |
|---|---|---|
| 1 | Fable | The ring map: every observer, its edges, the ring test, coordinators, what's missing |
| 2 | Fable | The gap analysis and STRIDE tables, built on the map, every citation reopened |
| 3 | Opus | Verification: open every load-bearing citation at its line and confirm, correct, or drop it |

The first two passes were citation-strict. Every claim needs a file and line the agent
opened, and anything it couldn't confirm it had to say so. The third pass checked that. A
claim survived only if the cited lines said what the claim said. If the lines said
something different, the claim was corrected when the true content still supported a
finding and dropped when it didn't. The verifier added no findings of its own; the one
addition, in the Community Notes run, was forced by a correction and marked as such.

After the three passes, the clean deliverable was written from the verified content into
the template shape, then style-checked by script.

Verification results across every run:

| Target | Confirmed | Corrected | Dropped |
|---|---|---|---|
| finagle | 86 | 0 | 1 |
| twitter-server | all | 1 clarification | 0 |
| finagle and twitter-server seam | all | 1 | 0 |
| finatra, standalone and three-layer | all | 0 | 0 |
| Community Notes | 71 | 11 | 2 |
| Community Notes seam | 90 | 9 | 2 |

Every change was small: a line range off by one, a citation pointing at the wrong route,
an impact overstated, one claim split into the separate behaviors it covered. The finagle
drop was a property rescored from gap to satisfied because the same code was already
scored as a gap under two other properties. No structural claim moved in any run, and no
drop removed a finding that the rest of the model depended on.

## 3. The targets, in order

1. **finagle.** The first run. It set the shape: a graph of observers with no cycle,
   twelve Catastrophic groups, every observer serving through a detected fault.
2. **twitter-server.** A request-driven star. No authentication on any admin route, heap
   and thread dumps, an environment dump, a one-POST halt.
3. **finagle and twitter-server, across the seam.** 22 seam edges, no cross-package ring,
   six seam threats, four chains, five rejected candidates. Route-then-dump appeared here.
4. **finatra, standalone.** A linear filter chain, 25 property gaps.
5. **finatra on both lower layers.** 23 seam edges, ten seam threats, five chains
   including extensions of the earlier ones, eight rejected candidates.
6. **The three-layer aggregate.** Every threat from finagle, twitter-server, finatra, and
   their two seams in one standalone file, with an entry-point key. Every chain lists each
   entry point as required, optional, or a branch.
7. **Community Notes, standalone.** A Python batch pipeline. Mapped as a graph of 50
   observers across four phases, with a cross-run loop that carries no health signal.
8. **Community Notes and the stack, across the seam.** Nine crossings, none with a code
   edge in any tree. Eleven seam threats. No chain stands in source; three stand only on
   inferred deployment links.
9. **The four-system aggregate.** The three-layer aggregate plus Community Notes and its
   seam, in one file. 150 recorded threats, 23 rejected candidates.

## 4. Aggregation rules

- Record every threat once, in a per-system table or a cross-boundary table.
- Give every threat an entry point from a fixed key. A chain lists every entry point and
  marks each one required, optional, one branch, one cause, or the reader.
- A chain stands in source only if every link is in a tree. A chain with an inferred
  deployment link is recorded, marked, and left out of the impact ranking.
- Keep the rejected candidates, with the reason each fails in source.
- Order impact by one rule: who can reach customer data or corrupt a public outcome, and
  from how far outside.
- The aggregate references no workspace file. The model is restated inside it.
