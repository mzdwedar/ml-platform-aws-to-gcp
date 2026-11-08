# ML Platform: AWS → GCP Translation Brief

**Author:** Mohamed Zeyad · **Date:** 2026-11-08 · **Details and sources:** [AWS-to-GCP-service-map.md](AWS-to-GCP-service-map.md)

## Approach

This is a like-for-like migration of my AWS ML platform (`ml-platform-graph.svg`), grouped by platform capability. Every claim was checked against the official Google Cloud and AWS docs on 2026-11-08.

**Naming:** Google renamed Vertex AI to **Gemini Enterprise Agent Platform** ("Agent Platform") on 2026-04-22. The API is still `aiplatform.googleapis.com`.

**Fit:** ● near-identical · ◐ strong overlap, some gaps · ○ needs extra build

## 1. Training and inference pipelines

| AWS | GCP | Fit | Overlap / gap |
|---|---|---|---|
| Databricks + Spark on S3 (Delta tables) | **Databricks on Google Cloud** (Delta on GCS), or Dataproc / Serverless Spark + BigLake | ● / ◐ | Databricks runs on GCP unchanged. Open-source Delta runs on Dataproc; the native path loses Databricks-only features (Delta Live Tables, Workflows) |
| SageMaker Pipelines | **Agent Platform Pipelines** (Kubeflow Pipelines / TFX) | ◐ | Caching and ML Metadata lineage match. Every step must be rewritten as a KFP component |
| SageMaker Processing (validation, feature transform) | Pipeline components, Serverless Spark, Dataflow | ◐ | Same code is reused at inference. No prebuilt Processing containers |
| SageMaker Training | **Agent Platform Serverless Training** + Vizier | ◐ | Spot VMs (auto-restart up to 6×), hyperparameter tuning, warm clusters via persistent resources. No built-in algorithms |
| Batch Transform → S3 | **Agent Platform Batch Inference** → GCS / BigQuery | ● | Reads and writes BigQuery directly, but without customer-managed encryption keys (CMEK) |
| Real-time endpoint + Triton | **Agent Platform Endpoints** (custom Triton container) or GKE | ◐ | No prebuilt Triton container, requests must use `rawPredict`, and **no GPU multi-model hosting** (co-hosting excludes custom containers) |

## 2. Experimentation: tracking, registry, data versioning

| AWS | GCP | Fit | Overlap / gap |
|---|---|---|---|
| ClearML on EC2 (all three) | **A:** ClearML on Compute Engine / GKE | ● | Identical; only the hosting changes |
| ClearML experiment tracker | **B:** Experiments on Agent Platform + managed TensorBoard | ◐ | Parameters, metrics and artifacts match; ClearML's UI is richer |
| ClearML model registry | **B:** Model Registry on Agent Platform | ◐ | Versions and aliases match, and models deploy straight to endpoints |
| **Delta tables (data versions through time travel)** | **Delta on Databricks on GCP**, or **open-source Delta on Dataproc** | ● / ◐ | `VERSION AS OF` / `TIMESTAMP AS OF` work the same on both. **BigQuery is not a substitute:** it reads only the *current* snapshot of a Delta table, and its own time travel stops at 7 days |
| ClearML Data (dataset metadata, analysis, visualisation) | **B:** ML Metadata lineage + Delta version numbers logged per run | ○ | Delta still provides the versions; what's lost is ClearML's dataset UI and diffs |

## 3. Model rollout (CI/CD)

| AWS | GCP | Fit | Overlap / gap |
|---|---|---|---|
| GitHub Actions: PR, metrics comment, approval gate | **Unchanged**, with Workload Identity Federation for keyless cloud access | ● | Only the authentication step changes |
| ECR | **Artifact Registry** + Artifact Analysis | ● | Scanning on push, cleanup policies, immutable tags, pull-through cache. Binary Authorization covers GKE and Cloud Run, **not Agent Platform Endpoints** |
| Update endpoint (deployment guardrails) | **Endpoint traffic split**; rolling deployment (*Preview*) | ○ | **The biggest gap.** No canary or linear shifting with alarm-based auto-rollback, and no managed shadow tests. You have to script them |
| Update batch pipeline | New model version in the batch pipeline | ● | Same pattern on both clouds |

## 4. Model monitoring

| AWS | GCP | Fit | Overlap / gap |
|---|---|---|---|
| Prediction ingestion (event job → S3) | **Request-response logging → BigQuery** | ◐ | No ingestion job is needed, but logging is sampled, set up only through the REST API, and **may not cover Triton containers** |
| Backtesting → Slack summary | **BigQuery** scheduled queries or a pipeline step | ● | Portable SQL and Python |
| *(not in the AWS design)* | Model Monitoring on Agent Platform | ◐ | Skew and drift (v1, GA). v2 is in Preview |
| Sentry | **Keep Sentry**, or Error Reporting | ● / ○ | Error Reporting groups errors and alerts on new ones, but has no release tracking or issue-tracker links |
| CloudWatch | **Cloud Logging + Cloud Monitoring** | ◐ | Alerts, synthetic monitors and SQL over logs match. No Contributor Insights equivalent; dashboards must be rebuilt |
| Slack | **Unchanged** | ● | Cloud Monitoring posts to Slack natively. AWS needs SNS + Amazon Q Developer in chat applications |

## 5. Infrastructure and security

| AWS | GCP | Overlap / gap |
|---|---|---|
| S3 | **Cloud Storage** | Archive storage reads within milliseconds, so there are no restore jobs |
| EC2 | **Compute Engine** / **GKE** | |
| IAM roles, GitHub OIDC | IAM + service accounts, **Workload Identity Federation** | No keys on either cloud |
| VPC, PrivateLink | **Global VPC** (regional subnets), **Private Service Connect** | |
| KMS, Secrets Manager, CloudTrail | Cloud KMS, Secret Manager, Cloud Audit Logs | |
| Organizations, SCPs | Resource Manager (projects ≈ accounts), Organization Policies | |
| EventBridge, Lambda | Eventarc, Pub/Sub, Cloud Run functions | |

## Summary

- **Half the stack moves unchanged:** Databricks, ClearML, GitHub Actions, Triton, Sentry and Slack. The real migration is SageMaker, S3, ECR, CloudWatch and IAM.
- **The gaps are concentrated in model rollout and serving:**
  1. Deployment guardrails and shadow tests have to be built on GCP.
  2. Triton GPU multi-model hosting has no managed equivalent; serve on GKE or test carefully.
  3. Prediction capture for custom containers needs verifying.
- **Data versioning carries over because it's Delta, not a cloud service.** Same caveat on both clouds: by default Delta keeps 30 days of history and `VACUUM` removes files after 7 days. Any version a model was trained on needs longer retention or a pinned snapshot.
- **Next step:** [CAPABILITY-MAP.md](CAPABILITY-MAP.md) is the GCP-native version of the platform, demonstrated on a page-topic classification workload. It closes the rollout gap with a shadow → canary → pointer-flip release process.
- **Preview features to flag:** endpoint rolling deployment, Scale To Zero, Model Monitoring v2.

## How I produced this

The work was LLM-assisted, and I verified every claim in the official docs. Verification caught real errors in the LLM's first draft:
- It used the pre-April 2026 "Vertex AI" names.
- It said there was no counterpart to SageMaker warm pools (persistent resources are one).
- It missed that Agent Platform co-hosting excludes custom Triton containers.
