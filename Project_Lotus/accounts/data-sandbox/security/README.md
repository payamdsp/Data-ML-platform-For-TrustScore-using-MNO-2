# Project Lotus, data-sandbox, security layer

See the [repository README](../../../README.md) for the layer split, the pipeline and the
conventions.

Empty of managed resources. It holds one data source, the permissions boundary lookup.

## Belongs here

KMS keys, aliases and key policies. Secrets Manager containers and their policies. All IAM roles and
policies, including the execution roles the infra layer's services run as. S3 bucket policies.

## The permissions boundary

Every role this root creates must set `permissions_boundary = local.permissions_boundary`, which
`data.tf` and `locals.tf` resolve from the account's contract parameter.

```hcl
resource "aws_iam_role" "example" {
  name                 = "${var.project}-${var.environment}-example"
  permissions_boundary = local.permissions_boundary
}
```

### Naming

| | This account |
| -- | -- |
| Policy | `sandbox-boundary` |
| Published at | `/contract/_account/iam/sandbox-boundary-arn` |

Every other account names its boundary for the account slug, `<slug>-boundary`, published at
`/contract/_account/iam/<slug>-boundary-arn`. Read the target account's real parameter on promotion.

## IAM

Least privilege applies. Scope every policy to the actions and resources the role requires. Do not
attach broad AWS managed policies such as `PowerUserAccess`, and do not grant `Action: "*"` or
`Resource: "*"`.

## What this Terraform root runs

This is the security half of the `data-sandbox` account deployment. The root
looks up the account permissions boundary and defines IAM roles and policies for
the Bronze arrival claim and schema validator, EMR Silver and Gold runtimes,
Step Functions, SageMaker, ECR, the Silver-to-Gold control Lambda, and the Trust
Score ML v2 resources. Resource definitions are divided across the `.tf` files
named for those services. The companion `infra/` root owns data and compute
resources; this root owns identity, key, secret, and bucket-policy controls.

Inputs are Terraform variables and account contract values resolved in
`data.tf`/`locals.tf`, including the published boundary ARN. The outputs are
AWS IAM/security resources consumed by the infra root and pipeline workflows;
this root does not process records or write Bronze, Silver, Gold, or model-score
data.

## Run procedure

Run from this directory with a configured AWS SSO/profile and the backend
settings approved for the target account:

```bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan
```

Review the plan and apply only through the approved pull-request workflow. CI
validates pull requests without AWS credentials and applies merged changes to
`main` using the layer-specific CI role. Keep security and infra changes in
separate pull requests. Never commit state, `.terraform/`, plans, or a
`terraform.tfvars` file containing real account values.

## Operational notes

- Every role must use `permissions_boundary = local.permissions_boundary`.
- Use the account's actual boundary parameter when promoting; do not copy the
  sandbox boundary ARN or hardcode account IDs/ARNs.
- Scope policies to required actions and resources. An IAM resource belongs in
  this root even when its service is defined in `infra/`.
- Example tfvars files are examples, not credentials or proof that a target
  account is ready to deploy.
