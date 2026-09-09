# Lab: Recovering a Resource Missing from Terraform State

## 🚨 The Situation
You have a critical production resource (like an EC2 instance or an S3 bucket) that is fully defined in your code and actively running in AWS. However, when you run `terraform plan`, Terraform treats it as brand new and wants to **Create** it. Running `terraform apply` crashes because the resource already exists in the real world.

This lab teaches you how to stitch the real cloud resource back into your state file without changing your code or triggering downtime.

---

## 🛠️ The Disconnected Setup
Imagine you have this configuration block in your `compute.tf` file:

```hcl
resource "aws_instance" "prod_api" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "production-api-server"
  }
}
```
*The instance is running in AWS with the physical ID `i-0123456789abcdef0`, but `terraform state list` shows absolutely nothing.*

---

## 📋 The Recovery Strategy (Terraform 1.5+)

The safest and most modern way to fix a missing state link is using a declarative `import` block. This allows you to preview the link via `terraform plan` before changing anything in the cloud.

### Step 1: Add the Import Link Block
Add an `import` block anywhere in your configuration files (e.g., at the top of `compute.tf`):

```hcl
import {
  # 1. The actual cloud ID of the missing resource from your AWS Console
  id = "i-0123456789abcdef0"

  # 2. The exact HCL address path of the resource block in your code
  to = aws_instance.prod_api
}
```

### Step 2: Run a Targeted Plan Verification
Run the plan command to see if Terraform can successfully locate the real asset and map it to your code:

```bash
terraform plan
```
*Review the stdout carefully. You should see a success message stating:*
`Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.`

### Step 3: Commit the State Sync
Apply the plan to write the real-world tracking data back into your `terraform.tfstate` logbook:

```bash
terraform apply
```
Once the apply completes, Terraform has fully regained its memory. 

### Step 4: Code Cleanup
Now that the logbook is fixed, you can safely **delete the `import` block** from your `compute.tf` file. Future runs of `terraform plan` will now show `No changes. Your infrastructure matches the configuration.`
