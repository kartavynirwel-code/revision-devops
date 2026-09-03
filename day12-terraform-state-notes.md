# Day 12 — Terraform State Management Revision Notes

## 1. Why State Matters
- Terraform's `.tfstate` file maps your `.tf` config to real-world resource IDs — it's how Terraform knows "this `aws_instance.web` block corresponds to EC2 instance `i-0abc123` that already exists," instead of trying to create a duplicate every run.
- Without state, Terraform would have no memory of what it already created.

## 2. Remote State (S3 + DynamoDB Lock Table)

### Why Local State Is Dangerous in Teams
- Local `terraform.tfstate` lives only on one person's machine — a second team member running `terraform apply` has no idea what the first person already changed, causing state drift or accidental resource duplication/destruction.
- No locking — two people running `apply` at the same time can corrupt the state file or cause conflicting changes to real infrastructure simultaneously.

### The Standard AWS Pattern
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```
- **S3 bucket**: Stores the actual state file — versioning should be enabled on the bucket so you can recover a previous state if something goes wrong.
- **DynamoDB table**: Used purely for **state locking** — when someone runs `terraform apply`, Terraform writes a lock entry to DynamoDB; anyone else running `apply` at the same time gets blocked until the lock releases.
- **`encrypt = true`**: State files can contain sensitive data (DB passwords, secrets in outputs) — encrypt at rest.
- **Interview point**: The `key` path convention (e.g., `prod/network/terraform.tfstate`) lets you organize multiple state files per environment/component in the same bucket — this is how teams avoid one giant monolithic state file for the entire infrastructure.

## 3. `terraform state` Commands

```bash
terraform state list                          # list all resources tracked in state
terraform state show aws_instance.web         # show detailed attributes of one resource in state

terraform state mv aws_instance.web aws_instance.web_server
# renames a resource in state WITHOUT destroying/recreating it in real infra
# (used when you refactor .tf code but the actual cloud resource shouldn't change)

terraform state rm aws_instance.web
# removes a resource from Terraform's state WITHOUT destroying the real resource
# (Terraform "forgets" about it — useful when handing a resource off to be
#  managed manually or by another tool)

terraform import aws_instance.web i-0abc123def456
# brings an EXISTING resource (created manually or by another tool) under
# Terraform management by writing it into state — you still need matching
# .tf config for it, import only populates state, not the config file
```

### Interview Point — `state mv` vs `state rm` vs `import`
- **`state mv`**: Code refactor scenario — resource stays exactly as is in the cloud, just changes its logical name/address in Terraform's tracking.
- **`state rm`**: "Stop managing this" scenario — resource keeps existing in the cloud, but Terraform no longer tracks or will touch it.
- **`import`**: "Start managing this" scenario — an existing resource (created outside Terraform, e.g. manually via AWS Console) gets adopted into Terraform's state so future `plan`/`apply` account for it.

## 4. Workspaces vs Separate State Files

### Terraform Workspaces
```bash
terraform workspace new staging
terraform workspace new production
terraform workspace select staging
terraform workspace list
```
- Each workspace gets its **own state file**, but shares the **same `.tf` configuration code**.
- Access current workspace in code: `terraform.workspace` (e.g., use it to conditionally set instance sizes per environment).

### Separate State Files (Directory-per-Environment) Pattern
```
environments/
├── dev/
│   ├── main.tf
│   └── backend.tf   (different S3 key per env)
├── staging/
│   ├── main.tf
│   └── backend.tf
└── prod/
    ├── main.tf
    └── backend.tf
```
- Each environment has its own directory, potentially its own `.tf` files (not just values) — fully independent, can even use different Terraform versions or provider configs per env.

### Interview Point — Which to Choose
- **Workspaces**: Good for lightweight environment differences (same infra shape, different sizing/counts) — less duplication, but risk of accidentally running `apply` in the wrong workspace since the code looks identical.
- **Separate directories/state files**: Preferred when environments genuinely diverge in structure (prod has multi-AZ HA setup, dev doesn't) or when you want **hard isolation** — a mistake in dev's `.tf` code can't accidentally touch prod's state at all, since they're completely separate configurations and backends.
- Most production setups favor **separate state files per environment** (sometimes combined with modules to share reusable logic) over workspaces, specifically for this blast-radius isolation reason.

---

## Quick Self-Test (do this without looking)
1. What two AWS resources make up the standard remote state backend, and what does each one specifically do?
2. You refactored your Terraform code and renamed a resource block, but don't want the real cloud resource destroyed and recreated — which command do you use?
3. What's the actual difference between `terraform state rm` and `terraform destroy` for a given resource?
4. Why do most production teams prefer separate state files per environment over Terraform workspaces, despite workspaces requiring less code duplication?
