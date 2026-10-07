# AWS Video Evidence Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the deployed video path into an idempotent, run-scoped AWS pipeline whose persisted service evidence passes ClaimGate before it can reach the published RunBundle.

**Architecture:** Extend the existing Lambda/CDK/Step Functions path around one validated RunManifest. EventBridge and the existing optional SQS failure boundary start run-scoped media/model work; Fargate remains CPU-only. Workers write staging evidence, a finalizer validates and assembles an immutable RunBundle, and `publish/pointer.py` remains the sole conditional writer of `published/current.json`. AgentCore reads only approved bundle facts and must not block baseline video output.

**Tech Stack:** Python 3.12, AWS CDK v2, Lambda, EventBridge, SQS/DLQ, Step Functions Standard, S3, DynamoDB, ECS/Fargate for CPU media work, existing `aws.optional_services.service_adapter`, CloudWatch/X-Ray, AgentCore.

**Inputs:** User request and `E:\AWS-NBA-Workspace\outputs\aws-video-analysis-architecture-upgrade-spec.md`. `docs\AWS_FULL_MIGRATION_PLAN.md`, `docs\AWS_SERVICE_PROOF.md`, and `aws\optional_services\service_adapter.py` are reference material and repository baseline only; they are not the official score rubric or immutable design requirements.

## Global Constraints

- AWS account `766815611718`, region `us-east-1`; use AWS CLI profile `nba` for all AWS CLI calls.
- Do not push, merge, or otherwise modify GitHub-hosted files; implementation stays in the local clone unless separately authorized.
- Every run carries `run_id`, `config_sha256`, `code_sha`, `input_sha256`, and `expires_at`; add dataset/model version and creation metadata without forwarding secrets or bulky config into Step Functions execution history.
- Workers write only to `staging/{run_id}`; only the conditional S3 pointer writer updates `published/current.json`. DynamoDB is a query/status mirror, never the publication authority.
- Fargate performs CPU compilation, FFmpeg, QA, Polly, and publishing. It does not run CV inference. Any GPU inference backend must be verified as available and permitted before selecting a pre-provisioned EC2/EKS path.
- A service is runtime-proven only when one run has the service job/resource ID, input/output S3 references and SHA-256 values, status, timestamps, and CloudWatch trace/log references in its evidence bundle.
- Optimize for the four competition outcomes (input, analyze, visualize, narrate) and meaningful AWS service use. Do not infer a fixed point value from service count; map each proven integration to the official machine-scoring checks when those checks are available.
- Preserve the baseline path when an optional branch fails; missing or conflicting evidence remains `unknown`/`NOT_AVAILABLE`, never fabricated.
- Do not create or modify AWS resources during local implementation. Cloud deployment and live service calls require a separate user-authorized execution step.

## Review Focus

- Replayed S3 events and API retries: the same run starts at most one execution; conflicting input for an existing run is rejected. Pin in Task 2.
- Missing checksum, invalid manifest, or an encoded/nested raw S3 key: reject or preserve an explicit unknown hash; never treat ETag as SHA-256. Pin in Task 1.
- An optional asynchronous media service times out or fails: retain branch status and baseline output; do not claim the service completed. Pin in Task 3.
- One evidence object is missing, corrupt, or hashes differently at finalization: ClaimGate fails and the current pointer is unchanged. Pin in Task 4.
- A Lambda/EventBridge/SQS partial batch fails: return only failed message IDs for retry and preserve a visible DLQ route. Pin in Task 2.

---

### Task 1: Freeze the RunManifest and evidence contracts

**Files:**
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\api-lambdas\contracts.py`
- Create: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\schemas\run_manifest.schema.json`
- Test: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\tests\test_job_contracts.py`
- Test: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\tests\test_static_contracts.py`

**Interfaces:**
- Consumes: current compatibility envelope returned by `run_context(...)`.
- Produces: `validate_run_manifest(payload: dict[str, Any]) -> dict[str, Any]`, retaining legacy `job_id` callers while enforcing the versioned RunManifest contract for new runs.
- Canonical required run fields: `schema_version`, `run_id`, `job_id`, `input_s3_uri`, `input_sha256`, `config_sha256`, `code_sha`, `region`, `created_at`, `expires_at`; optional evidence fields include `dataset_version`, `model_versions`, and `artifacts`.

- [ ] **Step 1: Add failing contract tests** for complete manifest round-trip, missing required fields, invalid region, invalid SHA-256, non-serializable config, and legacy `job_id` compatibility.

```python
def test_manifest_requires_sha256_not_etag():
    payload = valid_manifest()
    payload["input_sha256"] = '"opaque-etag"'
    with pytest.raises(ValueError, match="input_sha256"):
        contracts.validate_run_manifest(payload)
```

- [ ] **Step 2: Run the focused contract tests** with `python -m pytest aws/tests/test_job_contracts.py -q`; confirm the new cases fail before implementation.
- [ ] **Step 3: Implement normalization and validation** in `contracts.py`, keeping hash checks dependency-free and making `run_context` emit the same normalized fields. Do not silently convert ETag into an input digest.
- [ ] **Step 4: Add JSON Schema validation fixtures** in `test_static_contracts.py` for required fields, region, and hash patterns.
- [ ] **Step 5: Run both focused test files** and confirm legacy API inputs remain compatible while invalid run manifests fail closed.
- [ ] **Step 6: Commit locally** with message `feat: define versioned video run manifest`; do not push.

### Task 2: Make raw-video intake idempotent across S3, EventBridge, SQS, and API retries

**Files:**
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\api-lambdas\submit.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\api-lambdas\raw_event.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\cdk\nbaws\backend_stack.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\cdk\app.py`
- Test: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\tests\test_job_contracts.py`
- Test: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\tests\test_static_contracts.py`

**Interfaces:**
- Consumes: normalized output from Task 1 and canonical `raw/{job_id}.mp4` event shape.
- Produces: one Step Functions execution per `run_id`, DynamoDB status mirror with conditional idempotency, and SQS partial-batch failure responses when the existing opt-in failure boundary is enabled.

- [ ] **Step 1: Add failing tests** for duplicate identical event, same `job_id` with a different input key, malformed EventBridge envelope, URL-encoded S3 key, duplicate SFN execution, and mixed-success SQS batches.
- [ ] **Step 2: Run focused tests** with `python -m pytest aws/tests/test_job_contracts.py aws/tests/test_static_contracts.py -q`; confirm each new retry case fails.
- [ ] **Step 3: Update intake paths** so API submit and raw S3 events use the same manifest normalization and conditional write. Preserve the existing event route and `NBAWS_ENABLE_SQS_BUFFER` behavior; the failure queue remains a retry/DLQ boundary and must not create a second orchestration path.
- [ ] **Step 4: Tighten event source and queue assertions** in `backend_stack.py` and `app.py` so the actual configured upload bucket and raw prefix are explicit, SQS remains opt-in, and retry/DLQ counts and visibility timeout remain bounded.
- [ ] **Step 5: Run focused tests and local template validators** with `python -m pytest aws/tests/test_job_contracts.py aws/tests/test_static_contracts.py -q` and `python aws/scripts/validate_templates.py`; verify duplicate events do not start a second execution.
- [ ] **Step 6: Commit locally** with message `fix: make raw video intake idempotent`; do not push.

### Task 3: Orchestrate run-scoped media and model evidence branches

**Files:**
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\cdk\nbaws\backend_stack.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\optional_services\service_adapter.py`
- Create: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\api-lambdas\service_tasks.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\tests\test_optional_services.py`
- Test: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\tests\test_static_contracts.py`

**Interfaces:**
- Consumes: validated RunManifest from Tasks 1–2.
- Produces: `start_service(manifest, service, options) -> dict[str, str]` returning `service`, `job_id`, and `status`; `poll_service(manifest, service, job_id) -> dict[str, Any]` returning terminal status and output S3 references. Every branch writes only under `staging/{run_id}`.
- Supported initial branches: MediaConvert, Transcribe, Rekognition, and selected-frame Textract. Polly is permitted only after approved commentary exists. SNS is a non-blocking terminal notification.
- CV inference is routed through a verified AWS GPU path; CPU media work remains separate. Select SageMaker Training/Processing/Batch or an endpoint from measured needs and the participant role's actual permissions, not from an old service matrix.

- [ ] **Step 1: Add failing adapter tests** for manifest-to-service input mapping, malformed S3 URI, branch timeout, returned job IDs, polling failure, and no import-time AWS calls.
- [ ] **Step 2: Run** `python -m pytest aws/tests/test_optional_services.py -q` and confirm missing dispatcher/poll behavior fails.
- [ ] **Step 3: Add explicit service start/poll dispatch** in `service_tasks.py`; extend `service_adapter.py` only where an operation is missing. Persist each branch's service ID, input/output URI, SHA-256, timestamps, status, and trace reference. Do not claim asynchronous output before the job reaches a terminal success state.
- [ ] **Step 4: Extend the Step Functions definition** with a bounded parallel sidecar stage, per-branch retries/timeouts, a recorded optional failure result, and a join before ClaimGate. Keep the base CPU render path independently recoverable.
- [ ] **Step 5: Add template contract tests** confirming Fargate task definitions contain no CV model execution, run IDs reach each branch, and each optional service writes under run-scoped staging.
- [ ] **Step 6: Run focused adapter tests and workflow validators** with `python -m pytest aws/tests/test_optional_services.py aws/tests/test_static_contracts.py -q` and `python aws/scripts/validate_workflows.py`; no AWS calls are part of this local verification.
- [ ] **Step 7: Commit locally** with message `feat: orchestrate run-scoped media evidence`; do not push.

### Task 4: Gate and publish an immutable RunBundle

**Files:**
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\api-lambdas\finalize.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\publish\assemble.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\publish\pointer.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\web\index.html`
- Create: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\tests\test_run_bundle.py`
- Test: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\tests\test_static_contracts.py`

**Interfaces:**
- Consumes: branch evidence records from Task 3 and current `artifacts/gold/episode_houdal_0201` assembly conventions.
- Produces: immutable `published/runs/{run_id}/{bundle_sha256}/` objects, a GateReport with `PASS`, `DEGRADED`, or `FAIL`, and a pointer candidate accepted only through the existing conditional `publish/pointer.py` interface.
- AgentCore and the web UI may read only the current pointer and approved RunBundle facts; DynamoDB continues to mirror status and URLs.

- [ ] **Step 1: Add failing tests** for required artifact hashes, missing branch output, corrupted object hash, optional-service degradation, immutable bundle key collision, and stale `If-Match` pointer update.
- [ ] **Step 2: Run** `python -m pytest aws/tests/test_run_bundle.py aws/tests/test_job_contracts.py -q`; verify corrupt or incomplete evidence cannot pass.
- [ ] **Step 3: Implement finalization** to build one GateReport and immutable RunBundle per run. Preserve per-branch failures as visible degradation and require all configured baseline artifacts before `PASS`.
- [ ] **Step 4: Keep publication single-writer**: finalization produces the pointer candidate but delegates conditional pointer mutation to `publish/pointer.py`; a failed ClaimGate or precondition conflict leaves the existing pointer unchanged.
- [ ] **Step 5: Update the web reader and AgentCore handoff** to resolve only `published/current.json`, verify the bundle hash, and show `NOT_AVAILABLE` when the pointer or required bundle is absent. Do not add fallback reads from mutable DynamoDB data.
- [ ] **Step 6: Run focused tests and static contract validators** with `python -m pytest aws/tests/test_run_bundle.py aws/tests/test_static_contracts.py -q` and `python aws/scripts/validate_templates.py`.
- [ ] **Step 7: Commit locally** with message `feat: publish claim-gated immutable run bundles`; do not push.

### Task 5: Produce a runtime evidence report and optional service notifications

**Files:**
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\optional_services\service_adapter.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\api-lambdas\finalize.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\docs\AWS_SERVICE_PROOF.md`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\aws\tests\test_optional_services.py`

**Interfaces:**
- Consumes: GateReport and RunBundle from Task 4.
- Produces: one report row per service and run, including state (`declared`, `adapter wired`, `runtime proof`), AWS job/resource ID, output SHA, trace/log reference, and explicit reason when not run. SNS and Polly remain optional, run-scoped services.

- [ ] **Step 1: Add failing tests** for SNS notification payload linkage to `run_id`, Polly rejection without an approved commentary artifact, and service matrix state not advancing without ARN/job ID plus output hash.
- [ ] **Step 2: Run** `python -m pytest aws/tests/test_optional_services.py -q` and confirm the proof-state checks fail.
- [ ] **Step 3: Add evidence validation** to finalize and the adapter boundary. SNS failures are recorded but do not fail the baseline bundle; Polly reads only ClaimGate-approved text.
- [ ] **Step 4: Update `AWS_SERVICE_PROOF.md`** from actual implementation evidence states; do not mark local tests, CDK synth, or adapter presence as runtime proof.
- [ ] **Step 5: Run focused tests** and confirm each service remains `adapter wired` until a later authorized live run adds real identifiers, output hashes, and traces.
- [ ] **Step 6: Commit locally** with message `docs: track per-run AWS service proof`; do not push.

## Self-Review

- Spec coverage: Tasks 1–2 cover manifest, true raw-video routing, idempotency, and SQS/DLQ behavior; Tasks 3–4 cover run-scoped service outputs, ClaimGate, immutable RunBundle, and pointer publication; Task 5 covers evidence states and completion notification. Training lifecycle and managed NBA Knowledge Base are independent subprojects and are not silently folded into this per-video pipeline plan.
- Reference discipline: AWS migration/proof documents inform the starting inventory but are not treated as official scoring requirements. Service count alone earns no claimed score; every chosen service must support one of the competition outcomes and produce per-run evidence.
- Repository baseline: the plan evolves the current S3 pointer, `RunManifest`, ClaimGate, RunBundle, CPU media worker, and AgentCore path where those contracts still fit. GPU inference placement is a measured decision, subject to current account permissions and quota.
- Placeholder scan: no TBD/TODO or unspecified implementation steps remain. Each task names files, interfaces, tests, commands, outputs, and local-only commit boundaries.
- Type consistency: Task 1 emits a normalized manifest; Tasks 2–3 consume it; Task 3 emits branch evidence; Task 4 validates it and produces the bundle; Task 5 reports proof state from that bundle.
- Review Focus coverage: duplicate/mismatched intake is pinned in Task 2; malformed manifest/hash is pinned in Task 1; asynchronous branch failure in Task 3; corrupted evidence and pointer conflicts in Task 4; partial batch retry behavior in Task 2.
