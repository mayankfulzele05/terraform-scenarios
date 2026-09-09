# Lab: Recovering from a Partially Failed "terraform apply"

## 🚨 The Situation
You run `terraform apply`. Terraform successfully creates your network infrastructure, but while building the virtual machines, the execution crashes with an error:
`Error: monitoring logic failed: ResourceLimitExceeded`

Your terminal stops halfway through. Some resources are live in the cloud, while others were never built.

This lab teaches you how to inspect your partial state and safely resume execution without duplicating assets.

---

## 🛠️ The Interrupted Code Setup
Imagine your configuration script looks like this (`infra.tf`):

```hcl
# Resource 1: Success!
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# Resource 2: Success!
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

# Resource 3: CRASH! (AWS account hits a maximum instance limit)
resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_class = "p4d.24xlarge" # Extremely large instance type that triggers a quota block!
}
```

---

## 📋 The Recovery Roadmap

### Step 1: Inspect What Was Actually Saved
Do not panic or delete code files. First, check your state file to see exactly what components Terraform successfully recorded before the crash:

```bash
terraform state list
```
*Output will look like this:*
```text
aws_vpc.main
aws_subnet.public
```
Notice that `aws_instance.app_server` is missing from the list. It was never written to the cloud.

---

### Step 2: Fix the Root Cause
Modify the broken configuration block in your code to address the error message (e.g., switch to an instance type allowed by your cloud account quotas):

```hcl
resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_class = "t2.micro" # Fixed: Changed to a standard micro instance size
}
```

---

### Step 3: Run a Targeted Verification
Before applying changes globally, run a targeted plan focusing exclusively on the broken piece to make sure your fix works:

```bash
terraform plan -target=aws_instance.app_server
```
Terraform will read the existing VPC and Subnet from its state cache, bypass modifying them, and show that it only needs to build the single missing server element.

---

### Step 4: Resume the Apply Operation
Run the standard deployment command normally:

```bash
terraform apply
```
**Result:** Terraform recognizes that the VPC and Subnet are already complete and functional. It leaves them completely alone and cleanly provisions the `aws_instance.app_server` to complete your deployment graph successfully!
