# Lab 2.5: IaC as Compliance Evidence (AWS)

An S3 Object Lock vault and evidence capture script that packages a Terraform workspace into a tamper-evident, immutable bundle — producing audit evidence that no screenshot can match.

---

## 1. What this lab is

This lab builds two things: an **evidence vault** (an S3 bucket with Object Lock enabled) and a **capture script** that packages the output of a Terraform workspace into a cryptographically verifiable bundle and uploads it to the vault. The bundle is locked with Object Lock GOVERNANCE retention, meaning it cannot be deleted or modified for the retention period — even by the account owner without explicit bypass.

The result is compliance evidence with three properties auditors require:

| Property | How it's delivered |
|---|---|
| **Integrity** | SHA-256 hash of every file in the bundle, recorded in `manifest.json` |
| **Attribution** | `commit.txt` captures the exact git commit, author, and timestamp |
| **Reproducibility** | `plan.json` and `state.json` show exactly what Terraform planned and deployed |

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

---

## 2. Why it matters

Labs 2.3 and 2.4 prove that compliant infrastructure can be built. But an auditor reviewing a control years later doesn't just want to know that it was compliant — they want proof it was compliant at a specific point in time and that the proof itself hasn't been altered since.

A screenshot can be edited. A PDF can be regenerated. A git commit can be amended. Object Lock cannot be bypassed without leaving a trace in CloudTrail, and the SHA-256 manifest means any single-byte change to the evidence is immediately detectable.

In a FedRAMP or SOC 2 audit scenario, this answers the hardest chain-of-custody question:

> *"How do I know this evidence hasn't been modified since it was collected?"*

The answer is the S3 VersionId in `receipt.json` — an immutable, AWS-generated identifier tied to the exact bytes uploaded. The auditor can verify the VersionId against the bucket independently. No trust required.

---

## 3. Key design decisions

**GOVERNANCE mode, not COMPLIANCE.** COMPLIANCE mode cannot be bypassed by anyone — including account root — until the retention period expires. For a lab environment, this would make cleanup impossible without waiting days. GOVERNANCE mode allows bypass with explicit intent (`--bypass-governance-retention`) and a privileged IAM role. A real production evidence vault should use COMPLIANCE.

**SHA-256 manifest over individual files, not just the bundle.** The capture script hashes each file before packaging and writes a `manifest.json` inside the bundle. An auditor can open the bundle, hash any file, and compare against `manifest.json` without re-running the script.

**VersionId as the receipt anchor.** S3 generates a VersionId the moment an object is written, tied to the exact bytes stored. Storing it in `receipt.json` creates an unforgeable pointer from the receipt to the evidence. The receipt is then committed to git, creating a second chain of custody.

**Bucket policy denies `s3:DeleteBucket` to all principals.** Object Lock protects objects but not the bucket itself. The bucket policy adds a second layer — even a privileged IAM user cannot delete the vault without modifying the policy first, which leaves a CloudTrail event.

---

## 4. Results

After running `capture-evidence.sh`, the vault contains a locked bundle and the script outputs:

```json
{
  "run_id": "test-001",
  "vault": "cgep-lab-grc-evidence-vault-e4dc9db3",
  "key": "runs/test-001/bundle.tar.gz",
  "version_id": "abc123XYZ...",
  "captured_at_utc": "2026-05-26T02:40:44Z"
}
```

Destructive test — attempting to delete the locked object returns:
```
An error occurred (AccessDenied) when calling the DeleteObject operation:
Object Lock is enabled.
```

This `AccessDenied` is the proof of immutability.

---

## 5. How to reproduce

**Prerequisites:** Terraform >= 1.6, AWS CLI v2, Bash (Git Bash on Windows), a completed Lab 2.3 workspace.

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

---

## Evidence bundle contents

| File | Source | What it proves |
|---|---|---|
| `plan.json` | `terraform show -json tfplan` | Pre-deploy intent |
| `state.json` | `terraform state pull` | Post-deploy confirmed state |
| `commit.txt` | `git log -1 --pretty=full` | Exact code version reviewed |
| `version.txt` | `terraform version` | Tool version for reproducibility |
| `manifest.json` | SHA-256 of all above | Tamper detection |

## Project structure

```
scripts/capture-evidence.sh        Evidence capture and upload script (LF line endings)
terraform/primitives/evidence-vault/
    main.tf                        S3 Object Lock vault with encryption, public access block, bucket policy
    variables.tf                   project_name, lock_mode, retention_days
    outputs.tf                     vault_name
evidence/lab-2-5/
    receipt.json                   VersionId receipt from successful upload
```
