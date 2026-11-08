# Capability Map: ML Platform on GCP

Source architecture: `ml-platform-graph.svg` (AWS / Databricks / SageMaker / ClearML).
Status: **design proposal, 2026-11-08.** Every GCP and GitHub capability claimed here was checked against the official documentation on 2026-11-08 (see [Sources](#sources)). Google renamed Vertex AI to **Gemini Enterprise Agent Platform** ("Agent Platform") in April 2026.

**Context:** the GCP-native version of the platform, carrying over the capabilities of my Revic AWS platform (see [BRIEF-aws-to-gcp.md](BRIEF-aws-to-gcp.md)) and demonstrated on one concrete workload: classifying web pages by topic.

**Goal:** an MLOps platform that delivers three things well: **traceable training**, **safe model rollout** and **model monitoring**. The model is deliberately simple (D26) so the effort goes into the platform, not the modelling.

**What the model does:** it assigns each web page one **IAB tier-1 topic** with a confidence score. A frozen, pinned multilingual encoder turns page text into an embedding, and a trained logistic-regression head turns the embedding into a topic. Downstream systems read the result from Bigtable.

**Out of scope (D20):** crawling, text extraction, any streaming, labelling (all upstream), ad selection and images.
## Model assumptions (D26) — the simplifications

| # | Assumption | What it removes |
|---|---|---|
| M1 | **One task:** single-label IAB Content Taxonomy 3.x **tier-1** topic (36 tier-1 categories in v3.0, 37 in v3.1, + `other`), argmax + confidence | Multi-label heads, per-class thresholds and calibration, brand safety, sentiment, keywords |
| M2 | **Frozen encoder:** a pretrained multilingual sentence encoder (multilingual-E5-small class), exported once to ONNX, mirrored and pinned by sha256 for all of v1. The encoder version covers the weights, the tokenizer **and the preprocessing config**; a golden-embedding check blocks unversioned changes (D33) | Fine-tuning, GPU training, HPO, encoder drift between releases |
| M3 | **Only a linear head is trained:** scikit-learn logistic regression on stored embeddings, with a small cross-validated grid over `C`. Seeded, in a pinned image; every run is **traceable**, not re-run (D41) | Vizier, GPU training, scheduled re-runs to prove reproducibility |
| M4 | **Embeddings computed once per page version and stored** in BigQuery `embeddings.pages`, keyed by `url_hash`, `content_hash` and `encoder_version`. Training reads the same embeddings that serving produced | Training/serving skew (parity by construction); expensive re-scoring |
| M5 | **A release changes the head only.** Rescoring the whole corpus is a matrix multiply over stored embeddings (CPU, hours) | Partial backfills; pages left without context after a flip |
| M6 | **Input = title + first 256 tokens** of body text; no chunking | Long-document handling |
| M7 | **Labels arrive upstream** as immutable, versioned BigQuery tables from a labelling team (U10, U11); the platform only requests audit samples | Label Studio, LLM pre-labelling, an annotation team inside the platform |
| M8 | **CPU-only in steady state:** the incremental scorer runs the ONNX encoder + head in-process. GPUs (L4) are used only for the one-off bootstrap that embeds the existing corpus | Triton, always-on GPUs, GPU quota risk for shadow and backfill |
| M9 | **Encoder upgrades are out of v1.** Changing the encoder means re-running the bootstrap and is a separate, approved project | Cross-version embedding mixing |

## Decisions

Decision IDs are stable and cross-referenced; gaps in the numbering are decisions that were retired.

| # | Decision | Notes |
|---|---|---|
| D1 | **BigQuery-native** for ML data (embeddings, training sets, eval, history, monitoring). Serving store is Bigtable (D23). | |
| D2 | **MLflow** (tracker + Model Registry) on GKE, Cloud SQL + GCS, behind IAP. Single source of truth for model versions. | |
| D3 | **Incremental + batch inference.** An incremental scorer polls BigQuery every 5 min, embeds and scores new/changed pages; a batch job rescores the full corpus from stored embeddings when a head ships. | |
| D4 | **BigQuery is the only input interface.** All upstream data (pages, labels) is read from BigQuery. No Kafka and no Pub/Sub in the platform. | A sub-minute freshness need would bring back a stream (v2). |
| D6 | **Inference on GKE Autopilot** (long-running CPU scorer). Agent Platform (formerly Vertex AI) for Pipelines, CPU training jobs, batch rescoring, and the one-off GPU bootstrap. | M8 |
| D7 | **Monitoring: Evidently AI → Prometheus → Grafana → Slack** (Managed Service for Prometheus with managed rule evaluation, Pushgateway for batch jobs, managed Alertmanager with a Slack receiver). Cloud Logging, Error Reporting. Slack only. | Managed Alertmanager is **Preview**. The Pushgateway needs small config changes under managed collection and can inflate time-series counts, so keep batch-job labels low-cardinality. |
| D9 | **EU-only:** `europe-west4`, org policy `gcp.resourceLocations = in:eu-locations`. | The constraint covers only the products that support it, ignores global resources, and isn't a data-storage commitment. Residency also relies on regional resources, regional CMEK key rings and VPC-SC. Every service used here is available in `europe-west4`, including L4 GPUs for Agent Platform training and batch inference. |
| D10 | **SOC 2 + GDPR cross-cutting.** Inputs are public page content with user identifiers stripped upstream; no user personal data. | |
| D11 | **Data versioning = immutable tables** (no DVC/lakeFS): label sets mirrored write-once into platform-owned `labels_mirror` on first use; `embeddings.pages` (append-only, per-row `row_checksum`); `training_sets.topic_<run_id>` (per-row `row_hash`, ordered by `(url_hash, content_hash)`). Write-once is **IAM-enforced** for `labels_mirror` and training sets (no service account keeps write access after publish). `embeddings.pages` needs ongoing appends, so it relies on detection: a DML alert plus checksums verified at read. Every run verifies content hashes. Exception: legal takedowns, which record erased `row_hash` values so hashes stay verifiable (D40). | |
| D12 | **`stg` and `prd`** (+ `shared`). stg holds a frozen ~100k public-page sample with embeddings and matching label subsets, plus synthetic pages for load tests. Full-corpus training only in prd from merged code. | |
| D13 | **Batch scoring = Agent Platform Pipelines custom job loading the head from MLflow.** | |
| D14 | **Public GitHub repo on Free**, environments + required reviewers + prevent self-review. | |
| D15 | **Greenfield org**, Step 0 bootstrap; prefix `cnxtad`. | |
| D17 | **Scale (defaults):** 10M new/changed pages/day (peaks ≤ 1k/s), 1B pages stored, 50k Bigtable reads/s. | |
| D19 | **Rollout = dual-head scorer + canary + pointer flip.** Stage 1 **shadow** (zero risk, full load; ≥ 1 h and until the rescore finishes): the scorer also scores every page with the challenger head, and the **full-corpus rescore** runs at the same time. It tests stability, output shape and schema, SLOs and Bigtable latency under peak load, and **offline vs live consistency** (agreement, topic shares and confidence vs what the replay predicted). Stage 2 **canary** tests **downstream integration**, ramped 5% → 25% of pages with each step ≥ 24 h: guardrails (decode errors, latency, error budgets) plus statistical canary-vs-control analysis with computed minimum sample sizes. Then **flip** the active-release pointer. **No A/B phase in v1** (deferred to phase 3). The scorer keeps scoring the previous champion head for 7 days, so rollback is a clean flip. | M5 |
| D20 | **Scope = ML platform only.** | |
| D21 | **v1 workload = page topic classification, text only.** | M1 |
| D23 | **Output = Bigtable `page_context`** with release-prefixed qualifiers and an active-release pointer; read via the `mlp-context` Python client or any implementation passing its conformance fixtures. | |
| D24 | **Freshness SLO:** p95 ≤ 15 min from a row's `ingested_at` in upstream BigQuery to its topic in Bigtable. | Bounded by the 5-min BigQuery poll (D4). |
| D25 | **Roadmap:** phase 1 topic classification (this); phase 2 campaign matching (stored embeddings make this easy); phase 3 pCTR/pViewability ranking. | |
| D26 | **Simplified model (M1–M9) so the work focuses on MLOps.** | |
| D27 | **Retraining is label-driven, not drift-driven.** Triggers: merge to main and a new label-set version; no scheduled retrain (D36). Drift raises an alert and a **label request** for the drifted segment; the retrain follows when those labels land. | Retraining on unchanged code and labels adds nothing (D36). |
| D28 | **One source of truth per release:** `deploy/release.yaml`. A reconciliation check alerts when the MLflow `champion` alias, the Bigtable pointer or the heads loaded in the scorer disagree with it. | |
| D29 | **Triggers run in GitHub Actions; no message bus.** Merge and the daily label-set check are GitHub workflows (`on: push` / `on: schedule`) that start the Agent Platform pipeline, wait for it, and open the release or label-set PR themselves. Cloud Scheduler runs the monitoring and `lineage-verify` Cloud Run jobs. | Replaces Pub/Sub and `pipeline-launcher`. GitHub disables schedules in a public repo after 60 days of inactivity, so `monitoring` alerts on missed scheduled runs. Environments with required reviewers and **Prevent self-review** are available on GitHub Free for public repos. |
| D30 | **Drift and quality are measured against production, not the labelled training set.** Each release freezes a production reference (200k pages scored in the 28 days before its flip) and an audit baseline (the new champion on the previous 8 weeks of audit labels). The audit-quality alert uses 4 pooled weeks and a paired difference test. | |
| D31 | **Gates are paired statistical tests.** A domain-cluster bootstrap on the shared test set; a check fails only on a drop that is both larger than its tolerance and significant (Holm-corrected for slices and classes). Candidates are also compared with a fixed `anchor` head to stop gradual decline. | |
| D33 | **The encoder version includes preprocessing.** `preprocess.yaml` and the tokenizer are part of `encoder_version`; stored golden embeddings are checked in CI and at scorer readiness. Embeddings record their execution provider; CPU vs GPU parity uses a tolerance. | |
| D34 | **Rollbacks are reconciled, and the canary and the first 36 h after a flip roll back automatically.** A rollback writes `meta/rollback_pending` before flipping; reconciliation treats it as expected until the expedited revert PR merges. The canary and the post-flip watch roll back automatically on any breached check. | Refines D19, D28 |
| D35 | **Leakage is checked by the platform.** `check_inputs` fails on shared `content_hash`, `url_hash` or domain between train and test; train rows that are near-duplicates of test pages (cosine ≥ 0.98) are dropped; weekly audit pages never enter training. | |
| D36 | **No scheduled retrain; no-op guard.** Retraining on unchanged code and labels adds no information, so there's no monthly retrain; a run whose training set, config and test predictions equal the champion's registers nothing. | Refined by D41 |
| D37 | **Production audit labels are part of the gate.** Weekly audit labels are evaluation-only; the gate compares candidate and champion on the last 12 weeks of them (IPW, paired), alongside the test set. | |
| D38 | **Rollout orchestration and canary.** Stages run in the Agent Platform pipeline `topic-rollout` (Cloud Deploy only rolls out scorer manifests). Shadow = stability, SLOs and offline-vs-live consistency (live shifts vs replay-predicted shifts); canary = downstream integration, ramped 5% → 25% of pages by `url_hash` bucket, each step ≥ 24 h, judged by statistical canary-vs-control analysis. One change type per release: head **or** scorer image (code releases canary by scorer shard). No A/B phase in v1. Automatic rollback from the canary through a 36 h post-flip watch. | Refines D19. Cloud Deploy renders kustomize overlays through Skaffold profiles and supports per-target approvals. |
| D39 | **Monitoring watches its own pipeline.** Batch-job staleness, label-request fulfilment, and champion and label-set age are monitored. | |
| D40 | **Data lineage survives takedowns and age.** `training_set_hash` is computed over ordered per-row hashes; the `takedown` job records erased `row_hash` values in `training_sets.erasures`, so verification is remaining ∪ erased. Training sets, their label mirrors and referenced embedding rows are kept for 400 days or while the head version is `champion` or `previous`, whichever is longer. | Refines D11 |
| D41 | **Traceable training, not re-run training.** Every head records its full lineage (code, image, encoder, label sets, training set, seed, `model_sha256`), and its inputs stay immutable and hash-verifiable while it can serve (D11, D40). Training is never re-run to prove reproducibility: no `topic-reproduce`, no quarterly rebuild, no pinned-BLAS determinism. A read-only monthly `lineage-verify` job checks that the lineage of `champion` and `previous` still resolves. The golden-embedding check (D33) stays: it prevents training/serving skew. | Refines D36 |

## Upstream data contract (D20)

| # | Item | Requirement |
|---|---|---|
| U1 | **Location** | BigQuery datasets in `europe-west4`, read across projects through a documented VPC-SC rule. |
| U3 | **Pages table** | BigQuery `pages`, append-only, one row per fetched page version or delete: `url`, `canonical_url`, `url_hash`, `domain`, `publisher_id`, `language_hint`, `title`, `body_text`, `content_hash`, `fetched_at`, **`ingested_at`** (when the row landed; partition column, clustered by `url_hash`), `event` (`upsert` \| `delete`), `delete_reason`, `first_seen_at`, `last_seen_at`. Rows land **≤ 5 min after fetch** (streaming ingestion). ≥ 400 days of history. |
| U4 | **Privacy** | User identifiers stripped from URLs; body text is public content. |
| U5 | **Readiness** | Daily partition marker on `pages`. |
| U6 | **Deletes** | `event = delete`; `delete_reason = legal_takedown` for erase-everywhere. |
| U7 | **Rates** | ~10M rows/day; notice before bulk recrawls. |
| U8 | **Hashing** | `url_hash` canonicalization rules with golden vectors. |
| U9 | **Versioning** | Additive schema changes only; a breaking change means a new table version (`pages_v2`). |
| U10 | **Label sets** | Immutable BigQuery tables `labels.train_v<n>` and `labels.test_v<n>`: `url_hash`, `content_hash`, `topic_t1`, `labeler_agreement`, `taxonomy_version`. Test sets are human-labelled and disjoint by domain from train sets (verified by the platform, D35), and refreshed with a new version at least yearly so the test set doesn't go stale. Each published version is listed in `labels.versions` (version, table names, content hash, published_at). |
| U11 | **Audit labels** | The labelling team labels the samples we write to `label_requests` (weekly audit ~1k pages, plus drift-segment requests) within 5 working days, into `labels.audit_<week>`. Weekly audit pages are evaluation-only: they must not also appear in a train set (D35). |

## Output contract

| Item | Definition |
|---|---|
| `page_context` (Bigtable) | Row key `url_hash`. Per release `r<NN>:`: `topic_t1`, `confidence`, `model_version`, `encoder_version`, `content_hash`, `scored_at`. Row-level `schema_version`, `taxonomy_version`. ~60 bytes per release per row; the last 3 releases are kept. |
| `meta/active_release` | Active release id, plus optional `canary_release` and `canary_buckets` (pages with `url_hash` bucket < `canary_buckets` get the canary release, D38); clients cache 60 s. |
| `domain_context` | Daily per-domain topic distribution; fallback for unseen URLs. |
| `mlp-context` client (Python) | `get(url)` → hash (U8) → active release → topic, falling back to domain context, then `unknown`. Conformance fixtures (golden vectors + read fixtures) for other-language readers. p99 ≤ 10 ms. |

## AWS → GCP translation

| Diagram component | GCP target |
|---|---|
| Raw storage, data pipeline, labelling | **Upstream** (D20, U10–U11) |
| Data validation | `check_inputs` step + scorer schema validation |
| Feature store | `embeddings.pages` (append-only, versioned by encoder) |
| Orchestration | Agent Platform Pipelines (KFP v2), started and awaited by GitHub Actions (events and schedules); Cloud Scheduler for Cloud Run jobs |
| Training / HPO | Agent Platform Serverless Training (CPU); scikit-learn with a CV grid |
| Tracker / registry / data versioning | MLflow lineage tags; immutable label, embedding and training-set tables; monthly read-only `lineage-verify` (D41) |
| Image registry | Artifact Registry + scanning + Binary Authorization |
| Model package | ONNX encoder + scikit-learn head, loaded in the scorer image (no Triton, M8) |
| Online inference | Incremental scorer on GKE (polls BigQuery) → Bigtable |
| Batch inference | Full-corpus rescore from stored embeddings (Agent Platform) |
| Logs / metrics / alerts | Cloud Logging, Managed Service for Prometheus, Grafana, Error Reporting, managed Alertmanager (Preview) → Slack |
| Drift + backtesting | Evidently → Prometheus → Slack; replay on stored embeddings; weekly audit labels |
| CI/CD | GitHub Actions + WIF + environments + Cloud Deploy |

## Modules

| Module id | Responsibility | Depends on |
|---|---|---|
| `foundation` | Org policies, projects, VPC, VPC-SC, CMEK, IAM, WIF, Artifact Registry + Binary Authorization, audit logs, GKE Autopilot, Terraform state | — |
| `experiment-tracking` | MLflow, lineage conventions, aliases, registry for `page-encoder` and `topic-head` | foundation |
| `training-pipeline` | Encoder export + bootstrap embedding, training sets, head training, eval + gate, registration, lineage verification | experiment-tracking, upstream contract |
| `serving` | Dual-head incremental scorer, embeddings store writes, full-corpus rescore, flip, deletes/takedowns, domain fallback, client | experiment-tracking, training-pipeline |
| `cicd` | PR checks, image build/sign, prd training, release PR, shadow (stability + offline vs live) → canary (downstream integration) rollout + flip, rollback | training-pipeline, serving |
| `monitoring` | System + freshness + coverage SLOs, Evidently drift, audit-based quality, label requests, release reconciliation, Slack | serving, training-pipeline |

Build order: `foundation` → `experiment-tracking` → `training-pipeline` ∥ `serving` → `cicd` → `monitoring`

## Cross-cutting compliance requirements

**SOC 2:** group-based IAM with SSO + MFA, no SA keys, WIF for CI, CMEK on BigQuery/GCS/Bigtable/Cloud SQL/Agent Platform/GKE, VPC-SC perimeters, locked audit logs ≥ 400 days, change management through PRs + environment approvals, image scanning + Binary Authorization, Cloud SQL backups with a tested restore, modules write their own evidence to the locked evidence bucket.

**What these controls require in practice (from the docs):**
- **CMEK:** every service above supports customer-managed keys. Agent Platform requires a **regional key ring in the same region** as the resource (`europe-west4`), never multi-region or global.
- **Locked audit logs:** Cloud Logging buckets keep logs for 1–3650 days. Locking a bucket also locks its retention policy, is **irreversible**, and is done with gcloud or the API only, not the console.
- **Binary Authorization** enforces on **GKE and Cloud Run only**. Agent Platform training, pipeline and rescore containers aren't covered, so those images are pinned by digest and scanned by Artifact Analysis instead.
- **VPC-SC side effects:**
  - Agent Platform Pipelines **can't install from public PyPI**: bake dependencies into images, or use an Artifact Registry remote repository. The perimeter blocks internet egress from Agent Platform jobs, which is why the encoder is mirrored (M2).
  - **Cloud Deploy** inside a perimeter needs a **Cloud Build private worker pool** in the same perimeter and a **private GKE cluster**.
  - **Cloud Run** services in a perimeter must use internal-only ingress.
  - **Cloud Scheduler** can target Cloud Run jobs inside a perimeter.
  - Agent Platform request-response logging isn't available under VPC-SC. This design doesn't use it, because the scorer writes its own history.
- **IAP** protects MLflow and Grafana on GKE through a Gateway or Ingress, and users need the IAP-secured Web App User role.

**GDPR and content rights:** EU-only; no user personal data (U4); aggregates only in Slack, metrics and logs; `legal_takedown` removes a page from Bigtable, embeddings, history and training sets within 14 days (`takedown` job); vendors besides Google: GitHub (code) and Slack (aggregates).

## Assumptions

1. Projects `cnxtad-mlp-{stg,prd,shared}` under a GCP Organization.
2. Python 3.11, KFP v2, Terraform ≥ 1.9. No click-ops.
3. GitHub Free, public repo.
4. Single region `europe-west4`.
5. The data platform and labelling team agree to U1–U11; until then stg's sample stands in.

## Shared conventions

**Repo layout**
```
infra/            → Terraform: bootstrap/, modules/<name>/, envs/{shared,stg,prd}/
src/mlp/
  common/         → config, logging, text preprocessing + encoder runtime (shared by scorer, bootstrap, training)
  tracking/  training/  serving/  monitoring/
clients/python/   → mlp-context client + conformance fixtures
pipelines/        → KFP v2 pipeline definitions
services/scorer/  → incremental scorer
deploy/           → kustomize, Cloud Deploy config, release.yaml
tests/            → unit/, integration/, smoke/, load/
policy/           → conftest Rego policies
```

**Python tooling:** Python 3.11, `uv`, **ruff** + **ty** (blocking in CI and lefthook), pytest + pytest-cov, structlog, Pydantic v2, `mise run <task>`; CI runs `mise run check`.

**One encoder runtime:** `mlp.common.encoder` (preprocess → tokenize → ONNX Runtime → embedding) is the only way embeddings are produced, in the scorer, the bootstrap and stg fixtures. Its behaviour is pinned by the stored golden embeddings of the `page-encoder` version (D33).

## Sources

Checked 2026-11-08. The full AWS → GCP service sources are in [AWS-to-GCP-service-map.md](AWS-to-GCP-service-map.md#6-sources-fetched-2026-11-08).

- Residency: [Resource locations constraint](https://docs.cloud.google.com/resource-manager/docs/organization-policy/defining-locations) · [Agent Platform locations](https://docs.cloud.google.com/vertex-ai/docs/general/locations) · [Bigtable locations](https://docs.cloud.google.com/bigtable/docs/locations)
- Security: [CMEK-compatible services](https://docs.cloud.google.com/kms/docs/compatible-services) · [Agent Platform CMEK](https://docs.cloud.google.com/vertex-ai/docs/general/cmek) · [Log buckets and locking](https://docs.cloud.google.com/logging/docs/buckets) · [VPC-SC supported products](https://docs.cloud.google.com/vpc-service-controls/docs/supported-products) · [VPC-SC with Agent Platform](https://docs.cloud.google.com/vertex-ai/docs/general/vpc-service-controls) · [Cloud Deploy execution environments](https://docs.cloud.google.com/deploy/docs/execution-environment) · [Cloud Run with VPC-SC](https://docs.cloud.google.com/run/docs/securing/using-vpc-service-controls) · [Cloud Scheduler with VPC-SC](https://docs.cloud.google.com/scheduler/docs/secure-with-vpcsc) · [Binary Authorization](https://docs.cloud.google.com/binary-authorization/docs/overview) · [IAP for GKE](https://docs.cloud.google.com/iap/docs/enabling-kubernetes-howto)
- Monitoring: [Managed rule evaluation and Alertmanager](https://docs.cloud.google.com/stackdriver/docs/managed-prometheus/rules-managed) · [Managed collection (Pushgateway notes)](https://docs.cloud.google.com/stackdriver/docs/managed-prometheus/setup-managed) · [Alertmanager Slack receiver](https://prometheus.io/docs/alerting/latest/configuration/)
- Delivery: [Cloud Deploy overview](https://docs.cloud.google.com/deploy/docs/overview) · [Cloud Deploy manifests (Skaffold + kustomize)](https://docs.cloud.google.com/deploy/docs/skaffold) · [Cloud Run jobs on a schedule](https://docs.cloud.google.com/run/docs/execute/jobs-on-schedule)
- GitHub: [Scheduled workflows in public repos (60-day rule)](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows) · [Environments, required reviewers, prevent self-review](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment)
- Taxonomy: [IAB Tech Lab Content Taxonomy 3.0 / 3.1 TSVs](https://github.com/InteractiveAdvertisingBureau/Taxonomies/tree/main/Content%20Taxonomies) (tier-1 categories counted from the files)
