# AWS Model Evaluation and Governance Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make NBA model training results reproducible and promotion decisions depend on frozen, game-disjoint evaluation evidence instead of job completion or a single headline metric.

**Architecture:** Keep training as a separate SageMaker lifecycle. Each job consumes a versioned dataset/split manifest and pinned environment, emits named metrics plus a complete evaluation report, and may become a registered candidate only when acceptance gates pass. This plan does not deploy an inference endpoint or assume SageMaker Batch/Processing availability; those remain gated by account permissions, competition requirements, and the repo's SageMaker Training Job-only constraint.

**Tech Stack:** Python 3.12, SageMaker Training Jobs in `us-east-1`, S3 versioning, JSONL/JSON manifests, existing YOLO/RF-DETR training code, SageMaker Model Registry candidate metadata.

**Spec:** `E:\AWS-NBA-Workspace\outputs\aws-video-analysis-architecture-upgrade-spec.md`; canonical training contract: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\docs\training\2026-10-07-AWS-GPU数据集完整规范与YOLO-RF-DETR-PK执行方案.md`, `docs\AWS_TRAINING_HANDBOOK.md`, and `AGENTS.md`.

## Global Constraints

- Use account `766815611718`, region `us-east-1`, and AWS CLI profile `nba` for authorized AWS CLI work.
- Do not push or edit GitHub-hosted files; all implementation commits remain local unless separately authorized.
- A training job is not a model-quality result. Preserve dataset, split, source-license, dependency/image, code, model, and evaluator hashes with every report.
- Split by source game/video and time; adjacent frames must not cross train/validation/holdout. A frozen holdout cannot be used for tuning.
- The currently approved AWS path is SageMaker Training Job. Do not assume Batch Transform, Processing, Endpoint, EC2 GPU, or EKS is permitted or provisioned.
- Do not promote a model without per-class metrics, a clean game-disjoint holdout, domain-shift evidence with ground truth, and a passing downstream compatibility check.
- Repository training review records a 24-image player test set and a 36-image ball/foot/player test set; treat these as too small for broad generalization claims. Investigate the ball/foot validation-to-test drop before promotion.
- Fix and pin the RF-DETR dependency set before rerunning: the latest failed job `rfpk-rtrain-r1d-1007-222152` upgraded a PyTorch 2.1/CUDA 12.1 image to Torch 2.11/CUDA 12.6, NumPy 2.2.6, Transformers 5.17, and incompatible Triton/Protobuf packages; the process then failed while importing SciPy from scikit-learn. An earlier attempt also failed on `BackboneConfigMixin`. Retain the image digest and complete lock with each result.
- No AWS training job, registry mutation, or paid resource operation is authorized by preparing this plan.

## Review Focus

- Adjacent frames or the same game leak across splits: audit source-group and time intervals before starting training. Pin in Task 1.
- A manifest points at changed or missing data: compare S3 object hashes and counts with the frozen manifest; reject the run. Pin in Task 1.
- A container dependency changes between experiments or RF-DETR import breaks: verify pinned versions and a no-GPU import smoke before job submission. Pin in Task 2.
- A high aggregate score hides a weak class or a small test set: require per-class metrics, confidence intervals/sample counts, and holdout status. Pin in Task 3.
- A model candidate is registered without trustworthy evaluation or downstream compatibility: block registration/promotion and preserve the failure report. Pin in Task 4.

---

### Task 1: Audit and freeze dataset and split manifests

**Files:**
- Create: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\tools\aws_training\manifest_audit.py`
- Create: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\tools\aws_training\split_manifest.schema.json`
- Create: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\tools\aws_training\test_manifest_audit.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\docs\training\2026-10-07-AWS-GPU数据集完整规范与YOLO-RF-DETR-PK执行方案.md`

**Interfaces:**
- Produces `audit_manifest(path: Path) -> dict[str, Any]` with dataset ID, game/source groups, time ranges, train/validation/holdout membership, file count/bytes/SHA-256, license references, and `DRAFT`, `AUDITED`, or `FROZEN` readiness.
- A frozen split records `holdout_locked_at`, touched history, minimum temporal gap, and the immutable manifest SHA.

- [ ] **Step 1: Add failing tests** for duplicate image/label hashes, missing license/source, mismatched image/label pairs, adjacent-time leakage across splits, touched holdout, and a clean split passing audit.

```python
def test_adjacent_frames_cannot_cross_train_and_holdout(tmp_path):
    manifest = make_manifest(train=("hou-dal", 3400, 3420), holdout=("hou-dal", 3425, 3440))
    result = audit_manifest(manifest)
    assert result["readiness"] == "DRAFT"
    assert "temporal_leakage" in result["errors"]
```

- [ ] **Step 2: Run** `python -m pytest tools/aws_training/test_manifest_audit.py -q`; confirm the audit cases fail before implementation.
- [ ] **Step 3: Implement the auditor** using source game/video IDs and time intervals, not random frame-level splitting. Emit every rejection with a machine-readable reason and never rewrite source data.
- [ ] **Step 4: Add the split JSON Schema** and document the freeze command/output layout in the AWS GPU data specification.
- [ ] **Step 5: Run focused tests** and verify a touched or overlapping holdout stays `DRAFT` and cannot be reported as clean.
- [ ] **Step 6: Commit locally** with message `feat: audit frozen NBA training manifests`; do not push.

### Task 2: Pin reproducible SageMaker training environments and named metrics

**Files:**
- Create: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\tools\aws_training\metric_definitions.py`
- Create: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\tools\aws_training\test_metric_definitions.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\tools\ext\aiball\training\player_detection.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\tools\ext\aiball\training\ball_detection.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\tools\ext\aiball\training\keypoint_court_train.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\docs\AWS_TRAINING_HANDBOOK.md`

**Interfaces:**
- Produces a common `metrics.json` with `schema_version`, `dataset_manifest_sha256`, `split_manifest_sha256`, `code_sha`, `image_digest`, `model_family`, `class_metrics`, `latency_ms`, and `evaluation_status`.
- Each Training Job publishes named SageMaker metrics and the same metric values in its S3 output report; SageMaker `FinalMetricDataList` and the durable report must agree.

- [ ] **Step 1: Add failing metric tests** for empty metrics, missing class values, non-finite scores, missing lineage hashes, disagreement between named SageMaker metrics and report values, and valid detector/keypoint metrics.
- [ ] **Step 2: Run** `python -m pytest tools/aws_training/test_metric_definitions.py -q`; confirm missing validation is exposed.
- [ ] **Step 3: Add metric definitions and report emission** to training entry points while preserving their existing output artifacts and SageMaker Training Job API. Include per-class precision/recall/AP50/AP50:95 for detection and PCK/reprojection metrics for court keypoints where labels support them.
- [ ] **Step 4: Pin dependency sets** for YOLO and RF-DETR training. Install RF-DETR in an isolated, version-locked environment that does not replace the managed image's Torch/CUDA stack; smoke-import `rfdetr`, `transformers`, scikit-learn, and SciPy together. Pin mutually compatible NumPy, SciPy, scikit-learn, Transformers, Triton, Protobuf, and RF-DETR versions.
- [ ] **Step 5: Run CPU-only tests and import smoke** with `python -m pytest tools/aws_training/test_metric_definitions.py -q` and the repository's pinned RF-DETR environment; do not launch a SageMaker job in this step.
- [ ] **Step 6: Commit locally** with message `feat: publish reproducible SageMaker model metrics`; do not push.

### Task 3: Evaluate model candidates on a fixed game-disjoint holdout

**Files:**
- Create: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\tools\aws_training\evaluate_candidate.py`
- Create: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\tools\aws_training\test_evaluate_candidate.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\tools\court_v2\07_eval_gold.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\tools\court_v2\court_h_eval.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\docs\training\2026-10-07-AWS-GPU数据集完整规范与YOLO-RF-DETR-PK执行方案.md`

**Interfaces:**
- Produces `evaluate_candidate(metrics_path, acceptance_policy) -> EvaluationReport` with `PASS`, `BLOCKED`, or `FAIL`, and reasons for every gate result.
- Report includes per-class score and sample count, dataset/split/model/evaluator hashes, validation-vs-holdout deltas, domain-shift subset results, and evaluation date.

- [ ] **Step 1: Add failing tests** for small sample counts, validation-to-test regression, a holdout manifest mismatch, absent domain-shift ground truth, a missing class, and a candidate whose metrics meet every configured threshold.
- [ ] **Step 2: Run** `python -m pytest tools/aws_training/test_evaluate_candidate.py -q`; confirm non-promotion cases fail closed.
- [ ] **Step 3: Implement an explicit acceptance policy** with thresholds sourced from the official rubric or a documented project decision. Until thresholds are supplied, the report must say `BLOCKED_THRESHOLD_UNSET` rather than invent a passing score.
- [ ] **Step 4: Connect existing golden court evaluation** to the common report schema without changing the frozen evaluation set. Keep the Macau images without ground truth in a separate stress/domain-shift section, never as accuracy evidence.
- [ ] **Step 5: Run CPU-only report tests** and verify the current small player/ball test sets are marked insufficient for broad generalization.
- [ ] **Step 6: Commit locally** with message `feat: gate model candidates on holdout evidence`; do not push.

### Task 4: Register only passing model candidates

**Files:**
- Create: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\tools\aws_training\register_candidate.py`
- Create: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\tools\aws_training\test_register_candidate.py`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\docs\AWS_TRAINING_HANDBOOK.md`
- Modify: `E:\AWS-NBA-Workspace\NBA-AWS-CV-Agent\docs\AWS_SERVICE_PROOF.md`

**Interfaces:**
- Produces `register_candidate(report, model_artifact_uri, *, dry_run: bool) -> dict[str, str]`; default execution is dry-run. Live Model Registry writes require a later explicit AWS authorization and verified IAM support.
- Registration metadata contains report SHA, model artifact SHA, dataset/split manifest SHAs, image digest, code SHA, model family, and evaluation gate state.

- [ ] **Step 1: Add failing tests** proving `FAIL`, `BLOCKED`, missing model hash, or mismatched report/model lineage never calls `create_model_package`; a `PASS` dry-run emits the exact request without AWS calls.
- [ ] **Step 2: Run** `python -m pytest tools/aws_training/test_register_candidate.py -q` and verify the negative cases do not reach the boto3 client.
- [ ] **Step 3: Implement dry-run registration payload generation** and isolate the live boto3 call behind an explicit `dry_run=False` argument with region fixed to `us-east-1`.
- [ ] **Step 4: Update the training handbook and service proof matrix** to distinguish training job completion, evaluated candidate, registry registration, and deployed inference; do not claim an endpoint or inference runtime proof.
- [ ] **Step 5: Run CPU-only governance tests**. Keep account registration and GPU training out of this local plan execution.
- [ ] **Step 6: Commit locally** with message `feat: register evaluated NBA model candidates`; do not push.

## Self-Review

- Spec coverage: manifests/splits, named SageMaker metrics, the RF-DETR dependency failure, the small test sets, the ball/foot validation-to-test gap, game-disjoint holdout evaluation, model candidate registration, and promotion controls each have a task.
- Scope: this plan covers training/evaluation governance only. Video event orchestration is in `aws-video-evidence-pipeline-plan.md`; NBA Knowledge Base ingestion is not part of model governance.
- Placeholder scan: no TODO/TBD steps remain. Acceptance thresholds are explicitly blocked until supplied by the official rubric or a recorded project decision; no threshold is invented here.
- Review Focus coverage: every listed data leakage, integrity, dependency, sample-size, and registration failure has a corresponding test in its owning task.
- Constraint check: no SageMaker Batch/Processing/Endpoint or EC2/EKS runtime is assumed by this model-governance plan, and no paid job or live registry mutation is started by local implementation.
