# Distributed Systems Playground

**Project Plan — Version 0.3**

**Status:** Planning baseline; implementation details remain subject to validation

**Primary purpose:** Help the author obtain a Senior, Staff, or Principal engineering role by demonstrating software and system-design competence

**Broader purpose:** Practical experimentation and accessible communication of distributed-systems principles

**Primary reference:** *Designing Data-Intensive Applications*, supplemented by broader distributed-systems and system-design concepts

## 1. Vision and audience

Build a publicly accessible laboratory where visitors formulate hypotheses, manipulate real systems through bounded controls, inspect evidence, and evaluate architectural consequences.

The primary audience is experienced software engineers, software architects, and technical hiring managers evaluating Senior, Staff, or Principal-level competence. Secondary audiences include nontechnical hiring managers, recruiters, and engineers exploring distributed systems. Employment is the author's primary goal; the project's usefulness is not restricted to hiring.

Design for three levels of engagement:

| Time | Audience | Intended outcome |
|---|---|---|
| **30–60 seconds** | Hiring manager or recruiter | Understand the purpose and find credible evidence of working software |
| **3–5 minutes** | Technical hiring manager or experienced engineer | Complete a guided experiment and understand its architectural implications |
| **15–30+ minutes** | Architect or technical interviewer | Inspect implementation, assumptions, failure behavior, evidence, and design decisions |

Optimize for a technical evaluator, with a fast entry point and progressively deeper evidence. Keep explanations accessible without making beginner instruction the central experience. Demonstrate the ability to translate concepts into working systems, predict failure, measure outcomes, and defend engineering decisions.

## 2. Core principles

### Architectural consequences over infrastructure sophistication

The demonstrations are the product. The platform exists to support them.

Every experiment must explain why observed behavior matters to application invariants, user expectations, recovery, or product requirements. A replication experiment, for example, should explore how lag violates read-your-writes expectations and how mitigations affect latency, availability, and operational complexity.

Judge success by engineering insight and evidence, rather than the number of technologies deployed or visual sophistication. Complete one vertical slice before expanding the catalog. Introduce abstractions when repeated requirements justify them.

### Trustworthy evidence before presentation

Establish experimental correctness through automated scenarios before building the guided interface. The interface must communicate verified behavior, including uncertainty and missing observations.

**A demonstration's conclusions must never be stronger than the evidence its implementation can produce.**

### Transparency, provenance, and verifiability

Each experimental claim must be traceable to observable behavior from identifiable components. Identify the execution environment, participating services, implementation revision, measurement sources, and limitations.

Make definitions, implementation, deployment configuration, and sanitized execution records inspectable and independently reproducible. Clearly distinguish live execution, recorded execution, simulation, inference, and incomplete telemetry. Never silently substitute one presentation mode for another.

Telemetry from software controlled by the project remains self-reported evidence. Credibility comes from inspectable implementation, corroborating observations, and independent reproduction. Cryptographic attestation is outside the initial scope.

## 3. Scope and constraints

### In scope

- Real databases, brokers, workers, and application services.
- Predefined scenarios with bounded inputs and controlled fault injection.
- Source-attributed events, durable business records, basic metrics, and execution exports.
- Comparisons of architectural choices and their business consequences.
- Reproducible local and cloud environments, including teardown and reconstruction.
- Documented assumptions, decisions, limitations, and experimental findings.
- Anonymous access with separate run state and results.

### Outside the initial scope

- Production-grade availability, disaster recovery, or high-concurrency experimentation.
- Registration, arbitrary visitor commands, or visitor-controlled cloud provisioning.
- Per-visitor Droplets, database clusters, or unrestricted infrastructure administration.
- Kubernetes, generic workflow engines, sophisticated schedulers, or experiment plugin frameworks.
- Real financial transactions, external payment integrations, or order-fulfillment workflows.
- Custom replacements for established infrastructure or observability tools.

## 4. Initial technology and deployment

| Responsibility | Initial selection |
|---|---|
| Source and automation | GitHub / GitHub Actions |
| Frontend hosting | Cloudflare Workers Static Assets |
| Experimental compute | DigitalOcean Basic Droplet running Ubuntu |
| Containers and local environment | Docker / Docker Compose |
| Frontend | React + TypeScript + Vite |
| Experiment controller | Python FastAPI, provisional |
| Reverse proxy | Caddy |
| Durable business state | PostgreSQL |
| Messaging | RabbitMQ |
| Processing | Python worker |
| Initial fault injection | Worker checkpoints and controlled termination; Toxiproxy later if justified |
| Instrumentation | Structured events and basic metrics; OpenTelemetry when needed |

All experimental services initially share one DigitalOcean Droplet. Begin with 4 GB if measured requirements fit; increase capacity only with evidence. Cloudflare hosts the frontend independently, including documentation and recorded results when compute is archived.

Represent topology accurately: containers on one host are not independently hosted machines. Identify shared CPU, memory, storage, and failure boundaries when interpreting results. Add hosts or services only when an experiment requires them.

## 5. Architectural organization

The system has three logical layers:

- **Presentation:** Business scenarios, guided controls, comparisons, explanations, provenance, and evidence inspection. Communicates through HTTP APIs and, where useful, server-sent events.
- **Experiment control:** Parameter validation, run ownership, initialization, checkpoints, failure injection, evidence collection, time limits, cleanup, and recovery.
- **Experimental infrastructure:** The actual broker, database, workers, and fault-injection components under examination.

Start with straightforward mechanisms serving the account-credit experiment. Avoid a generalized orchestration platform until subsequent experiments reveal common requirements.

## 6. First experiment: Message delivery and account credits

**Architectural question:** How can an accepted business operation eventually take effect without producing duplicate committed credits when messages may be delivered more than once?

### Business consequence first

Open the guided experience with:

> Your customer received 50 credits when they should have received 25. The message broker behaved exactly as configured. What went wrong?

Use synthetic credits, not real money. A baseline account starts at **100**, and one operation requests **25** additional credits:

| Outcome | Balance | Committed business effect |
|---|---|---|
| Expected | 125 | One credit of 25 |
| Duplicate processing without protection | 150 | Two credits of 25 for the same logical operation |

Choose this model because its invariant is simple and measurable. Defer inventory, fulfillment, compensation, and external-service workflows.

### Identities and guarantee boundaries

| Concept | Meaning |
|---|---|
| Run | One initialized execution with a unique `run_id` |
| Operation | Business intent identified by `operation_id`, independent of transport identifiers |
| Message | A publication carrying that intent and an application-assigned message identifier |
| Delivery | One broker-to-consumer delivery; redelivery of the same message is another delivery |
| Processing attempt | One worker invocation, with a distinct attempt identifier |
| Committed effect | A durable credit ledger entry and corresponding balance adjustment committed in PostgreSQL |
| Prevented duplicate | An observed repeat attempt resolved against existing operation state without another committed credit |

Use **`(run_id, operation_id)`** as the idempotency key. Preserve it across retries and redeliveries. Redelivery need not change the message identifier; delivery and processing-attempt identities are separate. A new publication of the same operation must also preserve its business identity. Broker delivery tags are scoped to channels, not globally unique execution identifiers.

Bind operation identity to its account and credit amount. Reuse of the same key with different business inputs must be rejected as a conflict, rather than silently treated as a successful duplicate.

### Safety, liveness, and transaction boundaries

- **Safety:** In protected mode, each unique operation produces at most one committed credit.
- **Liveness:** An accepted operation eventually produces its intended credit if required services recover, durable state survives, the operation is valid, and retries continue with eventual processing opportunities.
- **Accounting consistency:** The account balance equals its baseline plus committed ledger credits.

Record the idempotency decision, durable credit ledger entry, and balance adjustment in the same PostgreSQL transaction. Enforce uniqueness on the idempotency key and handle retries without a separate check-then-write race. The unprotected comparison deliberately permits multiple ledger effects for one operation while preserving their logical identity for inspection.

Define acceptance precisely in the experiment specification: what durable record or confirmed broker publication creates the obligation to complete, how interrupted publication is recovered, and how uncertain client responses are retried without changing operation identity. Do not claim acceptance from an in-memory request alone. The specification must close this gap before liveness assertions are implemented.

Together, safety and liveness support **effectively exactly-once business effects within the stated transaction and recovery assumptions**. They do not promise exactly-once execution, one delivery, or atomicity with an external notification, payment provider, or second database.

The public run timeout is an operational bound, not proof of eventual completion. Liveness tests demonstrate recovery within specified bounds under tested assumptions; they cannot prove progress under arbitrary failure or endless disruption. Reset or expiration ends a run's experimental contract and must be shown explicitly, rather than reported as successful processing.

### Initial automated scenarios

1. Normal delivery, one committed credit, and acknowledgment.
2. Worker failure before commit: no partial credit, followed by successful recovery and retry.
3. Worker failure after successful commit and before acknowledgment, without idempotency: redelivery produces a duplicate committed credit.
4. The same failure with transactional idempotency: a repeat attempt occurs, but only one credit remains committed.
5. Repeated delivery or publication of the same operation, and multiple distinct operations: protection deduplicates by business identity without suppressing valid new credits.
6. Timeout, controller interruption, and cleanup recovery: no abandoned worker or stale message contaminates the next run.

Tests must cover balance and ledger assertions, stable operation identity, controlled checkpoint behavior, and reset correctness. Verify normal and pre-commit recovery with protection both enabled and disabled where applicable. The deliberately unprotected scenario succeeds as an experiment when it demonstrates the predicted safety violation.

### Reproducible failures and uncertain outcomes

Use explicit synchronization checkpoints around transaction boundaries and acknowledgment. The controller waits for the selected checkpoint, then terminates the worker and records the action and observed termination. Do not rely on guessed sleep durations.

Checkpoint placement makes the failure window reproducible; it does not make broker redelivery timing deterministic. Validate eventual database state and delivery observations within bounded timeouts.

A worker that knows its commit succeeded and crashes before acknowledgment is different from a worker that loses the database response and cannot determine whether commit occurred. Explain this distinction from the start. A later extension should inject commit-response uncertainty and validate reconciliation by operation identity against authoritative database state. Do not claim the initial checkpoint scenario demonstrates that separate failure mode.

An external side effect is another extension, requiring its own atomicity analysis. Explain that limitation in the initial demonstration.

## 7. Anonymous execution and run lifecycle

Share infrastructure while keeping browser-session controls and run results separate. Initially serialize all live executions of this experiment, including disruptive actions on shared workers. One visitor owns the active run; others can browse explanations and clearly labeled recorded executions.

Expose only bounded controls: scenario selection, idempotency mode, predefined failure checkpoint, message count, execution, and reset. The controller validates combinations and enforces limits. No arbitrary commands, SQL, management endpoints, or cloud provisioning are exposed.

Implement:

- Unique run and operation identifiers, with run-scoped state and evidence.
- A controller-owned execution lock with expiration and restart recovery.
- Session ownership checks for mutation; knowledge of a run identifier alone is not authorization.
- Deterministic initialization, a strict execution timeout, automatic cleanup, and abandoned-session expiration.
- Lifecycle states such as `created`, `running`, `completed` or `failed`, `cleaning_up`, and `terminated`.
- A separate preserved outcome and evidence-completeness status, so cleanup does not erase whether the scenario passed, failed, timed out, or remained indeterminate.

Expiration triggers recovery; it does not immediately authorize another run. Quiesce or fence the previous execution, remove pending work, restore the baseline, and verify isolation before granting new ownership. Recover the lock and interrupted cleanup after controller restart. If cleanup cannot be verified, keep execution unavailable and report the condition.

Reset must handle workers, queued or unacknowledged messages, account state, and run-scoped artifacts. Preserve retained evidence before removing mutable experimental state. Every new run must establish and verify its own baseline.

Choose exact message limits, run timeouts, expiration intervals, retention periods, and public request limits in the specification and deployment configuration. Tune them using measurements. Consider several isolated environments on the same host only if usage justifies the additional complexity.

## 8. Evidence contract and provenance

### Authoritative evidence

| Claim | Required evidence and limits |
|---|---|
| A credit committed | Durable PostgreSQL ledger entry, corroborated by account state |
| A credit committed twice | Two committed ledger effects for the same `(run_id, operation_id)` in the unprotected scenario |
| A duplicate was prevented | Evidence of a repeat attempt and its deduplication result, existing idempotency state, and no additional committed credit after the run is quiescent; one ledger entry alone does not prove prevention |
| A worker attempted processing | Source-attributed worker instrumentation, which may be incomplete after a crash |
| A message was redelivered | Broker delivery metadata observed by the consumer and correlated with message and operation identity; do not infer an exact delivery count from a redelivery flag alone |
| A worker crashed after commit | Controller checkpoint and runtime termination observations, corroborated by durable database state; a termination request alone is insufficient |

Worker statements about commits are observations, not substitutes for inspecting PostgreSQL. Broker acknowledgment requests and confirmed broker outcomes must be labeled according to what was actually observed. Never fill missing events with fabricated telemetry.

### Execution and infrastructure identity

Expose automatically collected information where practical:

- Run, operation, message, and processing-attempt identities.
- Deployment revision and immutable source links for relevant implementation and configuration.
- Provider, region, host/container boundaries, service versions, image identifiers or digests, and runtime health.
- Observation timestamps and freshness, including the last successful environment check.

Distinguish deployment configuration from observed runtime facts. Provider or region metadata may be deployment-supplied; identify its source rather than implying independent verification. Backend unreachability means health is unknown, not that a service is proven stopped. Recorded runs retain their original deployment metadata rather than inheriting the current deployment's identity.

### Source attribution, ordering, and completeness

Each observation identifies its component, evidence type, source timestamp, collection timestamp where available, and persistence status. Preserve relevant clock limitations. Use known causal relationships and per-source ordering; do not imply precise global event ordering from timestamps alone.

Distinguish observed facts from inferred explanations. Report missing telemetry, interrupted collection, and indeterminate outcomes explicitly. A complete-looking timeline is not a correctness criterion. Read final database evidence after processing has quiesced, using a consistent snapshot where needed, and record when it was collected.

### Inspection and reproduction

Each completed experiment provides:

- Relevant source, deployment configuration, and experiment-definition links.
- Expandable source-attributed observations and authoritative business records.
- A sanitized structured execution export containing configuration, identities, deployment metadata, observations, assertions, outcome, completeness, and an export schema version.
- Exact local reproduction steps and pinned dependencies or images sufficient to recreate the setup.
- A comparison of protected and unprotected runs using retained evidence; this need not begin as a general comparison platform.

Reproduction targets equivalent business outcomes and assertions, not identical timestamps or event schedules. Keep provenance read-only and scoped to permitted records. Exclude credentials, session ownership tokens, management access, and unrestricted logs. A sample export must be labeled as sample data.

## 9. Common experiment specification

Each experiment defines:

| Area | Required content |
|---|---|
| Question and consequence | Architectural question, realistic business effect, and decision context |
| Model and assumptions | Components, topology, transaction boundaries, recovery assumptions, and limitations |
| Identity and acceptance | Run, operation, message, and attempt identities; durable acceptance boundary |
| Properties | Safety, liveness, accounting invariants, and testable assertions |
| Scenarios | Hypotheses, bounded inputs, checkpoints, and expected outcomes |
| Lifecycle | Ownership, initialization, execution limits, reset, cleanup, and interruption recovery |
| Evidence | Authoritative records, source attribution, uncertainty, clock limitations, and completeness |
| Provenance | Execution environment, component versions, revision, and presentation mode |
| Reproduction | Exact configuration, commands, dependencies, and export format |
| Evaluation | Observed versus predicted results, tradeoffs, second-order consequences, and review-gate evidence |

Use this as a documentation contract, not a requirement to build a generic software framework.

## 10. Implementation roadmap

Development is milestone-driven. Experimental correctness precedes construction of the guided interface.

| Phase | Objective | Definition of done |
|---|---|---|
| **1. Formalize the first experiment** | Define account-credit invariants, acceptance, identities, transaction boundaries, checkpoints, evidence, lifecycle, and limits | Experiment specification contains testable assertions and resolves acceptance and recovery semantics |
| **2. Implement and validate locally** | Build RabbitMQ, PostgreSQL, worker, and minimal controller | Automated scenarios reproduce normal, duplicate, protected, and interrupted outcomes; assertions and reset checks pass; authoritative evidence is inspectable |
| **3. Create the guided interface** | Build React controls, business-state comparisons, source-attributed timelines, explanations, and provenance inspection | Complete guided local demonstration communicates verified behavior and can export its evidence |
| **4. Validate teardown and reconstruction** | Remove and recreate the local environment from version-controlled Docker Compose configuration | Scenarios pass from a clean environment with no undocumented steps or leftover state |
| **5. Deploy publicly** | Deploy Cloudflare frontend and DigitalOcean backend; enforce public limits and validate cloud lifecycle | External visitors can use a bounded experiment; ownership and recovery work; archive/restore and fresh cloud reconstruction have been exercised |
| **6. Obtain feedback and refine** | Evaluate technical validity, explanatory validity, and evidentiary credibility | Recorded reviews support a decision to improve depth, presentation, or the experiment catalog |

The backend portion of the technical gate—automated scenario assertions, evidence inspection, recovery, and reset—must pass before Phase 3. Complete reconstruction and deployment checks in their respective phases, and recheck affected assertions after integration changes. Lightweight feedback can begin with the local walkthrough; do not wait until public deployment to discover that the explanation fails. All three gates must pass before declaring the experiment complete or moving to a second experiment.

## 11. Three independent quality gates

### Technical validity

Does the system establish what the demonstration claims?

Require repeatable automated assertions for the scenario matrix, safety and accounting properties, recovery under stated assumptions, checkpoint behavior, evidence integrity, reset, and teardown/reconstruction. Distinguish an intended business failure in an unprotected scenario from a malfunctioning experiment.

### Explanatory validity

Does the experience communicate architectural consequences and engineering judgment?

Ask reviewers to predict an outcome before execution, interpret the resulting evidence, and identify a situation where the mitigation is insufficient. Record their reasoning and misunderstandings rather than relying only on reported understanding. Evaluate the fast entry point as well as the deeper technical walkthrough.

### Evidentiary credibility

Can a reviewer identify the infrastructure, trace claims to evidence, understand observational limits, and independently reproduce the behavior?

Check runtime and source identity, attribution and freshness, authoritative records, missing-event handling, mode labels, sanitized exports, and local reproduction. Seek an independent reproduction using documented instructions rather than relying solely on the author's environment.

A visually compelling demonstration with questionable measurements fails technical validity. A correct demonstration without meaningful insight fails explanatory validity. An apparently live demonstration with opaque or unverifiable execution fails evidentiary credibility.

## 12. Infrastructure lifecycle and cost management

Support extended inactivity without retaining compute unnecessarily.

### Active

DigitalOcean runs the experimental services; Cloudflare serves the interface. Monitor resource use and keep public controls bounded.

### Archive

Stop admission, drain or terminate active runs explicitly, and preserve selected execution records and required configuration. Create and verify a snapshot before destroying the Droplet. Check for remaining billable resources. Keep source, deployment definitions, documentation, and retained evidence available independently of that Droplet.

The frontend may remain online with an explicit archived status and labeled recorded executions. Never present stored traces or animations as current runtime activity.

### Restore and reconstruct

Exercise both paths:

1. Restore a snapshot, recover network and DNS configuration, and validate services and experiments.
2. Provision a fresh environment from version-controlled definitions, supply documented secrets securely, initialize known state, and execute the same smoke tests without snapshot dependence.

Test local reconstruction before public deployment and both cloud paths during deployment acceptance. Snapshots are a convenience, not the sole source of truth.

Classify data as disposable run state, retained evidence, or required configuration. Set retention and storage limits explicitly. Do not promise preservation of every transient run indefinitely.

### Budget discipline

Use the smallest measured compute capacity that supports the experiment. Target economical static frontend hosting. Revalidate provider pricing and allowances before provisioning, account for retained snapshots and other storage, and record an active and archived cost estimate. Add resources only with a defined purpose and cost review.

## 13. Feedback loop and decision records

Use **Define → Implement → Observe → Evaluate → Adjust** at every milestone.

Keep two separate feedback streams: experimental correctness and effectiveness of the explanation. Assess credibility across both. At each review, record:

- Prediction and assumptions, linked to scenario definitions and source revision.
- Actual observations, run identifiers, authoritative records, and evidence gaps.
- Incorrect assumptions, infrastructure distortions, and unnecessary complexity.
- Reviewer predictions, interpretations, and identified limitations.
- Whether the evidence supports an engineering decision and communicates senior-level judgment.
- The next adjustment, its reason, and how it will be evaluated.

Maintain lightweight Architecture Decision Records for consequential choices: context, alternatives, decision, tradeoffs, and conditions for revisiting it. Keep milestone review notes for experimental and presentation feedback. Repeat affected checks after changes; use gate failures to guide the next work rather than automatically adding features.

## 14. Candidate experiment backlog

Choose subsequent experiments for architectural and portfolio value after the first passes all three gates.

| Topic | Candidate question | Concepts |
|---|---|---|
| Commit uncertainty | How does a worker recover when it cannot tell whether a credit committed? | Ambiguous outcomes, reconciliation, retry identity |
| External side effects | What happens when database state commits but an external action does not? | Atomicity boundaries, retries, coordination tradeoffs |
| Caching | How do invalidation and stampedes affect correctness and load? | Consistency, contention, latency |
| Replication | When does lag violate read-your-writes expectations? | Consistency, latency, availability, failover |
| Backpressure | What happens when producers outpace consumers? | Queuing, saturation, load shedding |
| Network faults | How do latency and disconnections propagate? | Timeouts, retries, cascading failures |
| Event streaming | How do partitioning and replay affect application guarantees? | Ordering, offsets, consumer groups |
| Coordination | What happens when leadership and lease assumptions fail? | Split-brain risks, fencing, recovery |
| Data access | When do access patterns change the appropriate storage design? | Query plans, indexing, storage tradeoffs |

These are candidates, not commitments. A second domain should test whether shared platform abstractions are warranted; extensions should deepen the first experiment where that produces more useful evidence.

## 15. Portfolio presentation and initial success criteria

Use progressive disclosure. Start with the business consequence, bounded controls, and observed outcome. Show compact deployment identity, presentation mode, and evidence freshness nearby. Let interested reviewers expand authoritative records, execution traces, topology, assumptions, exports, and revision-specific source links.

Provide a concise introduction, accurate system diagram, guided scenario, deeper controls, limitations, and implications for real architectural decisions. The repository must document local execution, experiment definitions, deployment configuration, and decisions. Avoid forcing every visitor through raw logs or observability dashboards.

The first significant checkpoint is **one public, functioning, reproducible experiment that passes all three quality gates and helps demonstrate the author's engineering competence**.

It must establish that:

- A visitor can reproduce duplicate credits and compare transactional prevention.
- Safety, liveness assumptions, and transaction boundaries are explicit.
- Evidence is authoritative where claimed, attributed, inspectable, and honest about uncertainty.
- Runtime identity and physical topology are visible without exposing management access.
- Live, recorded, simulated, and inferred content are distinguishable.
- Anonymous runs have bounded controls, separate state and results, verified cleanup, and recovery.
- Local and deployed scenarios support the same assertions, with documented environmental limitations.
- Snapshot restore and fresh reconstruction work, and archived mode communicates its status honestly.
- Technical reviewers can interpret the results, challenge assumptions, and explain when the mitigation is insufficient.

Use that evidence to decide whether the next investment belongs in experimental depth, presentation, broader coverage, or platform improvements.

---

**Working philosophy:** Favor evidence over spectacle, depth over breadth, and useful engineering decisions over technological novelty. Evolve the architecture in response to observed requirements.

**Standard for inclusion:** Demonstrate what a system does, why its behavior matters, what the evidence establishes, and how an experienced engineer would use the findings to defend a decision.
