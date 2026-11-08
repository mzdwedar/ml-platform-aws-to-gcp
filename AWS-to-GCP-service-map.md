# AWS → GCP Service Map: Original ML Platform

Source: `ml-platform-graph.svg`, the original AWS design. This document is a **like-for-like migration**: every box in the diagram gets its closest GCP service, with what overlaps and what doesn't. It deliberately makes no redesign choices; those are in [CAPABILITY-MAP.md](CAPABILITY-MAP.md).

**Checked against official Google Cloud and AWS documentation on 2026-11-08.** Sources are listed in §6.

> **Naming:** on 2026-04-22 Google renamed **Vertex AI → Gemini Enterprise Agent Platform** ("Agent Platform"). Vertex AI Pipelines → Agent Platform Pipelines, Vertex AI Endpoints → Agent Platform Endpoints, Vertex AI Training → Agent Platform Managed/Serverless Training, and so on. The API endpoint is still `aiplatform.googleapis.com`. Below, "Agent Platform (Vertex AI)" means the same product. On the AWS side, SageMaker's docs now call it **Amazon SageMaker AI**.

**Fit:** ● same tool or near-identical · ◐ strong overlap, some gaps · ○ partial: needs extra build or a different pattern

## 1. Summary

| # | Diagram component | AWS / tool today | GCP service | Fit |
|---|---|---|---|---|
| 1 | Data pipeline: extract, merge, ingest | Databricks (Spark, Delta tables) on S3 | **Databricks on Google Cloud**, or **Dataproc / Serverless for Apache Spark** + **BigLake** + **BigQuery** | ● / ◐ |
| 2 | Raw storage, batch output, predictions | Amazon S3 | **Cloud Storage** | ● |
| 3 | Data validation + feature transform | SageMaker Processing steps | **Agent Platform Pipelines** components (custom jobs or Serverless Spark); **Dataflow** for large scale | ◐ |
| 4 | "Same feature pipeline at inference" | Shared code in the pipeline and endpoint | Shared code; optionally **Feature Store on Agent Platform** (BigQuery is the offline store) | ◐ |
| 5 | Orchestration | SageMaker Pipelines | **Agent Platform Pipelines** (Kubeflow Pipelines / TFX) | ◐ |
| 6 | Model training + evaluation | SageMaker Training jobs | **Agent Platform Serverless Training** (+ **Vizier** for tuning, **persistent resources** for warm clusters) | ◐ |
| 7 | Tracker, model registry, data versioning | ClearML self-hosted on EC2 | **A:** ClearML on **Compute Engine/GKE** · **B:** **Experiments + ML Metadata + Model Registry on Agent Platform** | ● / ○ |
| 8 | Source + CI/CD (PR, comment on PR, approval gate, merge) | GitHub + GitHub Actions | Unchanged. Cloud auth becomes **Workload Identity Federation** | ● |
| 9 | Model package | Triton Inference Server | Triton as a **custom container** on **Agent Platform Endpoints**, or on **GKE** | ◐ |
| 10 | Image registry | Amazon ECR | **Artifact Registry** + **Artifact Analysis** | ● |
| 11 | Test on batch, check business metrics | SageMaker Batch Transform from CI | **Agent Platform Batch Inference** + **BigQuery** | ● |
| 12 | Deploy: update batch pipeline + update endpoint | SageMaker endpoint update (deployment guardrails) | **Agent Platform Endpoints**: traffic split, rolling deployment (Preview) | ○ |
| 13 | Batch inference → S3 | SageMaker Batch Transform | **Agent Platform Batch Inference** → GCS or BigQuery | ● |
| 14 | Online endpoint ↔ backend service | SageMaker real-time endpoint | **Agent Platform Online Inference** endpoint (dedicated public or Private Service Connect) | ◐ |
| 15 | Prediction ingestion (event-based job) | Endpoint data capture → S3 → event job | **Request-response logging → BigQuery** (sampled); **Eventarc + Cloud Run functions** for custom handling | ◐ |
| 16 | Backtesting: predictions + labels → summary stats | Custom job | **BigQuery** scheduled queries or a pipeline step | ● |
| 17 | System monitoring / errors | Sentry | Sentry, unchanged, or **Error Reporting** | ● / ○ |
| 18 | Logging, metrics, alarms | Amazon CloudWatch | **Cloud Logging + Cloud Monitoring** | ◐ |
| 19 | Alerts, summaries | Slack | Slack, unchanged (Cloud Monitoring has a native Slack channel) | ● |

**Main takeaway:** about half the diagram is **cloud-agnostic and moves unchanged**: Databricks, ClearML, GitHub Actions, Triton, Sentry and Slack. The real migration is **SageMaker (pipelines, training, endpoints, batch), S3, ECR, CloudWatch and IAM**. The real functional gaps are in **deployment safety on endpoints** and **multi-model Triton serving**.

## 2. Component details

### 1. Data pipeline: Databricks + Spark on S3
- **GCP:** Databricks runs on Google Cloud, with compute in your own GCP account and Unity Catalog available, so the lowest-risk move is the **same Databricks platform on GCP**. The GCP-native alternative is **Dataproc** or **Serverless for Apache Spark** with Delta or Iceberg tables in Cloud Storage, catalogued in **BigLake metastore**, and queryable from **BigQuery**.
- **Overlaps:** Spark jobs, open table formats (Delta, Iceberg), notebooks, scheduled jobs.
- **Delta tables are the data-versioning layer.** Delta time travel (`VERSION AS OF` / `TIMESTAMP AS OF`) is a Delta Lake feature, not an AWS one. It works the same on **Databricks on Google Cloud** and in **open-source Delta on Dataproc / Serverless Spark**.
  - **Retention (same on both clouds):** history is kept 30 days by default (`delta.logRetentionDuration`), and `VACUUM` removes unreferenced files after 7 days (`delta.deletedFileRetentionDuration`). On Databricks Runtime 18.0+, time travel past that window is blocked. Versions used for training need longer retention or a pinned copy.
  - **BigQuery is not a substitute for versioning:** BigLake Delta tables read **only the current version**, and BigQuery's own time travel is **2–7 days** (plus a 7-day fail-safe that only Customer Care can restore from).
- **Doesn't carry over (GCP-native path only):** Databricks-specific features such as Delta Live Tables and Workflows.

### 2. Amazon S3 → Cloud Storage
- **Overlaps:** object storage, versioning, lifecycle rules, storage classes (**Standard / Nearline (30 d) / Coldline (90 d) / Archive (365 d)** ≈ S3 Standard / IA / Glacier tiers), object-change events (**Pub/Sub notifications**: finalize, delete, archive, metadata update), customer-managed keys.
- **GCP gains:** **every class, including Archive, reads within milliseconds**, so there are no Glacier-style restore jobs, though Archive has higher access costs.

### 3–4. Data validation + feature transform (feature pipeline)
- **GCP:** pipeline components running custom containers, Serverless Spark, or Dataflow. Validation code (Great Expectations, TFDV, pandera) moves as-is.
- **Overlaps:** containerised processing steps; the same code reused at inference; **data and model bias metrics** exist in **Model Evaluation on Agent Platform** (the counterpart of SageMaker Clarify's bias reports).
- **Doesn't carry over:** SageMaker's prebuilt Processing containers and Data Wrangler.
- **Feature Store on Agent Platform:** BigQuery tables *are* the offline store, so no separate store needs provisioning. Online serving: **use Bigtable online serving**. *Optimized online serving is deprecated, with no new features from 2026-05-17 and sunset on 2027-02-17.* Bigtable online serving doesn't serve embeddings.

### 5. SageMaker Pipelines → Agent Platform Pipelines
- **Overlaps:** a DAG of containerised steps, parameters, **execution caching** (with task-level and job-level opt-out), lineage of every parameter and artifact in **ML Metadata**, and runs from the SDK or CI. KFP and TFX pipelines are both supported.
- **Doesn't carry over:** SageMaker's step types (Tuning, Clarify, RegisterModel, Lambda, EMR, Callback) and the `@step` decorator must be rewritten as KFP components, where Google Cloud Pipeline Components cover many of them. Event triggers move from EventBridge to Cloud Scheduler or Eventarc.
- **GCP gains:** the KFP spec is open, so pipelines are portable beyond GCP.

### 6. SageMaker Training → Agent Platform Serverless Training
- **Overlaps:** bring-your-own and prebuilt containers, GPUs, distributed training, **hyperparameter tuning** (a built-in feature backed by **Vizier** ≈ SageMaker Automatic Model Tuning), **Spot VMs** (≈ managed spot training).
- **Spot behaviour:** when a Spot VM is preempted, the job fails with `STOCKOUT` and is **restarted up to six times**; you aren't billed while preempted. **Checkpointing and resuming are your code's job**, the same as SageMaker managed spot, which also needs your checkpoints. Spot VMs can't be submitted from the console and don't support TPU Pods.
- **Warm pools:** **persistent resources** keep a training cluster running between jobs, cutting startup time and caching data and images (≈ SageMaker managed warm pools). You pay the whole time it runs.
- **Doesn't carry over:** SageMaker built-in algorithms.
- **Evaluation:** custom code on both clouds, run as a pipeline step.

### 7. ClearML on EC2 (tracker, model registry, data versioning)
- **Option A, lift and shift:** run ClearML Server on **Compute Engine** or **GKE**, with storage on GCS. Same features and UI; only the infrastructure changes.
- **Option B, GCP-managed:** **Experiments** (parameters, summary and time-series metrics, artifacts; time-series stored in managed **TensorBoard**), **ML Metadata** (artifact lineage), **Model Registry** (versions, **aliases**: a `default` alias is always required on one version).
  - **Overlaps:** experiment tracking and comparison, model versioning with aliases, lineage from pipeline runs.
  - **Doesn't carry over:** **ClearML Data's** dataset metadata, analysis and visualisation. The versions themselves still come from **Delta tables** (§1), so log the Delta table version with each run in Experiments or ML Metadata. ClearML Agent queues and ClearML's richer UI are also lost.

### 8. GitHub + GitHub Actions
- **Unchanged.** Only the cloud credentials change: the AWS OIDC role becomes **Workload Identity Federation**. The workflow's GitHub OIDC token is exchanged for short-lived Google Cloud credentials (`google-github-actions/auth`, `id-token: write`), so no service-account keys are needed. Approval gates stay as GitHub Environments.

### 9. Triton model package
- **GCP:** Agent Platform supports Triton **only as a custom container**: you pull NVIDIA's image from NGC and push it to Artifact Registry. AWS, by contrast, ships **prebuilt SageMaker Triton containers**.
- **Constraints on Agent Platform:** requests must use **`rawPredict`** (Triton's format isn't compatible with the standard predict format); the documented GPU types are A100, P100, P4, T4 and V100, and **P100 support ends 2026-09-15**.
- **Doesn't carry over:** **SageMaker multi-model endpoints on GPU**, which SageMaker supports *only through Triton*. Agent Platform's **model co-hosting doesn't support custom containers**, so Triton can't use it. Running several models inside one Triton server is possible in Triton itself, but Google's docs only show single-model deployment, so **test it, or serve Triton on GKE**.

### 10. Amazon ECR → Artifact Registry
- **Overlaps:** Docker/OCI repositories; **automatic vulnerability scanning on push** (Artifact Analysis, continuously updated); cleanup policies with dry run (≈ ECR lifecycle policies); immutable tags; pull-through caching via **remote repositories** (Docker Hub, PyPI, Maven Central, npm); IAM-scoped access.
- **GCP gains:** one registry for Docker, Python, npm, Maven, Go, Apt/Yum and more (AWS needs CodeArtifact for packages). **Binary Authorization** blocks unsigned images at deploy time on **GKE and Cloud Run**, *but not on Agent Platform Endpoints*.

### 11 & 13. Batch testing / batch inference → Agent Platform Batch Inference
- **Overlaps:** managed batch scoring from object storage with custom containers (≈ SageMaker Batch Transform).
- **GCP gains:** **BigQuery in and out**, so business-metric checks are SQL over the outputs.
- **Limits:** BigQuery input requires BigQuery output, and **CMEK isn't supported with BigQuery input/output**. Input and output must be in the model's region.

### 12. Deploy: update endpoint → Agent Platform Endpoints
- **Overlaps:** several models behind one endpoint with **traffic splitting**, which you adjust by hand. **Rolling deployment** (*Preview*) swaps in a new model version on the same compute, with surge and unavailable-replica controls, and reverts automatically **only if the new replicas never become ready**.
- **Doesn't carry over (the biggest gap):**
  - **SageMaker deployment guardrails:** blue/green with all-at-once, **canary** or **linear** traffic shifting, a baking period, and **automatic rollback on CloudWatch alarms**. Agent Platform's docs describe no alarm-based rollback or automatic canary progression. You'd script it: a Cloud Monitoring alert triggers a Cloud Run function, which resets the traffic split.
  - **SageMaker shadow tests:** a managed copy of live traffic sent to a shadow variant, whose responses are never returned. Google's docs describe no managed equivalent; build it with a mirroring proxy, Pub/Sub fan-out or offline replay.

### 14. Online endpoint ↔ backend service
- **GCP endpoint types:** **dedicated public** (recommended; 10 MB payload, timeouts up to 1 h, isolated traffic), **Private Service Connect** (recommended for private access ≈ AWS PrivateLink), shared public (1.5 MB payload, 60 s timeout, 30k QPM), and private via VPC peering.
- **Serverless or scale-to-zero:** **Scale To Zero** is in *Preview* (`minReplicaCount: 0`). Requests that arrive while the model is at zero get a **429 and are dropped, not queued**, so clients must retry. It only works with a single model per endpoint, and models left at zero for 30 days are undeployed. That's weaker than SageMaker Serverless or Async Inference; **Cloud Run with GPUs (L4, RTX PRO 6000)** scales to zero for container serving.
- **Backend service:** wherever it runs today (not in the diagram), it maps to **Cloud Run** or **GKE**.

### 15. Prediction ingestion (event-based job)
- **Overlaps:** **request-response logging** writes a **sample** (configurable up to 100%) of requests and responses straight to a **BigQuery** table (≈ SageMaker data capture to S3), so the separate S3 event job largely disappears.
- **Limits:** configured at the endpoint level through the **REST API only** (not the console or gcloud); on dedicated and PSC endpoints it supports **TensorFlow, PyTorch, sklearn and XGBoost models only**, with payloads under 10 MB. **A Triton custom container may not be covered, so test it.** It can't be combined with Model Monitoring v1 on the same endpoint.
- **Custom handling:** **Eventarc + Cloud Run functions** (≈ EventBridge + Lambda).
- **Related:** **Model Monitoring on Agent Platform**: v1 (GA) covers training-serving skew and drift on endpoints; v2 (*Preview*) adds output drift and feature attribution, and also works for models served outside Agent Platform (≈ SageMaker Model Monitor).

### 16. Backtesting: predictions + labels → Slack summary
- Join predictions and labels in **BigQuery**, run it as a scheduled query or pipeline step, and post stats to Slack. The logic is fully portable.

### 17. Sentry
- **Keep Sentry.** It's SaaS and cloud-agnostic, with no migration needed.
- **GCP-native alternative:** **Error Reporting** groups errors by exception type and the top five stack frames, notifies on new error groups (email, Slack, webhooks via Cloud Monitoring), and tracks Open / Resolved / Muted with regression detection. **It has no release tracking or issue-tracker integration** like Sentry's.

### 18. CloudWatch → Cloud Logging + Cloud Monitoring
- **Overlaps:** log collection, log-based metrics, metrics, dashboards, alerting policies (≈ CloudWatch Alarms), **uptime checks and synthetic monitors** (≈ CloudWatch Synthetics; synthetic monitors run as Node.js Cloud Run functions), log querying with **Observability Analytics** (SQL with pipe syntax, with optional linked BigQuery datasets; ≈ CloudWatch Logs Insights).
- **Doesn't carry over:** I found no direct GCP equivalent of **CloudWatch Contributor Insights** (top-N contributors ranked in real time); approximate it with log-based metrics or SQL. Dashboards and alarms must be rebuilt.
- **GCP gains:** a **native Slack notification channel** (the Slack workspace owner or admin sets it up). AWS routes alarms through **SNS + Amazon Q Developer in chat applications** (formerly AWS Chatbot).

### 19. Slack
- **Unchanged.**

## 3. Implicit services (not drawn, but needed for the migration)

| AWS | GCP | Note |
|---|---|---|
| IAM roles / instance profiles | IAM + **service accounts** | Roles attach to service accounts |
| GitHub OIDC → IAM role | **Workload Identity Federation** | No keys in CI on either cloud |
| VPC (regional), PrivateLink | **VPC (global**, regional subnets), Private Service Connect | One GCP VPC spans regions |
| KMS | **Cloud KMS** | Note the CMEK gap for BigQuery batch inference (§11) |
| Secrets Manager | **Secret Manager** | |
| CloudTrail | **Cloud Audit Logs** | |
| EventBridge / Lambda | **Eventarc / Pub/Sub / Cloud Run functions** | |
| AWS Organizations, SCPs | **Resource Manager** (org, folders, projects), **Organization Policies** | GCP projects ≈ AWS accounts |

## 4. Where the gaps are, ranked by migration effort

1. **Endpoint deployment safety:** canary and linear shifting, alarm-based rollback and shadow tests must all be built yourself (traffic split + Cloud Monitoring + automation). Rolling deployment is Preview and only reverts on readiness failure.
2. **Triton multi-model on GPU:** no prebuilt Triton container on Agent Platform, co-hosting excludes custom containers, and calls must use `rawPredict`. Serving on GKE may be simpler.
3. **SageMaker Pipelines → KFP rewrite:** mechanical, but it touches every step.
4. **Prediction capture for a Triton endpoint:** request-response logging may not cover custom containers on dedicated endpoints; verify it or log from the backend.
5. **ClearML Data:** only a gap if you choose Option B.
6. **CloudWatch dashboards and alarms:** rebuild by hand.

## 5. Preview features this map relies on

Pre-GA features carry limited support. Flag them if production depends on them:
- Scale To Zero for endpoints
- Rolling deployment for endpoints
- Model Monitoring v2

## 6. Sources (fetched 2026-11-08)

**Google Cloud**
- Rename: [Vertex AI name changes](https://docs.cloud.google.com/gemini-enterprise-agent-platform/vertex-ai-name-changes) · [Agent Platform release notes](https://docs.cloud.google.com/gemini-enterprise-agent-platform/release-notes)
- Training: [Spot VMs](https://docs.cloud.google.com/vertex-ai/docs/training/use-spot-vms) · [Persistent resources](https://docs.cloud.google.com/vertex-ai/docs/training/persistent-resource-overview) · [Vizier](https://docs.cloud.google.com/vertex-ai/docs/vizier/overview)
- Pipelines: [Introduction](https://docs.cloud.google.com/vertex-ai/docs/pipelines/introduction) · [Execution caching](https://docs.cloud.google.com/vertex-ai/docs/pipelines/configure-caching)
- Registry and tracking: [Model aliases](https://docs.cloud.google.com/vertex-ai/docs/model-registry/model-alias) · [Experiments](https://docs.cloud.google.com/vertex-ai/docs/experiments/intro-vertex-ai-experiments)
- Features: [Feature Store overview](https://docs.cloud.google.com/vertex-ai/docs/featurestore/latest/overview)
- Serving: [Triton](https://docs.cloud.google.com/vertex-ai/docs/predictions/using-nvidia-triton) · [Model co-hosting](https://docs.cloud.google.com/vertex-ai/docs/predictions/model-co-hosting) · [Deployment and traffic split](https://docs.cloud.google.com/vertex-ai/docs/general/deployment) · [Rolling deployment](https://docs.cloud.google.com/vertex-ai/docs/predictions/rolling-deployment) · [Autoscaling / Scale To Zero](https://docs.cloud.google.com/vertex-ai/docs/predictions/autoscaling) · [Endpoint types](https://docs.cloud.google.com/vertex-ai/docs/predictions/choose-endpoint-type) · [Online inference logging](https://docs.cloud.google.com/vertex-ai/docs/predictions/online-prediction-logging) · [Batch inference](https://docs.cloud.google.com/vertex-ai/docs/predictions/get-batch-predictions) · [Cloud Run GPUs](https://docs.cloud.google.com/run/docs/configuring/services/gpu)
- Monitoring and evaluation: [Model Monitoring](https://docs.cloud.google.com/vertex-ai/docs/model-monitoring/overview) · [Data bias metrics](https://docs.cloud.google.com/vertex-ai/docs/evaluation/data-bias-metrics) · [Model bias metrics](https://docs.cloud.google.com/vertex-ai/docs/evaluation/model-bias-metrics)
- Data: [BigQuery time travel](https://docs.cloud.google.com/bigquery/docs/time-travel) · [Spark with Delta tables](https://docs.cloud.google.com/lakehouse/docs/use-spark-delta-tables) · [BigLake Delta tables](https://docs.cloud.google.com/bigquery/docs/create-delta-lake-table) · [Delta history on Databricks GCP](https://docs.databricks.com/gcp/en/delta/history) · [Delta Lake time travel (OSS)](https://docs.delta.io/latest/delta-batch.html) · [Dataproc Iceberg + BigLake metastore](https://docs.cloud.google.com/dataproc/docs/guides/iceberg-metadata-biglake-metastore) · [Storage classes](https://docs.cloud.google.com/storage/docs/storage-classes) · [GCS Pub/Sub notifications](https://docs.cloud.google.com/storage/docs/pubsub-notifications)
- Registry and supply chain: [Supported formats](https://docs.cloud.google.com/artifact-registry/docs/supported-formats) · [Remote repositories](https://docs.cloud.google.com/artifact-registry/docs/repositories/remote-overview) · [Cleanup policies](https://docs.cloud.google.com/artifact-registry/docs/repositories/cleanup-policy) · [Artifact Analysis](https://docs.cloud.google.com/artifact-analysis/docs/artifact-analysis) · [Binary Authorization](https://docs.cloud.google.com/binary-authorization/docs/overview)
- Operations: [Notification channels](https://docs.cloud.google.com/monitoring/support/notification-options) · [Uptime checks and synthetic monitors](https://docs.cloud.google.com/monitoring/uptime-checks/introduction) · [Observability Analytics](https://docs.cloud.google.com/logging/docs/log-analytics) · [Error Reporting grouping](https://docs.cloud.google.com/error-reporting/docs/grouping-errors) · [Error Reporting notifications](https://docs.cloud.google.com/error-reporting/docs/notifications)
- Identity and network: [Workload Identity Federation for pipelines](https://docs.cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines) · [VPC overview](https://docs.cloud.google.com/vpc/docs/vpc)
- Databricks: [Databricks on Google Cloud architecture](https://docs.databricks.com/gcp/en/getting-started/overview)

**AWS**
- [SageMaker deployment guardrails](https://docs.aws.amazon.com/sagemaker/latest/dg/deployment-guardrails.html) · [Shadow tests](https://docs.aws.amazon.com/sagemaker/latest/dg/shadow-tests.html) · [Triton on SageMaker](https://docs.aws.amazon.com/sagemaker/latest/dg/triton.html) · [Multi-model endpoints](https://docs.aws.amazon.com/sagemaker/latest/dg/multi-model-endpoints.html) · [Amazon Q Developer in chat applications rename](https://docs.aws.amazon.com/chatbot/latest/adminguide/service-rename.html) · [CloudWatch Contributor Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContributorInsights.html)

**Not fetched individually:** these are well-established product mappings — Dataflow, Cloud Composer, Cloud KMS, Secret Manager, Cloud Audit Logs, Eventarc, Resource Manager / Organization Policies, and the AWS side's SageMaker Processing, Async/Serverless Inference and built-in algorithms.
