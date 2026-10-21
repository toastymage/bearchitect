# Modernizing Legacy Enterprise Applications
## A Practical Technical Architecture and Implementation Guide

### 1. Executive Summary

Legacy enterprise applications frequently combine workflow engines, scheduled batch processing, database stored procedures, service APIs, and server-side user interfaces within a tightly coupled application.

Modernization should focus on eliminating verified technical constraints rather than replacing technologies simply because they are old.

This guide presents a concrete reference architecture based on five principles:

1. Separate ingestion, batch, and online execution.
2. Retain the existing relational database and proven processing logic initially.
3. Replace implicit processing dependencies with explicit, restartable jobs.
4. Isolate business workflow management from technical job orchestration.
5. Introduce standardized execution controls, observability, and regression testing.

The recommended target is a modular architecture with separately deployable workloads, not a distributed microservices architecture.

### 2. Proposed Target Architecture

The reference implementation assumes an existing Java-based enterprise application with a relational database, scheduled data feeds, legacy ETL processing, Activiti workflows, and a JSP-based interface.

#### 2.1 Logical architecture

**Primary data-processing path**

External Sources → Ingestion Worker → Raw Data Archive and Staging → Job Coordinator → Batch Worker → Processing Core → Validated Outputs → Publication

**Online interaction path**

Browser → Web Application → Application APIs → Processing and Workflow Services → Operational Database

**Shared infrastructure**

Relational Database · Enterprise Scheduler · Identity and Access Management · Secrets Management · Logging and Monitoring · CI/CD

The database is initially shared, but responsibility for reading and modifying different datasets is explicitly defined.

#### 2.2 Component responsibilities

| Component | Responsibility | Recommended implementation |
|---|---|---|
| Ingestion Worker | Acquire, validate, archive, and normalize incoming data | Spring Boot-based worker or approved ETL replacement |
| Job Coordinator | Manage dependencies, execution status, retries, and scheduling integration | Enterprise scheduler with application job-control tables |
| Batch Worker | Execute transformations, calculations, and batch processing steps | Spring Batch |
| Processing Core | Implement reusable business processing logic | Java modules and retained SQL procedures |
| Operational Database | Store processing data, execution state, and results | Existing relational database |
| Application API | Provide controlled online access to application functions | Spring Boot REST services |
| Web Application | Provide interactive screens and operational controls | Existing JSP initially; optional modern frontend |
| Workflow Component | Manage persistent human tasks, approvals, and exceptions | Retain/upgrade Activiti or evaluate a BPM alternative |
| Publication Adapter | Deliver validated results to authorized consumers | Java-based file/API delivery adapter |
| Observability | Capture execution metrics, traces, alerts, and diagnostic logs | OpenTelemetry and approved monitoring infrastructure |

These are logical components. They do not all require separate deployments.

### 3. Physical Deployment Architecture

The initial target should use three primary runtime types.

#### Runtime A: Ingestion

An independently executable ingestion worker receives external files or records and prepares data for application processing.

Responsibilities include:

- Source acquisition and authentication.
- File validation and schema checks.
- Duplicate detection.
- Raw input archival.
- Transformation into an agreed staging format.
- Loading staging tables.
- Publishing ingestion status.

The ingestion runtime should not contain presentation logic or manage human workflow state.

#### Runtime B: Batch Processing

A separate batch runtime handles scheduled or dependency-driven processing.

Responsibilities include:

- Executing named jobs and steps.
- Calling processing modules and stored procedures.
- Maintaining execution checkpoints.
- Managing retries and restart behavior.
- Recording processing outcomes.
- Preparing validated result sets.
- Initiating controlled publication.

A Spring Boot application containing Spring Batch jobs is one implementation.

Each job runs in its own process or managed execution context. Concurrent execution limits should be configurable by job type and partition.

#### Runtime C: Online Application and APIs

The online runtime supports interactive users and application services.

Initially, a JSP application can remain in this runtime while backend functions are exposed through defined application service interfaces.

Responsibilities include:

- Authentication and authorization.
- Data inquiry.
- Processing-status retrieval.
- Authorized operational actions.
- Exception and approval handling.
- Integration with the workflow engine.

The online runtime should not execute large batch calculations directly.

Long-running actions should submit controlled processing requests and return execution identifiers.

#### Deployment model

| Deployment | Initial arrangement | Future option |
|---|---|---|
| Ingestion Worker | Independent process | Independently scaled workers |
| Batch Worker | Independent process | Parallel processing partitions |
| Online/API Runtime | Existing online application, progressively modularized | Separate web frontend and API runtime |
| Workflow Engine | Embedded or existing deployment | Dedicated runtime if justified |
| Database | Existing shared database | Schema and access boundaries, with later separation only if necessary |
| Scheduler | Existing enterprise scheduler | Retain or replace based on demonstrated limitations |

Containerized deployment is an option, not a prerequisite. The same architecture can be implemented using controlled JVM processes on virtual machines.

### 4. Concrete Application Module Structure

A single source repository can contain multiple independently built applications and shared modules.

An illustrative structure is:

| Module | Contents | Deployment |
|---|---|---|
| `ingestion-worker` | File readers, validation, mapping adapters | Independent executable |
| `batch-worker` | Spring Batch jobs, step definitions, restart handling | Independent executable |
| `api-service` | REST controllers, command/query services, security | Online runtime |
| `web-ui` | JSP pages, or later a separate frontend | Online runtime initially |
| `processing-core` | Reusable processing rules and domain logic | Shared internal library |
| `persistence` | Repository interfaces, SQL access, stored procedure adapters | Shared internal library |
| `workflow-adapter` | Workflow operations behind an interface | Library or separate service |
| `processing-contracts` | Input/output schemas and internal contracts | Shared versioned module |
| `database-migrations` | Version-controlled database changes | Deployment pipeline |

This establishes modularity without requiring every module to become an independent network service.

The batch worker and API application may use the same domain library while having separate deployment and runtime lifecycles.

Shared libraries should be versioned, and dependencies between modules should be controlled to avoid recreating a tightly coupled architecture.

### 5. Data Architecture

The existing relational database should remain the initial system of record.

The main change is clearer separation of data responsibilities.

#### 5.1 Logical schemas

| Schema | Example tables | Purpose |
|---|---|---|
| `staging` | `source_file`, `input_record`, `rejected_record` | Incoming and normalized data |
| `control` | `feed_delivery`, `job_run`, `step_run`, `processing_lock` | Technical execution management |
| `core` | Existing application tables | Established processing state and results |
| `workflow` | Workflow engine tables | Process instances and human tasks |
| `audit` | `change_history`, `publication_history` | Traceability and operational evidence |

These can be actual database schemas or logical groupings where the database platform or existing design makes separate schemas impractical.

**The objective is clear ownership, not immediate database restructuring.**

#### 5.2 Access boundaries

Recommended access rules include:

- Ingestion workers write to staging and ingestion-control structures.
- Batch workers read staged inputs and write authorized core results.
- Application APIs expose defined commands and queries.
- The web interface does not access the database directly.
- The workflow adapter owns engine-specific persistence interactions.
- Reporting and inquiry functions use defined queries or views rather than uncontrolled table access.

A shared database does not imply that every component should be allowed to modify every table.

Where possible, each critical dataset should have a clearly designated writer.

#### 5.3 Example control tables

**`feed_delivery`**

| Field | Purpose |
|---|---|
| `delivery_id` | Unique delivery identifier |
| `source_id` | Identity of the source |
| `business_date` | Effective processing date |
| `file_name` | Original file reference |
| `checksum` | Input content fingerprint |
| `input_version` | Source revision |
| `received_at` | Arrival timestamp |
| `validation_status` | Input validation result |
| `archive_location` | Immutable raw input reference |

**`job_run`**

| Field | Purpose |
|---|---|
| `run_id` | Unique processing-run identifier |
| `job_name` | Processing job |
| `delivery_id` | Input lineage |
| `status` | Execution state |
| `attempt_number` | Recovery tracking |
| `started_at` | Execution start |
| `completed_at` | Execution completion |
| `error_code` | Standardized failure category |
| `result_version` | Output version, if successfully produced |

**`step_run`**

Tracks individual processing stages, their start and completion times, execution counts, checkpoints, and errors.

Spring Batch supplies its own execution metadata. Custom control tables should complement that metadata rather than duplicate every framework field.

#### 5.4 Data lineage

Every published result should be traceable to:

- The specific input delivery and version.
- The processing run.
- The processing logic or mapping version.
- The relevant transformation steps.
- The result version.
- Its publication status.

This allows controlled processing of corrected inputs and investigation of discrepancies.

### 6. Concrete Batch Processing Design

Batch modernization should introduce explicit processing stages, rather than a single large chain with hidden dependencies.

#### 6.1 Reference job pipeline

| Step | Operation | Success condition |
|---|---|---|
| 1 | Acquire input | Input persisted and identified |
| 2 | Validate | Schema and required fields accepted |
| 3 | Stage | Accepted records loaded |
| 4 | Normalize | Standardized records available |
| 5 | Execute calculations | Processing successfully committed |
| 6 | Validate results | Defined reconciliation and completeness checks pass |
| 7 | Prepare publication | Complete output version created |
| 8 | Publish | Authorized consumers receive a committed output |
| 9 | Close execution | Final state and metrics recorded |

Not every implementation requires all nine steps to be separate jobs. Some may be steps within a single job.

#### 6.2 Execution states

A useful execution state model is:

`RECEIVED → VALIDATED → STAGED → PROCESSING → VERIFIED → PUBLISHED`

Alternate states include:

`REJECTED`, `FAILED`, `RETRY_PENDING`, and `CANCELLED`.

Transitions should be validated centrally rather than inferred from ad hoc database updates.

#### 6.3 Restartability and idempotency

Restartability means that a failed job can resume or be rerun under defined conditions.

Idempotency means repeating an operation does not unintentionally duplicate its effects.

They are related but separate properties.

Recommended mechanisms include:

- Durable step checkpoints.
- Unique input identifiers.
- Database constraints against duplicate business effects.
- Defined transaction boundaries.
- Controlled retry policies.
- Processing locks for conflicting runs.
- Separate identifiers for corrected input versions.
- Publication records that prevent duplicate delivery effects.

For example, if a batch fails after processing 70% of its records, recovery should not require restarting unrelated jobs or duplicating already committed calculations.

The correct restart behavior depends on the transaction model. Chunk-based processing can provide checkpoints, but database procedures with large internal transactions may require different recovery strategies.

#### 6.4 Partitioning

Where records can be processed independently, jobs may be partitioned by:

- Source.
- Business date.
- Account or processing group.
- Record range.
- Other stable partitioning keys.

Partitioning should only be enabled where independence and transaction isolation are proven.

The architecture should initially support serial execution and introduce parallelism based on measured performance needs.

### 7. Workflow Architecture

Business workflow and technical job orchestration should be treated as distinct concerns.

#### 7.1 Business workflow

A BPM engine remains appropriate for persistent human processes such as:

- Investigation.
- Approval.
- Exception resolution.
- Review and escalation.

An application-level workflow adapter should expose operations such as:

`startProcess()`

`getTask()`

`completeTask()`

`getProcessStatus()`

`cancelProcess()`

The rest of the application should not depend directly on engine-specific implementation APIs.

This makes it easier to retain, upgrade, or replace Activiti without rewriting unrelated processing functions.

#### 7.2 Technical orchestration

Technical orchestration should manage:

- Scheduled jobs.
- Input dependencies.
- Conditional processing steps.
- Failed-job recovery.
- Job completion and downstream triggers.

A BPM engine is not necessarily required for these operations.

An enterprise scheduler combined with batch execution metadata and application-specific control tables is a reasonable default.

#### 7.3 Workflow migration

A phased approach is preferable:

1. Inventory workflow definitions and active instances.
2. Identify custom engine dependencies.
3. Classify business workflows versus technical orchestration.
4. Establish a workflow adapter.
5. Upgrade or replace only where support, complexity, or operating requirements justify it.

Migration feasibility depends on the exact Activiti version, workflow definitions, extensions, and persistence data.

### 8. Online and Service Architecture

The online application should communicate through defined service interfaces rather than directly coordinating backend implementation details.

#### 8.1 Example APIs

| Endpoint | Purpose |
|---|---|
| `GET /api/v1/jobs` | List processing runs |
| `GET /api/v1/jobs/{runId}` | Retrieve run details |
| `POST /api/v1/jobs/{jobName}/runs` | Request authorized execution |
| `POST /api/v1/jobs/{runId}/retry` | Request controlled recovery |
| `GET /api/v1/feeds/{deliveryId}` | Retrieve input delivery status |
| `GET /api/v1/exceptions` | Retrieve processing exceptions |
| `POST /api/v1/workflows/{id}/actions` | Submit an authorized workflow action |

These interfaces illustrate boundary design, not a requirement to expose all functions publicly.

An API should validate caller permissions, maintain audit records for state-changing operations, and use idempotency controls where requests might be repeated.

Batch execution requests should normally return an execution ID rather than waiting for processing to finish.

#### 8.2 JSP modernization path

The UI can evolve in three increments.

**Increment 1: Retain JSP.**

Remove critical business logic from pages and establish application service boundaries.

**Increment 2: Upgrade framework dependencies.**

Update supported Java/web frameworks and improve authentication, authorization, and automated testing.

**Increment 3: Selective frontend replacement.**

Replace screens with a modern frontend framework where justified by maintainability, compatibility, or usability requirements.

A React or Angular frontend is an option, not a mandatory architectural choice.

The API boundary should be established independently of the frontend technology decision.

### 9. Output Publication and Consistency

Publishing partially processed results is a significant architectural risk.

A processing run should produce a complete, validated result version before exposing it as final.

A possible publication protocol is:

1. Write results to a run-specific staging or result version.
2. Validate record counts and business invariants.
3. Mark the version as verified.
4. Atomically update the active publication reference where the database permits.
5. Deliver or expose that committed version.
6. Record publication outcome and downstream acknowledgement.

For internal database consumers, a versioned view or active-version reference can provide controlled visibility.

For external asynchronous delivery, an outbox table or equivalent durable delivery mechanism can prevent the database transaction and outbound publication process from becoming inconsistently recorded.

An outbox is optional if simple transactional publication or reliable file delivery already meets the requirement.

A message broker should not be introduced solely to make the architecture appear event-driven.

### 10. Failure Handling and Recovery

A concrete design must explain what happens when processing fails.

| Failure | Expected behavior |
|---|---|
| Invalid file | Quarantine input; record validation errors |
| Duplicate delivery | Identify duplicate; prevent unintended reprocessing |
| Missing dependency | Hold dependent execution in a waiting state |
| Stored procedure failure | Roll back or preserve committed checkpoints according to transaction design |
| Worker process crash | Recover execution metadata and resume from a safe point |
| Online application outage | Batch execution continues independently where supported |
| Workflow engine outage | Preserve task state and resume authorized workflow operations |
| Publication failure | Preserve completed result version and retry delivery safely |
| Corrected input | Create a new input version and explicitly identify superseded results |

Recovery design should address both technical consistency and operational usability.

Every failure should have an identifiable execution, a documented recovery action, and sufficient retained data to reproduce the problem where feasible.

### 11. Security, Observability, and Deployment

#### 11.1 Security

The target should use:

- Centralized identity and access management.
- Role-based authorization for online and operational actions.
- Separate application service accounts.
- Least-privilege database access.
- Managed secrets.
- Encryption in transit.
- Audit logging for state-changing operations.
- Controlled administrative and retry operations.

#### 11.2 Observability

Monitoring should capture:

- Input arrival and validation status.
- Job execution counts and durations.
- Failed steps and retry frequency.
- Queue or dependency waiting time.
- Processing throughput.
- Database contention.
- API error rates and latency.
- Publication lag and failures.

Logs and traces should carry a consistent correlation identifier, ideally including `run_id` or equivalent execution context.

Spring Boot observability and OpenTelemetry are reasonable implementation choices in a Java-based architecture.

#### 11.3 Deployment and release management

The minimum desired outcome is that a change to a batch job does not automatically require redeploying the online application.

Recommended release controls include:

- Separate build artifacts for each executable runtime.
- Versioned shared libraries.
- Automated tests.
- Database migration scripts.
- Automated deployment pipelines.
- Environment-specific configuration.
- Controlled rollback procedures.
- Compatibility testing between components sharing the database.

Database changes require particular care because separately deployed applications may temporarily run different code versions against the same schema.

Backward-compatible schema changes and phased cutovers are preferable.

### 12. Technology Selection

The following stack is a reasonable reference option for an existing Java application.

| Layer | Initial recommendation | Alternative |
|---|---|---|
| Programming language | Java on a supported LTS runtime | Retain another supported existing language |
| Application framework | Spring Boot | Approved enterprise framework |
| Batch processing | Spring Batch | Existing enterprise batch framework |
| Scheduling | Retain approved enterprise scheduler | Quartz or platform-native scheduler |
| ETL replacement | Java ingestion or approved ETL platform | Database-centric transformations |
| Workflow | Existing Activiti behind adapter | Supported Activiti version, Flowable, or another BPM solution |
| Database | Existing relational DBMS | No replacement by default |
| SQL access | Spring JDBC or existing persistence approach | jOOQ or another appropriate framework |
| UI | Retain JSP initially | React, Angular, or another supported frontend |
| Observability | OpenTelemetry-compatible instrumentation | Approved monitoring tools |
| Deployment | Separate JVM processes or containers | Kubernetes workloads where infrastructure justifies them |
| Database changes | Versioned migration scripts | Approved database release tooling |

Specific library versions should be selected through compatibility and support assessment.

**Not required in the initial target:** microservices, Kafka, service mesh, separate databases, event sourcing, cloud migration, or a complete frontend rewrite.

### 13. Incremental Implementation Roadmap

#### Phase 0 — Establish the Baseline

Deliver:

- Current deployment topology.
- Runtime and framework inventory.
- Batch dependency graph.
- Workflow classification.
- Stored procedure dependency analysis.
- ETL mapping inventory.
- Failure and recovery baseline.
- Initial regression test datasets.

Exit criterion: the system's dependencies and critical change risks are sufficiently understood to select a representative migration candidate.

#### Phase 1 — Establish Execution Controls

Deliver:

- Input identification and version tracking.
- Job execution identifiers.
- Standard execution state.
- Logging and monitoring conventions.
- Regression comparison framework.
- Controlled retry design.

Exit criterion: one representative processing path can be traced from input receipt to final output.

#### Phase 2 — Replace Legacy Ingestion

Deliver:

- Independently executable ingestion worker.
- Validated staging interfaces.
- Raw input archive.
- New mappings and transformations.
- Parallel legacy-versus-new comparison.
- Feed-by-feed cutover mechanism.

Exit criterion: migrated workloads meet functional-equivalence and operational acceptance criteria.

#### Phase 3 — Isolate Batch Execution

Deliver:

- Separate batch executable.
- Explicit job/step definitions.
- Scheduler integration.
- Restart and recovery controls.
- Database access boundaries.
- Independent deployment and resource configuration.

Exit criterion: selected batch workloads can be deployed and executed without redeploying the online application.

#### Phase 4 — Simplify Workflow and Database Logic

Deliver:

- Workflow adapter.
- Remediation of unsupported workflow dependencies.
- Selective relocation of technical orchestration.
- Refactoring of problematic stored procedures.
- Automated regression tests for modified calculations.

Exit criterion: identified high-risk dependencies have been removed or adequately mitigated.

#### Phase 5 — Optional UI Renewal

Deliver:

- Stable backend APIs.
- Modernized or replaced screens where needed.
- Supported frontend dependencies.
- Automated UI regression tests.

Exit criterion: the relevant security, compatibility, and maintenance objectives have been met.

### 14. Architecture Acceptance Criteria

A target architecture should be evaluated against observable capabilities.

| Capability | Example acceptance criterion |
|---|---|
| Deployment separation | Batch runtime can be deployed independently of online runtime |
| Failure isolation | A failed input does not require unnecessary reprocessing of unrelated inputs |
| Restartability | A failed job can resume or restart from a documented consistent state |
| Duplicate protection | Replayed inputs or delivery retries do not create unintended duplicate outcomes |
| Traceability | Final output can be linked to its input and processing run |
| Functional equivalence | Migrated processing meets defined result-comparison tolerances |
| Workflow isolation | Application code accesses workflow functions through controlled interfaces |
| Database control | All critical write operations have defined ownership and transaction boundaries |
| Observability | Critical processing failures produce identifiable alerts and diagnostic context |
| Maintainability | Representative changes require less unnecessary deployment and regression effort |

These capabilities provide clearer evidence of modernization success than a list of technologies replaced.

### 15. Cost and Investment Considerations

The primary cost drivers include:

- Number and complexity of ETL mappings.
- Coupling among batch, workflow, database, and UI components.
- Quality of existing test coverage.
- Number of interfaces and integration dependencies.
- Required parallel-running periods.
- Complexity of data and processing recovery.
- Scope of retained versus rewritten functionality.

Illustrative US enterprise planning ranges may be:

| Workstream | Planning range |
|---|---:|
| Architecture assessment and prototype | $100K–$250K |
| ETL replacement | $450K–$1.2M |
| Batch isolation | $350K–$800K |
| Workflow remediation | $150K–$600K |
| Database refactoring and testing | $250K–$900K |
| Optional UI modernization | $200K–$600K |

These are hypothetical cost envelopes, not application-specific estimates. Workstreams overlap, and their figures should not be summed without a delivery resource model.

A credible investment case should compare mandatory migration expenditure with the additional cost of targeted improvements.

Value should be quantified using metrics such as change lead time, incident frequency, recovery time, regression effort, unsupported dependencies, and unnecessary deployment scope.

### 16. Architectural Decision Records

To prevent scope expansion, major architectural choices should be recorded with their context and tradeoffs.

A minimal set of decisions is:

| Decision | Preferred initial direction | Revisit when |
|---|---|---|
| Application decomposition | Modular application with separate runtimes | Independent scaling or ownership demands further separation |
| Database | Retain shared relational database | Strong data isolation requirements emerge |
| Batch framework | Standardize restartable Java batch execution | Existing platform offers equivalent capabilities |
| Workflow | Retain behind abstraction initially | Version support or workflow complexity warrants replacement |
| UI | Preserve initially | Security, compatibility, or change cost justifies renewal |
| Messaging | Avoid mandatory message broker | Reliable asynchronous delivery requirements justify it |
| Infrastructure | Use existing approved runtime platform | Capacity or operational constraints require change |

Recording these decisions helps distinguish intentional architectural compromises from undocumented legacy dependencies.

### 17. Final Recommendation

For an application with legacy ETL, scheduled batch workloads, Activiti workflows, substantial database logic, and a JSP interface, the preferred default is a **modular monolith with independently executable workloads**.

The initial target should contain:

- An ingestion worker with input validation, archival, and staging.
- A restartable batch runtime with explicit job execution controls.
- A retained processing core using existing Java and SQL logic.
- A controlled relational database with clearer access boundaries.
- An online application exposing stable service interfaces.
- A workflow abstraction that isolates the BPM engine.
- Controlled output publication and comprehensive execution monitoring.

This design addresses common maintenance and operational limitations without the additional complexity of a full distributed application rewrite.

Further decomposition should occur only when measurable requirements—such as independent scaling, availability, release cadence, or technical ownership—justify it.

**The key architectural outcome is not a larger number of services. It is independent execution, predictable recovery, controlled dependencies, and a demonstrably lower cost of change.**
