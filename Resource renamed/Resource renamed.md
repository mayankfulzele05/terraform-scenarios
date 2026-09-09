# Lab: Safely Renaming Resources in Terraform Without Downtime

## 🚨 The Situation
You are refactoring your infrastructure code to match new team naming standards. You rename a resource block definition inside a `.tf` file. When you run `terraform plan`, Terraform fails to realize it is the same asset and schedules the production resource to be **Destroyed and Recreated**.

This lab teaches you how to map code renames directly to your state logbook, ensuring zero downtime and zero resource recreation.

---

## 🛠️ The Refactored Code Setup
Imagine you are managing a core production storage bucket.

#### Before Refactoring:
```hcl
resource "aws_s3_bucket" "company_assets_bucket" {
  bucket = "my-global-company-assets-prod"
}
```

#### After Refactoring (The Change):
You update the block name to follow a shorter format, but leave the actual cloud bucket configuration exactly the same:
```hcl
resource "aws_s3_bucket" "assets" {
  bucket = "my-global-company-assets-prod"
}
```

If you run `terraform plan` right now, you will see a destructive plan:
`Plan: 1 to add, 0 to change, 1 to destroy.`

---

## 📋 The Solution: The Declarative `moved` Block (Terraform 1.1+)

The absolute safest way to handle resource renaming is by writing a `moved` block directly into your configuration files. This acts as a clear historical record for your team and updates the state file automatically on the next plan.

### Step 1: Add the Moved Statement
Add a `moved` configuration block anywhere in your code files (e.g., at the bottom of `main.tf`):

```hcl
moved {
  # The original HCL address path before the rename
  from = aws_s3_bucket.company_assets_bucket

  # The updated HCL address path after the rename
  to   = aws_s3_bucket.assets
}
```

### Step 2: Verify the Execution Plan
Run the plan command in your terminal:

```bash
terraform plan
```
*Review the stdout carefully. Instead of showing text about destruction and additions, you will see a clean execution statement:*
```text
Plan: 0 to add, 1 to change, 0 to destroy.
```
*(Terraform will explicitly log that the resource has been renamed internally within the state tracking engine).*

### Step 3: Apply the Sync Configuration
Run the apply command to finalize the record switch inside your state file:

```bash
terraform apply
```
**Result:** The live AWS S3 bucket remains completely untouched and online. The internal logbook path is safely updated to `aws_s3_bucket.assets`. You can leave the `moved` block in your code repository permanently so your team members don't encounter errors when pulling your code changes!
