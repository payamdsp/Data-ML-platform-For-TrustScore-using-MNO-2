# Project Lotus

| Directory | What it is |
| -- | -- |
| `project_lotus/` | The Python data and ML project. See [its README](project_lotus/README.md) |
| `accounts/` | The Terraform for the AWS infrastructure it runs on. This file covers that |

Lotus runs in `data-sandbox` (`162591926854`) and nowhere else.

## Data governance

**Confidential Data and Restricted Data are prohibited in this account. Synthetic data only.**

| Classification | Definition |
| -- | -- |
| **Restricted** | Personally identifiable information |
| **Confidential** | Partner data |
| Internal | EnStream intellectual property, such as code and documentation |

Restricted fields in the Exchange schema: `first_name`, `last_name`, `street_number`, `street_name`,
`unit_number`, `city`, `province`, `postal_code`, `date_of_birth`, `phone_number`, `imei`.
`source_institution` is Confidential.

Internal data is permitted. Raise a ticket for any classification question.

## The infra and security split

`accounts/data-sandbox/` holds two root modules. Resource type determines which one a definition
belongs in.

| Directory | Define here |
| -- | -- |
| `infra/` | S3, S3 Tables, Glue, EMR, Athena, DynamoDB, Step Functions, EventBridge, log groups, VPC resources |
| `security/` | KMS keys, aliases and key policies; Secrets Manager; **all IAM roles and policies**; S3 bucket policies |

The split is enforced by the CI roles' permissions. A misplaced definition fails the run.

An authorization error on `role/gha-lotus-data-sandbox-infra` means the resource must be defined in
`security/`.

A `couldn't find resource` error on a data source means the `security/` root has not been applied;
define the resource there and merge that pull request first.

## Pipeline

Triggered by changes under `accounts/data-sandbox/**`.

| Event | Runs | AWS credentials |
| -- | -- | -- |
| Pull request | `fmt -check`, `init -backend=false`, `validate` | none |
| Merge to `main` | `plan` then `apply` | the layer's CI role |

Pull requests do not plan. Plan locally against your own SSO session. Apply runs only on `main`.

Do not add a workflow that assumes the CI roles. Role trust pins `job_workflow_ref` to the shared
workflow, so a workflow defined in this repository receives a token STS rejects.

## EMR capacity and job lifecycle

The Terraform-managed test cluster and the daily Step Functions cluster use
the same instance variables, Spark/YARN configuration, storage sizes, and idle policy.
Gold reuses the daily cluster created by Silver; it does not create another cluster.

| Fleet | Nodes | Instance | vCPUs per node | Memory per node | Purchase option |
| -- | -- | -- | -- | -- | -- |
| Primary | 1 | `m5.xlarge` | 4 | 16 GiB | On-Demand |
| Core | 2 | `r5.2xlarge` | 8 | 64 GiB | On-Demand |
| Task | 2 | `r5.2xlarge` | 8 | 64 GiB | On-Demand |

Workers provide **32 vCPUs and 256 GiB** in total. Instance specifications:
[M5](https://docs.aws.amazon.com/ec2/latest/instancetypes/gp.html) and
[R5](https://docs.aws.amazon.com/ec2/latest/instancetypes/mo.html).
The primary has a 50 GiB gp3 volume; each worker has a 200 GiB gp3 volume.

Core nodes provide stable driver capacity. Task nodes also use On-Demand so
Spot interruptions do not change capacity or invalidate the Gold pressure test.
Spot task capacity defaults to zero; it can be enabled after establishing a
stable baseline. There is no managed scaling policy to remove workers during a job.

EMR uses `emr-7.14.0` and step concurrency **1**. In YARN cluster deploy mode,
node labels constrain the Spark driver/ApplicationMaster to the core fleet,
not the primary. Gold's configured driver heap is `24g` with `4g` overhead.
Automated arguments come from the live S3 job config, which Terraform seeds
once; subsequent seed-file edits do not update that live object automatically.

Gold retries a failed stage once, then terminates the daily cluster before
sending the failure notification. Successful Gold publish also terminates it.
Silver submits at most two job attempts, with a fresh run ID for its retry;
terminal failure terminates the shared daily cluster and prevents Gold release.
This also affects any other Silver work queued on that cluster.

Both cluster creation paths default to **3,600 seconds of idle time** before
auto-termination. This is a safety net, not a one-hour limit on active jobs;
see [EMR idle criteria](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-auto-termination-policy.html).
Independent Silver arrivals more than an hour apart may outlive the idle
cluster and require deliberate recovery. Manual EMR steps do not execute
Step Functions retries or cleanup. Existing executions and running clusters
do not automatically adopt these defaults.

See the [pipeline run handbook](project_lotus/docs/docs/transient_emr_and_bronze_arrivals.md)
for artifact checks, live configuration, and manual test submission.

## Raising a ticket

If you run into an issue, raise a ticket on the Project Lotus Linear team and tag InfraSec:
<https://linear.app/enstream-workspace/team/LOTUS/overview>

## Conventions

Each requirement below supports a control elsewhere in the system.

- **Prefix every resource name with `var.project`** (`lotus`). IAM and KMS policies are written
  against `lotus-*`; an unprefixed resource falls outside them silently.
- **Build names from `var.environment`**, never a literal `sandbox`. Promotion is a directory copy
  with that one variable changed.
- **No hardcoded account ids or ARNs.** Resolve by name, or read `/contract/_account/*` from SSM.
- **No account names in file banners or comments.** A promotion copies them and they become false.
- `required_version = "~> 1.15.0"`, provider pinned exactly, `.terraform.lock.hcl` committed.
- Run `terraform fmt -recursive` before committing. CI fails on unformatted files.
- **Never commit** state, `.terraform/`, saved plans, or `.tfvars` with real values.
- Branch and open a PR; never push `main`. Keep infra and security changes in separate PRs.

## Complete source tree

This tree captures the current source snapshot, including all checked-in
examples, fixtures, notebooks, vendored dependencies, and documentation. It
contains 307 files. Refresh it with `tree -a Project_Lotus` after files move.

```text
Project_Lotus
├── .github
│   └── workflows
│       ├── build-and-push-autoencoder.yml
│       ├── build-and-push.yml
│       └── terraform-data-sandbox.yml
├── README.md
├── accounts
│   └── data-sandbox
│       ├── infra
│       │   ├── README.md
│       │   ├── autoencoder_algorithm.tf
│       │   ├── backend.tf
│       │   ├── bronze_arrival.tf
│       │   ├── bronze_arrival_claim_lambda.tf
│       │   ├── bronze_schema_registry.tf
│       │   ├── bronze_schema_validation_iceberg.tf
│       │   ├── bronze_schema_validator_data.tf
│       │   ├── bronze_schema_validator_lakeformation.tf
│       │   ├── bronze_schema_validator_lambda.tf
│       │   ├── bronze_schema_validator_layer.tf
│       │   ├── bronze_schema_validator_prefixes.tf
│       │   ├── data.tf
│       │   ├── developer_lakeformation.tf
│       │   ├── ecr.tf
│       │   ├── emr.tf
│       │   ├── emr_gold_lakeformation.tf
│       │   ├── emr_pp.tf
│       │   ├── emr_runtime_security_configuration.tf
│       │   ├── emr_silver_lakeformation.tf
│       │   ├── gitignore.htm
│       │   ├── gold_iceberg.tf
│       │   ├── gold_job_config.json
│       │   ├── gold_job_config.tf
│       │   ├── iceberg.tf
│       │   ├── lambda_v2
│       │   │   └── pipeline_helper.py
│       │   ├── layers
│       │   │   └── pyarrow_layer.zip
│       │   ├── locals.tf
│       │   ├── providers.tf
│       │   ├── s3.tf
│       │   ├── sagemaker.tf
│       │   ├── sagemaker_pre-processing.tf
│       │   ├── sample-training-data
│       │   │   └── problem=fraud_detection
│       │   │       ├── testing
│       │   │       │   ├── fraud
│       │   │       │   │   └── NBC-v1.0
│       │   │       │   │       └── dt=2026-06-01
│       │   │       │   │           └── data
│       │   │       │   │               └── part-00000.parquet
│       │   │       │   ├── labels
│       │   │       │   │   └── fraud_labels.parquet
│       │   │       │   └── non-fraud
│       │   │       │       └── time_window=2026-06-01_00-00-00
│       │   │       │           └── data
│       │   │       │               └── part-00000.parquet
│       │   │       ├── training
│       │   │       │   └── non-fraud
│       │   │       │       └── time_window=2026-04-01_00-00-00
│       │   │       │           └── data
│       │   │       │               └── part-00000.parquet
│       │   │       └── validation
│       │   │           ├── fraud
│       │   │           │   └── NBC-v1.0
│       │   │           │       └── dt=2026-05-01
│       │   │           │           └── data
│       │   │           │               └── part-00000.parquet
│       │   │           ├── labels
│       │   │           │   └── fraud_labels.parquet
│       │   │           └── non-fraud
│       │   │               └── time_window=2026-05-01_00-00-00
│       │   │                   └── data
│       │   │                       └── part-00000.parquet
│       │   ├── silver_gold_control_data.tf
│       │   ├── silver_gold_control_iceberg.tf
│       │   ├── silver_gold_control_lakeformation.tf
│       │   ├── silver_gold_control_lambda.tf
│       │   ├── silver_job_config.tf
│       │   ├── silver_quarantine_iceberg.tf
│       │   ├── sns_pipeline_alerts.tf
│       │   ├── stepfunctions_bronze_silver.tf
│       │   ├── stepfunctions_data.tf
│       │   ├── stepfunctions_gold.tf
│       │   ├── telecom_fraud_prediction_100_rows_test_data.csv
│       │   ├── terraform.lock.hcl
│       │   ├── terraform.tf
│       │   ├── terraform.tf.vars.example
│       │   ├── terraform.tfvars.example
│       │   ├── trust_score_ml_v2_endpoint.tf
│       │   ├── trust_score_ml_v2_eventbridge.tf
│       │   ├── trust_score_ml_v2_lambda.tf
│       │   ├── trust_score_ml_v2_stepfunctions.tf
│       │   ├── trust_score_ml_v2_variables.tf
│       │   └── variables.tf
│       └── security
│           ├── README.md
│           ├── backend.tf
│           ├── bronze_arrival_claim.tf
│           ├── bronze_schema_validator.tf
│           ├── bronze_schema_validator_lakeformation.tf
│           ├── data.tf
│           ├── ecr_iam.tf
│           ├── emr_gold_runtime.tf
│           ├── emr_iam.tf
│           ├── emr_silver_runtime.tf
│           ├── locals.tf
│           ├── providers.tf
│           ├── sagemaker_iam.tf
│           ├── silver_gold_control.tf
│           ├── silver_gold_control_lakeformation.tf
│           ├── stepfunctions_iam.tf
│           ├── terraform.lock.hcl
│           ├── terraform.tf
│           ├── trust_score_ml_v2_endpoint_iam.tf
│           ├── trust_score_ml_v2_iam.tf
│           └── variables.tf
├── gitignore.txt
└── project_lotus
  ├── Makefile.txt
  ├── README.md
  ├── docs
  │   ├── README.md
  │   ├── docs
  │   │   ├── bronze_tu_portps.md
  │   │   ├── bronze_tu_portps_tulip.md
  │   │   ├── getting-started.md
  │   │   ├── index.md
  │   │   ├── pipeline_orchestration.md
  │   │   ├── silver_pipeline_quarantine.md
  │   │   ├── silver_tu_portps.md
  │   │   ├── silver_tu_portps_tulip.md
  │   │   └── transient_emr_and_bronze_arrivals.md
  │   ├── gitkeep.txt
  │   └── mkdocs.yml
  ├── gitignore.txt
  ├── lambdas
  │   ├── bronze_arrival_claim
  │   │   └── bronze_arrival_claim.py
  │   ├── bronze_schema_validator
  │   │   └── bronze_schema_validator.py
  │   └── silver_gold_control
  │       └── silver_gold_control.py
  ├── notebooks
  │   ├── data_preparation
  │   │   ├── TU_PortPS_silver.ipynb
  │   │   ├── TU_PortPS_silver_tests.ipynb
  │   │   ├── TU_PortPS_tulip_silver.ipynb
  │   │   ├── TU_PortPS_tulip_silver_tests.ipynb
  │   │   ├── account_changes_batch_cleaning.ipynb
  │   │   ├── account_changes_batch_tests.ipynb
  │   │   ├── audit_trail_services_3_cleaning.ipynb
  │   │   ├── audit_trail_services_3_tests.ipynb
  │   │   ├── device_lookup_batch_cleaning.ipynb
  │   │   ├── device_lookup_batch_cleaningipynb.ipynb
  │   │   ├── device_lookup_batch_tests.ipynb
  │   │   └── tests
  │   │       └── test_tu_portps_tulip.py
  │   ├── data_quality_check
  │   │   ├── CLI.txt
  │   │   ├── checks
  │   │   │   ├── dataset=account_changes_batch
  │   │   │   │   └── checks.yaml
  │   │   │   ├── dataset=audit_trail_services_3
  │   │   │   │   └── checks.yaml
  │   │   │   ├── dataset=device_lookup_batch
  │   │   │   │   └── checks.yaml
  │   │   │   ├── dataset=tu_portps
  │   │   │   │   └── checks.yaml
  │   │   │   └── dataset=tu_portps_tulip
  │   │   │       └── checks.yaml
  │   │   ├── schemas
  │   │   │   ├── metrics.schema.json
  │   │   │   └── summary.schema.json
  │   │   ├── silver_dq
  │   │   │   ├── __init__.py
  │   │   │   ├── __main__.py
  │   │   │   ├── config.py
  │   │   │   ├── metrics.py
  │   │   │   └── runner.py
  │   │   ├── silver_dq_metrics_tests.ipynb
  │   │   ├── silver_dq_run.ipynb
  │   │   └── tests
  │   │       ├── conftest.py
  │   │       ├── test_all_checks_yaml.py
  │   │       ├── test_cli.py
  │   │       ├── test_contract.py
  │   │       ├── test_empty_table.py
  │   │       ├── test_idempotency.py
  │   │       ├── test_metrics.py
  │   │       └── test_per_mno.py
  │   ├── gitkeep.txt
  │   └── rule_based_layer
  │       ├── rule_based_layer_step1_highnull_features.ipynb
  │       ├── rule_based_layer_step2_test_datasets.ipynb
  │       ├── rule_based_layer_step3_validation_datasets.ipynb
  │       ├── rule_based_layer_step4_thresholds_and_results.ipynb
  │       └── rule_based_layer_step5_potential_fraud_flag.ipynb
  ├── project_lotus
  │   ├── __init__.py
  │   ├── preprocessing
  │   │   ├── Dockerfile.txt
  │   │   ├── build_and_push.sh
  │   │   ├── preprocess.py
  │   │   ├── requirements.txt
  │   │   └── run_processing_job.py
  │   └── silver_pipeline
  │       ├── README.md
  │       ├── __init__.py
  │       ├── artifacts.py
  │       ├── common.py
  │       ├── config.py
  │       ├── configs
  │       │   ├── account_changes_batch.yaml
  │       │   ├── api_types.yaml
  │       │   ├── audit_trail_services_3.yaml
  │       │   └── device_lookup_batch.yaml
  │       ├── dq.py
  │       ├── publish.py
  │       ├── quarantine.py
  │       ├── references.py
  │       ├── runner.py
  │       ├── transforms
  │       │   ├── __init__.py
  │       │   ├── account.py
  │       │   ├── audit.py
  │       │   └── device.py
  │       └── yaml
  │           ├── PyYAML-6.0.2.dist-info
  │           │   ├── LICENSE.txt
  │           │   ├── METADATA.txt
  │           │   ├── RECORD.txt
  │           │   └── WHEEL.txt
  │           └── yaml
  │               ├── __init__.py
  │               ├── composer.py
  │               ├── constructor.py
  │               ├── cyaml.py
  │               ├── dumper.py
  │               ├── emitter.py
  │               ├── error.py
  │               ├── events.py
  │               ├── loader.py
  │               ├── nodes.py
  │               ├── parser.py
  │               ├── reader.py
  │               ├── representer.py
  │               ├── resolver.py
  │               ├── scanner.py
  │               ├── serializer.py
  │               └── tokens.py
  ├── pyproject.toml
  ├── references
  │   └── gitkeep.txt
  ├── reports
  │   ├── figures
  │   │   └── gitkeep.txt
  │   ├── gitkeep.txt
  ├── sagemaker
  │   ├── conf
  │   │   └── ml
  │   │       ├── base.yaml
  │   │       └── sandbox.yaml
  │   ├── docs
  │   │   ├── README.md
  │   │   ├── docs
  │   │   │   ├── getting-started.md
  │   │   │   ├── handover.md
  │   │   │   └── index.md
  │   │   └── mkdocs.yml
  │   ├── jobs
  │   │   ├── inference
  │   │   │   ├── Dockerfile.txt
  │   │   │   ├── container_entry.py
  │   │   │   ├── inference.py
  │   │   │   ├── requirements-inference.txt
  │   │   │   ├── run_pipeline.py
  │   │   │   ├── serve.py
  │   │   │   └── sm_common.py
  │   │   ├── processing
  │   │   │   ├── Dockerfile.txt
  │   │   │   └── feature_engineering.py
  │   │   └── training
  │   │       ├── Dockerfile.txt
  │   │       ├── requirements-training.txt
  │   │       ├── run_pipeline.py
  │   │       ├── sm_common.py
  │   │       └── train.py
  │   ├── notebooks
  │   │   ├── 0.0-standalone
  │   │   │   ├── 00-setup-and-data.ipynb
  │   │   │   ├── 01-explore-and-validate.ipynb
  │   │   │   ├── 02-select-features.ipynb
  │   │   │   ├── 03-train-and-score.ipynb
  │   │   │   └── 04-compare-and-report.ipynb
  │   │   ├── 1.0-exploration
  │   │   │   ├── 1.1-schema-discovery.ipynb
  │   │   │   └── 1.2-population-and-labels.ipynb
  │   │   ├── 2.0-features
  │   │   │   ├── 2.1-build-feature-matrix.ipynb
  │   │   │   └── 2.2-feature-selection.ipynb
  │   │   ├── 3.0-modeling
  │   │   │   ├── 3.1-preprocess-and-fit.ipynb
  │   │   │   ├── 3.2-sweep.ipynb
  │   │   │   ├── 3.3-evaluate-and-compare.ipynb
  │   │   │   └── 3.4-drift.ipynb
  │   │   └── README.md
  │   ├── scripts
  │   │   ├── package.sh
  │   │   └── submit_emr_serverless.py
  │   ├── tests
  │   │   ├── pandas_spark_standin.py
  │   │   ├── smoke_local.py
  │   │   ├── test_launchers.py
  │   │   └── test_pipeline_helper.py
  │   └── trust_score_05
  │       ├── __init__.py
  │       ├── common
  │       │   ├── __init__.py
  │       │   ├── io
  │       │   │   ├── __init__.py
  │       │   │   ├── s3.py
  │       │   │   └── serialization.py
  │       │   ├── metrics
  │       │   │   ├── __init__.py
  │       │   │   ├── classification.py
  │       │   │   └── ranking.py
  │       │   ├── splitting.py
  │       │   └── viz
  │       │       ├── __init__.py
  │       │       └── plots.py
  │       ├── features
  │       │   ├── __init__.py
  │       │   ├── assemble.py
  │       │   ├── calculators.py
  │       │   ├── config.py
  │       │   ├── docs
  │       │   │   └── feature_library_findings.md
  │       │   ├── enstream.py
  │       │   ├── events.py
  │       │   ├── expressions.py
  │       │   └── snapshots.py
  │       ├── lineage
  │       │   ├── __init__.py
  │       │   ├── config.py
  │       │   ├── dq
  │       │   │   ├── __init__.py
  │       │   │   ├── framework.py
  │       │   │   ├── rules_enrichment.py
  │       │   │   └── rules_gold.py
  │       │   ├── io
  │       │   │   ├── __init__.py
  │       │   │   ├── control.py
  │       │   │   ├── merge_sink.py
  │       │   │   └── reader.py
  │       │   ├── jobs
  │       │   │   ├── __init__.py
  │       │   │   ├── base.py
  │       │   │   ├── imei_map_job.py
  │       │   │   └── lineage_job.py
  │       │   ├── logging_utils.py
  │       │   ├── progress.py
  │       │   ├── schemas.py
  │       │   ├── slice.py
  │       │   ├── spark.py
  │       │   ├── staging.py
  │       │   ├── timezone.py
  │       │   └── transforms
  │       │       ├── __init__.py
  │       │       ├── accounts.py
  │       │       ├── boundaries.py
  │       │       ├── canonical_events.py
  │       │       ├── chains.py
  │       │       ├── common.py
  │       │       ├── customers.py
  │       │       ├── imei_map.py
  │       │       ├── lifecycle.py
  │       │       ├── normalized.py
  │       │       └── tu_lifecycle.py
  │       └── ml
  │           ├── __init__.py
  │           ├── config.py
  │           ├── datasets.py
  │           ├── discovery.py
  │           ├── docs
  │           │   └── ml_pipeline_findings.md
  │           ├── drift.py
  │           ├── evaluation.py
  │           ├── grids.py
  │           ├── jobs
  │           │   ├── __init__.py
  │           │   ├── base.py
  │           │   ├── discovery_job.py
  │           │   ├── selection_job.py
  │           │   └── sweep_job.py
  │           ├── models.py
  │           ├── paths.py
  │           ├── preprocessing.py
  │           ├── selection.py
  │           ├── sweep.py
  │           └── taxonomy.py
  └── tests
    └── test_data.py
```

## Phase 1 and Phase 2 delivery context

The attached vendor brief and Trust Score 0.5 MVP describe target scope and
acceptance criteria. This repository contains enabling implementation, not
evidence that every criterion has passed against production data.

| Brief area | Present in this checkout | Boundary or evidence still required |
| --- | --- | --- |
| Phase 1 data substrate and Bronze/Silver/Gold batch processing | Terraform data-sandbox resources; Bronze arrival claim/schema validation; three-dataset Silver transform, DQ/quarantine, and publish; lineage and feature code/tests | Input files must already be delivered to configured S3 paths. The brief's MySQL incremental loader, TU/MNO API pollers, and SFTP exchange lander are not implemented here. |
| Phase 1 modular ML and batch workflow | Reusable lineage/features/ML package; discovery, feature selection, sweep/evaluation, champion bundle, batch scoring, local synthetic notebooks and tests | Confirm real-data contracts; run end-to-end on a controlled cohort; record score/reason-code/tier reconciliation and operational acceptance. Existing artifacts do not prove the MVP's full Trust Score 0-100, reason-code, or potential-fraud-flag acceptance. |
| Phase 1 DQ, run records, package and runbook | Silver YAML DQ/quarantine and run artifacts; ML discovery gates, run IDs/manifests, summaries, tests, runbooks, Terraform workflows | The orchestration document includes design-only portions. Confirm live gates, freshness/count alerting, backfill, recovery, and acceptance metrics in the target account. |
| Phase 2 scheduled automation and handoff | Terraform Step Functions/EventBridge/SageMaker/EMR resources, CI workflows, submission scripts, versioned run artifacts, handover documentation | Demonstrate schedules, retries, dependencies, delivery contracts, SLA/freshness alerts, rollback, shadow reconciliation, and platform-level DQ in staging/production. This repository alone does not show those criteria have been achieved. |
| MVP model scope | Unsupervised model registry/sweeps, per-use-case configuration and feature selection, evaluation/drift code, serialized champion and batch inference | Real-time API inference, online feature store, operational customer-specific calibration, lifecycle retraining/promotion, and graph ML are later-phase or not demonstrated by this batch implementation. |

## Data flow, inputs, and outputs

| Component | Main inputs | Main outputs |
| --- | --- | --- |
| Bronze arrival and schema-validation Lambdas | Arrival event, S3 Parquet folder, schema registry/version, control-table settings | Claim/validation status, per-file schema results in the control table, optional SNS alert |
| Silver pipeline | One approved Bronze Parquet dataset; audit partner/provider references when applicable; YAML dataset contract | Silver Iceberg append of accepted new rows, dataset quarantine Iceberg append, issue artifacts and run summary |
| Gold/lineage jobs | Silver source tables, lineage/reference inputs, YAML configuration and run controls | Gold Iceberg lineage/entity tables, control markers, DQ reports and run logs |
| Feature Engineering Processing job | Configured Gold/Silver lineage and event tables, feature-window parameters, optional fraud labels | Training feature groups/windows or one scoring feature batch in Parquet, manifests and run artifacts |
| ML discovery/selection/sweep/training | Configured Parquet populations and labels, ML YAML overlays, approved feature/model settings | Schema/DQ reports, ranked feature lists, per-arm models/metrics, leaderboard, champion and drift references |
| Batch inference | Feature Parquet batch, date/configuration, champion bundle and reference distributions | All scored records, top-k review queue, feature/score drift report, retraining recommendation and summary |

## Mini runbook

1. Read the data governance section above and confirm the account and
  classification. Use synthetic data in the sandbox; Confidential or
  Restricted data is prohibited there.
2. For documentation-only work, follow `project_lotus/docs/README.md` or
  `project_lotus/sagemaker/docs/README.md`. For Python work, use the SageMaker
  getting-started guide and tests. The active Silver package has no dedicated
  `tests/silver/` directory in this checkout; review its YAML contract and
  quarantine guide.
3. Provision security before infra. From each Terraform root run
  `terraform fmt -recursive`, `terraform init`, `terraform validate`, and
  `terraform plan`; apply only through the approved pull-request pipeline.
4. Land a small approved Parquet smoke partition, pass the Bronze schema gate,
  and validate one Silver dataset before enabling writes. For publish runs,
  verify Iceberg tables, permissions, run ID, output location, quarantine
  capacity, and recovery owner before submission.
5. Run lineage/features and ML stages using their config and handover guides.
  Discovery must pass before selection/sweep; use one run ID across sweep
  slices and finalize only when required arms are ready. Score only after a
  champion is published and the full feature contract is present.
6. Review run summaries, DQ/quarantine counts, freshness, output completeness,
  score/feature drift, and reconciliation before releasing results. On
  failure, stop downstream release, retain artifacts, and follow the relevant
  Bronze/Silver/Gold recovery procedure.

This runbook is a development handoff, not a substitute for Phase 2 production
on-call, rollback, SLA, or vendor acceptance sign-off.
