# Lab: Fixing "terraform plan succeeds but apply fails" Errors

## 🚨 The Situation
You execute `terraform plan` and receive a clean output: `Plan: 3 to add, 0 to change, 0 to destroy.` However, when you run `terraform apply`, execution abruptly crashes halfway through with a severe cloud provider API error.

This lab teaches you how to identify runtime blocks and resolve real-world environment limits.

---

## 🛠️ The Deceptive Code Setup
Imagine you write the following infrastructure blueprint (`storage.tf`):

```hcl
resource "aws_s3_bucket" "global_cache" {
  # ❌ Pass syntax checks perfectly, but fails at execution time!
  bucket = "my-app-cache" 
}
```

### Why the Plan Succeeds:
Terraform validates that `bucket` takes a string value and matches S3 syntax guidelines. The plan passes without issues.

### Why the Apply Fails:
AWS S3 bucket names must be **globally unique** across all AWS accounts worldwide. When Terraform attempts to provision it during `terraform apply`, the AWS API rejects the request with an error:
`Error: OperationAborted: The requested bucket name is not available.`

---

## 📋 The Troubleshooting & Fix Strategies

### Step 1: Diagnose the Error Class
Look closely at the error block thrown during the apply execution phase. They generally fall into three distinct buckets:

| Error Type | Common Message Example | How to Fix It |
| :--- | :--- | :--- |
| **Global Name Lock** | `BucketNameUnavailable`, `Conflict` | Change the name parameter to be more unique. |
| **Quota Breached** | `InstanceLimitExceeded`, `VpcLimitExceeded` | Request a quota increase in the cloud console, or scale down your counts. |
| **Access Denied** | `UnauthorizedOperation`, `AccessDenied` | Upgrade the IAM policy of your active deployment credentials. |

---

### Step 2: Implement a Unique Name Resolution
To fix the global namespace issue shown in our setup block, use string interpolation combined with a random resource utility to guarantee uniqueness without hardcoding random strings.

Update your configuration to include a `random_id` block:

```hcl
# Generates a unique 4-byte random suffix hex string
resource "random_id" "bucket_suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "global_cache" {
  # Resolves runtime naming collisions dynamically!
  bucket = "my-app-cache-\${random_id.bucket_suffix.hex}" 
}
```

---

### Step 3: Clear and Re-Execute
Because your previous apply failed partially, verify your tracking memory is intact before re-running the configuration pipeline:

1. Validate the new configuration layout:
   ```bash
   terraform plan
   ```
2. Finalize the deployment execution:
   ```bash
   terraform apply
   ```
   **Result:** Terraform reads the newly generated unique name variation, bypasses the API block restrictions, and completes the provision graph cleanly.
