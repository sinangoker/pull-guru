# Pull Guru — Base Specification

## 1. Purpose

Pull Guru is an AI-powered merge request (MR) and pull request (PR) reviewer for Java projects. It helps developers identify actionable defects before merging by reviewing changed code in its repository context and publishing concise findings with evidence and suggested fixes.

This document defines a proposed MVP. Hosting provider, implementation stack, and deployment choices remain open unless explicitly specified below.

## 2. Goals

- Find correctness, security, reliability, and meaningful performance problems in changed Java code.
- Explain each finding using a concrete failure scenario and relevant code locations.
- Reduce review effort while keeping noise and duplicate comments low.
- Support Maven and Gradle projects, including multi-module repositories.
- Keep human reviewers responsible for approval and merge decisions.

## 3. MVP Scope

### Included

- One Git hosting integration, selected before implementation: GitLab MRs or GitHub PRs.
- Reviews triggered when a request is opened, reopened, marked ready, or updated with new commits.
- A manual rerun action for authorized repository contributors.
- Java source and test changes, plus relevant build and configuration changes.
- Repository-aware analysis using the diff, surrounding code, referenced types, and related tests within configured limits.
- Inline findings and one review summary, associated with the reviewed commit.
- Repository-level configuration and structured operational logs.

### Excluded

- Automatic approval, merging, or pushing code changes.
- General-purpose reviews for other programming languages.
- Whole-repository audits unrelated to the proposed changes.
- Style and formatting comments already covered by deterministic tooling.
- Executing repository builds, tests, plugins, or scripts in the MVP.
- Training models on customer source code.

## 4. Users and Main Flow

**Developer:** receives findings, applies fixes, and requests another review.

**Maintainer:** enables the integration, configures review rules, and monitors review quality.

1. The hosting provider sends an authenticated event.
2. Pull Guru validates the event, checks repository permissions and configuration, and records a review job for the current head commit.
3. A worker fetches the diff and bounded repository context using read-only source access.
4. The worker selects relevant files and analyzes them against the review criteria.
5. Findings are validated, filtered, deduplicated, and mapped to valid diff positions.
6. Pull Guru checks that the head commit is still current, then publishes inline findings and a summary.
7. Subsequent commits produce a new review; superseded jobs do not publish stale findings.

Draft requests are skipped by default. A force-push invalidates work for the previous head commit.

## 5. Review Criteria

The reviewer prioritizes defects introduced or exposed by the change. Existing issues are reported only when the change makes them newly relevant.

| Area | Examples |
| --- | --- |
| Correctness | Null handling, boundary conditions, incorrect equality, broken control flow, invalid assumptions |
| Java semantics | `equals`/`hashCode` contracts, collection behavior, generics misuse, exception handling |
| Concurrency | Shared mutable state, race conditions, deadlocks, unsafe publication, incorrect asynchronous error handling |
| Resource management | Unclosed streams or connections, executor lifecycle problems, missing cleanup |
| Security | Injection, missing authorization, unsafe deserialization, path traversal, exposed secrets |
| Persistence | Transaction boundaries, query behavior, data consistency, accidental bulk operations |
| Performance | Unbounded work or allocation, repeated remote calls, N+1 queries, blocking critical execution paths |
| Compatibility | Breaking public contracts, serialization changes, incompatible configuration or schema assumptions |
| Tests | Missing coverage for a specific identified risk, ineffective assertions, nondeterministic behavior |

Framework-specific checks, such as Spring transaction or dependency injection behavior, apply only when repository evidence establishes that framework and its relevant usage. Language features are evaluated against the Java version declared by the project. Missing or conflicting version information is disclosed as a limitation.

## 6. Finding Contract

Every published finding contains:

- A short title describing the defect.
- Severity: `critical`, `high`, `medium`, or `low`.
- Confidence: `high`, `medium`, or `low`.
- A file path and line or small line range in the reviewed revision.
- An explanation of the trigger, observed code behavior, and expected impact.
- Supporting code evidence and an actionable remediation suggestion.
- A stable identifier for deduplication across reruns.

Severity expresses impact; confidence expresses strength of evidence. Critical findings describe severe security or data-integrity failures; high findings describe major functional failures; medium findings describe bounded functional or reliability defects; low findings describe minor actionable defects.

By default, publish only high-confidence findings of medium severity or above. A finding must identify a plausible execution path or violated contract; generic best-practice advice is insufficient. Never claim that code compiled or tests passed unless an actual result was supplied and attributed.

If a finding cannot be attached to a valid diff position, include it in the summary with a revision-specific code reference. A review with no findings says “No actionable findings identified” and states its coverage and limitations.

## 7. Repository Configuration

Use an optional `.pull-guru.yml` file with a versioned, validated schema. Load policy from the target branch so that a proposed change cannot silently weaken its own review policy. Document defaults and reject invalid configuration with an actionable status message.

Example proposed configuration:

```yaml
version: 1
enabled: true
review_drafts: false
java:
  version: auto
review:
  minimum_severity: medium
  minimum_confidence: high
  max_findings: 10
  max_files: 50
  max_changed_lines: 5000
  exclude:
    - "**/generated/**"
    - "**/target/**"
    - "**/build/**"
    - "**/vendor/**"
```

Java version overrides must accept an explicit major version. Configuration may narrow deployment limits but cannot raise server-enforced resource or security limits. Credentials and model API keys must never be stored in repository configuration.

## 8. Architecture

- **Provider adapter:** authenticates events, fetches source and request metadata, and publishes review results.
- **Job coordinator:** deduplicates events, queues jobs, manages retries, and cancels superseded work.
- **Context collector:** selects relevant files and references within file, byte, and token budgets.
- **Review engine:** applies review instructions and calls a model through a provider-independent interface.
- **Finding validator:** validates structured output, verifies paths and lines, enforces thresholds, and removes duplicates.
- **Publisher:** reconciles previously published bot findings and updates the review summary without modifying human comments.
- **Persistence layer:** stores job state, reviewed revisions, finding identities, and publication identifiers.

Minimum job states: `queued`, `running`, `completed`, `failed`, `cancelled`, and `skipped`. Record coverage separately as `full` or `partial`; completed analysis does not imply complete coverage.

A review identity includes provider, repository, request number, head commit, and effective policy version. Manual reruns create distinct attempts under that identity. Publication must remain idempotent after retries and worker restarts.

## 9. Safety and Data Handling

- Verify webhook signatures or provider-specific authentication and reject unauthorized requests.
- Grant only permissions needed to read repository content and publish review comments or statuses.
- Treat source files, comments, descriptions, and repository instructions as untrusted data. They cannot override review policy, request credentials, or trigger tools.
- Do not execute repository code in the MVP. Any later execution requires an isolated environment with explicit resource and network controls.
- Send only context necessary for the review to the configured model provider. Deployment must explicitly select an approved provider and its data-handling policy.
- Redact detected credentials from model input, logs, and published output; report the location of a suspected secret without reproducing it.
- Isolate repository and tenant data in storage, caches, and job processing.
- Encrypt credentials and stored sensitive data, use encrypted transport, and restrict operational access.
- Keep source snapshots ephemeral and delete them at job completion or timeout. Define retention and deletion settings for job metadata and findings before deployment.

## 10. Reliability and Limits

- Acknowledge valid events after durable enqueueing; perform analysis asynchronously.
- Retry transient provider and model failures with bounded exponential backoff, respecting rate limits.
- Use configurable job deadlines, token budgets, concurrency limits, and per-review cost ceilings.
- Skip binary files and excluded paths. Mark reviews partial when relevant content exceeds limits or cannot be fetched.
- Publish a clear failed or partial status when analysis is incomplete; do not present incomplete analysis as a clean review.
- Verify structured model output before publication. Invalid output may receive one bounded repair attempt before the job fails.
- Reconcile partial publication after failures so retries do not duplicate comments.
- Capture review duration, queue delay, token usage, estimated cost, failure rate, and finding counts without logging raw source by default.

Initial performance target: finish 95% of reviews of up to 20 Java files and 1,000 changed lines within five minutes after worker execution starts, excluding hosting-provider outages. Validate this target against a representative benchmark before release.

## 11. Acceptance Criteria

1. A configured repository receives a summary and valid inline findings after an eligible request event.
2. Representative Maven and Gradle fixtures, including multi-module projects, are reviewed using their declared Java version and relevant module context.
3. Duplicate webhook delivery and retries after partial publication do not create duplicate findings.
4. Updating or force-pushing a request prevents superseded jobs from publishing against the new revision.
5. Excluded files are not sent to the model; exceeded budgets result in explicit partial coverage.
6. Every finding satisfies the finding contract and references code in the reviewed revision.
7. Invalid configuration, unavailable models, provider rate limits, and malformed model output produce clear, recoverable outcomes.
8. Prompt-injection fixtures cannot change policy, expose credentials, or cause repository code execution.
9. Automated tests cover event authentication, configuration validation, job transitions, diff mapping, deduplication, and provider/model error handling using deterministic doubles.
10. A versioned benchmark includes known Java defects and clean changes. Before pilot release, maintainers manually validate at least 80% precision for published findings and measure recall separately. Record benchmark size, category coverage, model version, and prompt version with results.

## 12. Delivery Stages and Open Decisions

**Stage 1 — Foundation:** select the first hosting provider, runtime, model provider, deployment environment, persistence layer, and retention policy; implement authenticated events and durable jobs.

**Stage 2 — Review pipeline:** implement context collection, structured analysis, finding validation, configuration, and comment publication.

**Stage 3 — Pilot:** verify acceptance criteria, evaluate precision and recall, tune budgets, and run on selected Java repositories with maintainer feedback.

**Later candidates:** a second hosting provider, integration with existing static-analysis reports, developer feedback on findings, and opt-in suggested patches.

Before implementation, also decide supported Java versions, expected repository sizes and review volume, acceptable cost per review, deployment tenancy, and whether a review status should ever block merging. The MVP is advisory by default.
