# Lab 2.5 — IaC as Compliance Evidence (AWS)

> **CGEP Lab Series · Module 2 · Lab 5**
> A reviewed, signed, immutably-stored Terraform commit is stronger evidence than a screenshot.

## What This Lab Builds

This lab builds an **evidence vault** — an S3 bucket with Object Lock enabled — and a capture script that packages a Terraform workspace into a tamper-evident bundle and uploads it to the vault. The result is compliance evidence that satisfies the three properties auditors require:

| Property | How it's delivered |
|---|---|
| **Integrity** | SHA-256 hash of every file in the bundle, recorded in `manifest.json` |
| **Attribution** | `commit.txt` captures the exact git commit, author, and timestamp |
| **Reproducibility** | `plan.json` and `state.json` show exactly what Terraform deployed |

## Architecture

```
Lab 2.3 workspace          capture-evidence.sh            Object Lock Vault
─────────────────          ───────────────────            ─────────────────
tfplan, .tf files  ──▶     collect + SHA-256 each  ──▶   s3://VAULT/runs/RUN_ID/
git log, state             tar into bundle                bundle.tar.gz
                           aws put-object                 Mode: GOVERNANCE
                                  │                       Retention: 1 day
                                  ▼
                          receipt.json
                          (run_id, key, version_id)
```

## Repository Structure

```
cgep-lab-2.5/
├── .gitattributes                              # Enforces LF line endings on shell scripts
├── .gitignore                                  # Excludes Terraform state and provider cache
├── evidence/
│   └── lab-2-5/
│       └── receipt.json                        # Upload receipt with S3 VersionId
├── scripts/
│   └── capture-evidence.sh                     # Evidence capture and upload script
└── terraform/
    └── primitives/
        └── evidence-vault/
            ├── main.tf                         # S3 Object Lock vault + all controls
            ├── variables.tf                    # project_name, lock_mode, retention_days
            └── outputs.tf                      # vault_name output
```

## Key Resources Deployed

| Resource | Purpose |
|---|---|
| `aws_s3_bucket` | The vault bucket with `object_lock_enabled = true` |
| `aws_s3_bucket_versioning` | Required by Object Lock; every object gets a VersionId |
| `aws_s3_bucket_object_lock_configuration` | Sets GOVERNANCE mode, 1-day default retention |
| `aws_s3_bucket_server_side_encryption_configuration` | AES-256 encryption at rest |
| `aws_s3_bucket_public_access_block` | Blocks all public access |
| `aws_s3_bucket_policy` | Denies `s3:DeleteBucket` to all principals except account root |

## Prerequisites

- AWS CLI v2 with a configured profile
- Terraform >= 1.6
- Bash (Git Bash on Windows, or any POSIX shell on Linux/macOS)
- A completed Lab 2.3 workspace (source of evidence files)

## Usage

### 1. Deploy the vault

```bash
cd terraform/primitives/evidence-vault
terraform init
terraform apply -auto-approve
VAULT=$(terraform output -raw vault_name)
```

### 2. Capture evidence from a Terraform workspace

```bash
bash scripts/capture-evidence.sh \
  --workspace <path-to-terraform-workspace> \
  --run-id    test-001 \
  --vault     "$VAULT" \
  --profile   <aws-profile>
```

The script outputs a single-line JSON receipt:

```json
{
  "run_id": "test-001",
  "vault": "cgep-lab-grc-evidence-vault-XXXXXXXX",
  "key": "runs/test-001/bundle.tar.gz",
  "version_id": "<s3-version-id>",
  "captured_at_utc": "<iso-utc-timestamp>"
}
```

Save this to `evidence/lab-2-5/receipt.json`.

### 3. Verify Object Lock

```bash
# Bucket-level lock configuration
aws s3api get-object-lock-configuration --bucket "$VAULT" --profile <profile>

# Object-level retention
aws s3api get-object-retention \
  --bucket "$VAULT" \
  --key runs/test-001/bundle.tar.gz \
  --profile <profile>
```

### 4. Proof of immutability (destructive test)

```bash
aws s3api delete-object \
  --bucket "$VAULT" \
  --key runs/test-001/bundle.tar.gz \
  --version-id "<version-id>" \
  --profile <profile>
# Expected: AccessDenied because object protected by object lock
```

### 5. Cleanup (GOVERNANCE mode only)

```bash
aws s3api delete-object \
  --bucket "$VAULT" \
  --key runs/test-001/bundle.tar.gz \
  --version-id "<version-id>" \
  --bypass-governance-retention \
  --profile <profile>

cd terraform/primitives/evidence-vault
terraform destroy -auto-approve
```

## GOVERNANCE vs COMPLIANCE Mode

| Mode | Can be bypassed? | Use case |
|---|---|---|
| `GOVERNANCE` | Yes, with `--bypass-governance-retention` by privileged callers | Lab work, testing |
| `COMPLIANCE` | No — not even by root until retention expires | Real production evidence |

This lab uses **GOVERNANCE** so cleanup is possible. Switch to **COMPLIANCE** for real evidence vaults.

## What the Evidence Bundle Contains

Each `bundle.tar.gz` contains:

| File | Source |
|---|---|
| `plan.json` | `terraform show -json tfplan` |
| `state.json` | `terraform state pull` |
| `commit.txt` | `git log -1 --pretty=full` |
| `version.txt` | `terraform version` |
| `manifest.json` | SHA-256 hash + size of every file above |

## Portfolio Checklist

- [x] `terraform/primitives/evidence-vault/` deploys a fully configured Object Lock vault
- [x] `scripts/capture-evidence.sh` is committed with LF line endings and is executable
- [x] `evidence/lab-2-5/receipt.json` contains a real VersionId from a successful upload
- [x] Destructive test confirmed `AccessDenied` from Object Lock

## Lab Reference

[Lab 2.5 Guide — IaC as Compliance Evidence](https://github.com/GRCEngClub/cgep-labs/blob/main/guides/02_05_iac_as_compliance_evidence.md)
