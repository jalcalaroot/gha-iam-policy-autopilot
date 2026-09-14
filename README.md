# gha-iam-policy-autopilot

A composite GitHub Action that generates a baseline least-privilege IAM policy from a Terraform plan, using AWS Labs' [`iam-policy-autopilot`](https://github.com/awslabs/iam-policy-autopilot) — static analysis only, no live AWS account touched. Shared across every repo in this account that provisions AWS infrastructure with Terraform, so the check is written once and stays consistent instead of being copy-pasted (and drifting) per repo.

## Why this exists

Checkov already catches *shape* problems in a hand-written IAM policy (`Resource: "*"`, full-admin statements, etc.) — see any of this account's Terraform repos. What it can't tell you is whether the policy you wrote is actually *complete or too broad* for what the plan creates. `iam-policy-autopilot` reads the plan and maps each planned resource to the AWS API actions the Terraform AWS provider actually performs to create it, giving you a policy to diff against what's committed by hand.

## Usage

Inside an existing `terraform plan` job, right after producing the plan:

```yaml
- name: Terraform plan
  run: terraform plan -input=false -out=plan.tfplan

- name: Terraform plan (JSON)
  run: terraform show -json plan.tfplan > plan.json

- name: IAM Policy Autopilot
  uses: jalcalaroot/gha-iam-policy-autopilot@<pinned-sha>
  with:
    plan-file: plan.json
    region: us-east-1        # optional, used for resource ARNs
    account-id: "123456789012" # optional, used for resource ARNs
```

The generated baseline policy is uploaded as a workflow artifact (`iam-policy-autopilot-baseline`) for manual review against whatever is hand-written in the repo — this action doesn't fail the build or compare anything automatically, it only generates and publishes the baseline.

## Inputs

| Input | Required | Description |
|---|---|---|
| `plan-file` | yes | Path to the Terraform plan in JSON format (`terraform show -json <planfile>`) |
| `region` | no | AWS region used for resource ARNs in the generated policy |
| `account-id` | no | AWS account ID used for resource ARNs in the generated policy |

## Outputs

| Output | Description |
|---|---|
| `policy-file` | Path to the generated baseline policy JSON file |

## Design notes

- **Static only, no credentials needed.** The action never calls AWS — it reads the plan JSON that CI already produced. No `id-token: write`, no `configure-aws-credentials` step.
- **Telemetry disabled explicitly** (`DISABLE_IAM_POLICY_AUTOPILOT_TELEMETRY=true`) — a CI runner shouldn't phone home by default.
- **Runs via `uvx`** (no separate install step) — `astral-sh/setup-uv` is the only other action this depends on, kept minimal on purpose.
- Consuming repos should pin this action by commit SHA, same convention as every other action reference in this account's workflows — never a floating tag.

## Test

`.github/workflows/test.yml` runs this action against a fixture in `test/` (a single `aws_s3_bucket` + `aws_iam_role`, fake credentials, `skip_credentials_validation = true` — no real AWS account involved) on every push/PR, asserting the output is valid JSON with at least one generated policy.
