# AWS Training and Service Audit

**Checked:** 2026-10-07, AWS account `766815611718`, region `us-east-1`, CLI profile `nba`  
**Scope:** Read-only SageMaker, S3, service-job, and CloudTrail inspection. No training was started and no AWS resource was changed.

## Assessment

The account has trained useful detector candidates, but the evidence does not support calling the models production-ready or broadly generalized. Two delivered models have test metrics in S3, while SageMaker's own `FinalMetricDataList` is empty for the sampled jobs. The player test set has only 24 images; the ball/foot/player test score falls materially below validation; and a 9-class detection run was dispatched where pose was expected. The latest RF-DETR job fails during dependency imports, and the latest pose job fails before training because `shutil` is undefined.

There is real AWS service usage beyond the older service matrix: Transcribe, Polly, and Rekognition have successful proof jobs. MediaConvert was called but both requests failed. That proves API activity for the successful services, but not yet a cohesive video-analysis run: the proof jobs do not share a complete RunManifest, persisted output SHA-256 values, ClaimGate result, and published RunBundle. The official rubric material available in the repository describes AWS service diversity as encouraged; it does not provide a fixed point formula for the number of services. Exact score attribution still needs the evaluator's rubric.

## Training Evidence

SageMaker lists **81 jobs**: 54 `Completed`, 16 `Failed`, 10 `Stopped`, and 1 `InProgress`. Model Package Groups, endpoints, transform jobs, and SageMaker Pipelines are empty.

| Candidate | AWS evidence | Readout |
|---|---|---|
| `players_v25i_v1` | Job `nbaws-p25i-v1-1007-100801`, completed. S3 manifest: 1,196 images, 1,140/32/24 train/validation/test, 9 detection classes, test mAP50 `0.85383`, mAP50-95 `0.53541`. | Promising detector result, but 24 test images are too few for a reliable generalization claim. The manifest says the dispatched task was expected to be pose, but the actual data format was 9-class detection; this does not provide COCO-17 pose evidence. No game-disjoint split manifest or dataset/model SHA is recorded in the training manifest. |
| `ball_foot_v1` | Job `nbaws-bfoot-v1-1007-100759`, completed. S3 manifest: 582 images, 510/36/36 split, classes `ball`, `foot`, `player`. Validation mAP50/mAP50-95 `0.86519`/`0.51071`; test `0.75782`/`0.43910`. | Test is lower by `0.10737` mAP50 and `0.07161` mAP50-95. The 36-image test set is small; investigate per-class counts, split leakage, label quality, and domain shift before promotion. |
| RF-DETR keypoint training | `rfpk-rtrain-r1d-1007-222152` failed. CloudWatch shows the base PyTorch 2.1/CUDA 12.1 image was changed in-job to Torch 2.11/CUDA 12.6, NumPy 2.2.6, Transformers 5.17, and other new packages. Dependency conflicts were reported; import then failed in SciPy's `_fitpack_impl` while scikit-learn was loading. Earlier attempt `rfpk-rtrain-r1-1007-055110` failed on `BackboneConfigMixin`. | Environment is not reproducible yet. The newer import smoke passed one symbol check but did not validate the complete `rfdetr` → Transformers → scikit-learn → SciPy stack. Failed output archives were only about 234 bytes, not usable model artifacts. |
| Pose training | `nbaws-pose-s-r3-212411` failed with `NameError: name 'shutil' is not defined`; several preceding pose jobs were stopped. | No successful pose candidate is evidenced by the current SageMaker job inventory. Fix the code and complete an evaluation before claiming pose coverage. |
| Court model evaluation | Earlier S3 evaluation review recorded 116 test images, YOLO detection rate 1.0 and hull IoU `0.7967`; RF-DETR detection rate 1.0 and hull IoU `0.8567`; PCK at 2% diagonal `0.9539`/`0.9355`. Average latency was `0.478`/`1.54` seconds per frame. Macau images had no ground truth. | The metrics are useful candidate evidence, not a clean cross-game acceptance result. The ground-truth-free Macau subset cannot establish accuracy. Keep this historical evaluation separate from current successful SageMaker job counts until its exact manifest and output hashes are linked. |

The player S3 manifest is `awsbatch/model/players_v25i_v1/manifest.json`; the ball/foot manifest is `awsbatch/model/ball_foot_v1/manifest.json`. Their `results.csv` files contain validation curves, but the SageMaker jobs report no `FinalMetricDataList`. The manifests do not bind a frozen game-disjoint split SHA or the model artifact SHA. S3 ETags observed for service outputs are not SHA-256 and must not be reported as such.

The latest RF-DETR job `rfpk-rtrain-r1d-1007-215312` is still `InProgress` but has no start time; its last status is `Pending / Training job waiting for capacity`. It was left untouched.

## Service Runtime Evidence

| Service | Current account evidence | Assessment |
|---|---|---|
| Transcribe | `nbaws-proof-transcribe-001` is `COMPLETED`; input is a 10-second WAV under `awsbatch/proof/inputs/`; transcript object is `awsbatch/proof/transcribe/transcribe.json` (5,438 bytes, versioned). | Successful service job and persisted output. No SHA-256, shared video `run_id`, or ClaimGate/RunBundle linkage was found in this proof record. |
| Polly | Task `ba53f244-a396-4fc7-ba4e-36d8a64ae092` is `completed`; Zhiyu `cmn-CN` MP3 at `awsbatch/proof/polly/voice.<task-id>.mp3` (63,835 bytes, versioned). | Successful service job and persisted audio. No SHA-256 or approved commentary/run bundle linkage was found. |
| Rekognition | `StartLabelDetection` succeeded as job `7fcfef0b692e7b595b689eae5da20120a43cbb1305be00f60e54368fc347fc53`; 5-second MP4 input; `GetLabelDetection` returned `SUCCEEDED` and basketball/person labels. | Successful sidecar proof. It is generic scene evidence, not a player/ball/court model; results were not shown persisted with a run manifest or output SHA. |
| MediaConvert | Two `CreateJob` attempts with `run_id=proof-mc-001`; no completed jobs. One was rejected because MediaConvert could not assume `NBAHackathonWorkloadRole`; the other used invalid output group type `FILE_GROUP`. | Not integrated successfully. Fix both the service trust/role setup and request schema before another authorized proof attempt. |
| SQS | `list-queues` returned no queues. | No deployed queue or DLQ evidence in the account. |
| SNS | Topic `arn:aws:sns:us-east-1:766815611718:nbaws-training-notify` exists; it has no subscriptions. CloudTrail has no `Publish` event. | Resource exists, but no runtime notification proof. |
| Bedrock Knowledge Bases | `list-knowledge-bases` returned no knowledge bases. | The local NBA corpus is not an AWS Knowledge Base in this account. |
| Glue/Athena | No Glue databases; Athena has only the enabled `primary` workgroup. | No catalogued RunBundle reporting path is evidenced. |
| Textract | No `DetectDocumentText` or `StartDocumentTextDetection` events in the available CloudTrail lookup. | No OCR runtime proof. |

The local GitHub clone has extensive HOU/DAL source material, but its full local RAG index is not present in the clone. Do not claim that the basketball corpus is already searchable through Bedrock Knowledge Bases.

## Requirement Fit

The competition brief calls for four outcomes: ingest video and deep game data; identify key possessions and tactical changes; overlay analysis onto source video; and narrate in sync with video and data. Current AWS evidence is strongest for training and isolated service probes. It does not yet establish the full chain from one input video to validated CV facts, selected events, synchronized overlay/commentary, and a published result.

Current model readiness:

- **Detection:** useful candidates exist, but sample sizes and domain coverage are not enough for broad accuracy claims.
- **Pose:** not met by the 9-class detection model; latest pose job failed before training.
- **Court geometry:** historical comparisons exist; Macau has no ground truth and cannot establish domain-shift accuracy.
- **Ball/foot:** candidate exists, but held-out metrics regress materially from validation.
- **Run-level governance:** no Model Registry, SageMaker Pipeline, immutable model evaluation report, or linked model/data SHA chain was found.

Do not translate these into fixed competition points. The local official-facts note records AWS service diversity as encouraged but not a fixed count-to-points formula; the live scoring rubric remains necessary to quantify service points.

## Recommended Upgrade

Build one evidence-producing product path, then attach services that do useful work on that same run:

1. **Ingest:** versioned S3 raw video plus a manifest containing input SHA-256, source/game, dataset and model versions, config SHA, code SHA, and `run_id`. EventBridge routes to SQS/DLQ; a Lambda validates and deduplicates before starting Step Functions.
2. **Analyze:** Step Functions runs independent branches: SageMaker model execution on the account-permitted GPU path; Transcribe on source audio; Rekognition as generic scene evidence; Textract on selected scoreboard frames; MediaConvert for normalized output after the role/request issues are fixed. Each branch writes a status and artifact to that run's staging prefix.
3. **Validate:** ClaimGate checks model version, source/time reference, schema, output hashes, conflicts, and branch status. Failed optional branches become visible degradation; they cannot invent or override basketball facts.
4. **Publish:** write an immutable RunBundle with input/model/service IDs, metrics, output hashes, trace links, and gate result. A conditional S3 pointer publishes only a passing bundle. DynamoDB mirrors run state.
5. **Explain:** AgentCore reads approved RunBundle claims plus a curated HOU/DAL Bedrock Knowledge Base; Polly speaks only approved commentary. CloudFront serves the synchronized video, overlays, transcript, and evidence drawer.
6. **Measure:** Glue catalogs RunBundles; Athena produces model/service coverage reports; SNS notifies a subscribed operator after success/degradation. CloudWatch/X-Ray carries `run_id` through every stage.

### Upgrade Order

1. Fix data governance first: game/video-level split manifests, enough holdout samples, per-class counts, licenses, and SHA-256 for files, datasets, models, and evaluation reports.
2. Repair RF-DETR dependency isolation and the pose script import bug. Rerun only after CPU-only import checks pass; do not promote on `Completed` status alone.
3. Connect the existing successful Transcribe, Polly, and Rekognition proofs to the same run contract; persist their outputs and hashes.
4. Add the SQS/DLQ boundary, wire Step Functions branches, and fix MediaConvert permissions/schema. Add Textract, Glue/Athena, SNS, and Bedrock Knowledge Bases only when each produces a visible run artifact or report.
5. Re-evaluate service additions against the official scoring checks. Prioritize services with distinct inputs/outputs in the four competition outcomes; avoid disconnected proof jobs used only to inflate service count.

## Limits

- The official service-specific point allocation was not available in the source material inspected. Exact score prediction remains unavailable.
- All service and training findings above are read-only account observations at the timestamps stated; the single pending RF-DETR job may change after this check.
- No training, deployment, service call, or AWS resource mutation was performed during this audit.
