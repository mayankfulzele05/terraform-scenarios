# Lab: Fixing AWS Authentication Errors in Terraform

## 🚨 The Situation
You run `terraform plan` or `terraform init` and get hit with a wall of red text saying:
`Error: error configuring Terraform AWS Provider: no valid credential sources found` 
or
`Error: ExpiredToken: The security token included in the request is expired`

This lab teaches you how to properly set up authentication between your local terminal and AWS so Terraform can work safely.

---

## 📋 Method 1: The Quick Fix (Environment Variables)

The fastest way to give Terraform temporary access is to export your AWS credentials directly into your terminal session. 

1. Gather your keys from your AWS IAM console or your cloud portal.
2. Run the following commands in your terminal (replace the values with your actual keys):

**For Linux / macOS:**
```bash
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_DEFAULT_REGION="us-east-1"
```

**For Windows (PowerShell):**
```powershell
\$env:AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
\$env:AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
\$env:AWS_DEFAULT_REGION="us-east-1"
```

3. Test it by running `aws sts get-caller-identity` or `terraform plan`.

---

## 📋 Method 2: The Production Way (AWS CLI Profiles)

Hardcoding keys into your terminal can lead to accidental leaks. The standard corporate way is to use the **AWS CLI** to create encrypted local profiles.

1. Install the AWS CLI on your machine and run:
   ```bash
   aws configure --profile my-company-prod
   ```
2. Enter your Access Key, Secret Key, and default region when prompted.
3. Update your Terraform code (`provider.tf` or `main.tf`) to reference this profile:

```hcl
provider "aws" {
  region  = "us-east-1"
  profile = "my-company-prod" # Tells Terraform exactly which local profile to use!
}
```

---

## 📋 Method 3: Fixing Expired Tokens (AWS SSO)

If your team uses AWS IAM Identity Center (SSO), your terminal login expires every few hours.

1. If you see an `ExpiredToken` error, re-authenticate your terminal session:
   ```bash
   aws sso login --profile my-company-prod
   ```
2. Once the browser window confirms success, rerun your `terraform plan`.
