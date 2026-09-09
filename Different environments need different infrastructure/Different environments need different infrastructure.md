# Lab: Managing Multiple Infrastructure Environments in Terraform

## 🚨 The Situation
You need to provision a lightweight single-node compute instance for the **Development** team, but a robust, multi-instance, highly available deployment for **Production**—all without duplicating your underlying `.tf` source code.

This lab teaches you how to leverage environment-specific variable files and structural isolation to scale safely.

---

## 🛠️ The Architecture Blueprint (`main.tf`)
Create a core template file that dynamically scales its size, count, and settings based on the environment variables fed into it:

```hcl
terraform {
  required_version = ">= 1.5.0"
}

variable "environment" {
  type        = string
  description = "The target deployment stage (dev, qa, prod)"
}

variable "instance_count" {
  type        = number
  description = "Number of web servers to deploy"
}

variable "instance_class" {
  type        = string
  description = "Hardware sizing for the compute instance"
}

variable "enable_backups" {
  type        = boolean
  description = "Toggle automated retention backups"
}

# 🌐 Reusable Web Instance Template
resource "aws_instance" "web_nodes" {
  # Dynamically scales the number of machines built!
  count          = var.instance_count 
  ami            = "ami-0c55b159cbfafe1f0"
  instance_type  = var.instance_class

  tags = {
    Name        = "web-server-\${var.environment}-\${count.index}"
    Environment = var.environment
  }
}

# 🗄️ Conditional Backup Storage Bucket
# This resource will ONLY be built if enable_backups is set to true!
resource "aws_s3_bucket" "prod_backup_vault" {
  count  = var.enable_backups ? 1 : 0 
  bucket = "company-global-vault-\${var.environment}"
}
```

---

## 📋 The Environment Definition Files

Instead of running a generic configuration, create specific input maps for your target deployment zones.

#### File 1: `environments/dev.tfvars`
```hcl
environment    = "dev"
instance_count = 1
instance_class = "t2.micro"    # Cheap tier
enable_backups = false         # No backup costs needed for test data
```

#### File 2: `environments/prod.tfvars`
```hcl
environment    = "prod"
instance_count = 3             # Multi-node for high availability
instance_class = "m5.large"    # Enterprise performance tier
enable_backups = true          # Mandatory data retention active
```

---

## 🚀 Execution Strategy: Running the Lab

To test and provision your environments without overlapping or overwriting state logs, use explicit variable files:

### 🛠️ Provisioning Development:
1. Initialize the workspace:
   ```bash
   terraform init
   ```
2. Run the deployment execution using the Dev configurations:
   ```bash
   terraform plan -var-file="environments/dev.tfvars"
   terraform apply -var-file="environments/dev.tfvars"
   ```
   *Review the output:* Terraform builds exactly **1 micro server** and **0 backup buckets**.

---

### 🚀 Provisioning Production:
To avoid destroying your Dev environment, point your infrastructure state destination to a unique workspace backend or state path, then deploy:

```bash
# Create and switch to a isolated sandbox state lane
terraform workspace new production

# Deploy the high-availability architecture
terraform plan -var-file="environments/prod.tfvars"
terraform apply -var-file="environments/prod.tfvars"
```
   *Review the output:* Terraform builds **3 premium servers** and creates the **S3 production backup vault**.
