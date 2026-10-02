# Data-ML-platform-For-TrustScore-using-MNO-2

This workspace contains [Project Lotus](Project_Lotus/README.md), its AWS
Terraform roots and Python/SageMaker code. The attached planning references are
the vendor platform brief and [Trust Score 0.5 MVP](Trust%20Score%200.5%20-%20MVP.txt);
they are context, not deployment inputs.

Start with the [Project Lotus overview](Project_Lotus/README.md) for the Phase 1
and Phase 2 scope mapping, data-flow inputs and outputs, run procedures, safety
notes, and complete source tree. The primary implementation-specific guides are
the [Silver job](Project_Lotus/project_lotus/project_lotus/silver_pipeline/README.md),
[ML handover](Project_Lotus/project_lotus/sagemaker/docs/docs/handover.md),
[ML notebooks](Project_Lotus/project_lotus/sagemaker/notebooks/README.md), and
[pipeline orchestration design](Project_Lotus/project_lotus/docs/docs/pipeline_orchestration.md).

The reference documents describe target scope and acceptance criteria as well
as the current project context. Do not treat a planned capability or an
acceptance criterion as proof that it has been implemented or production-
validated; the project overview calls out that distinction.