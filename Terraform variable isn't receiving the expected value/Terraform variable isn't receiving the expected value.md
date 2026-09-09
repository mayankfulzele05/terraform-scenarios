# Lab: Debugging Variable Value Conflicts in Terraform

## 🚨 The Situation
You define an environment variable or edit your `terraform.tfvars` file to change a configuration setting (like changing an instance type from `t2.micro` to `m5.large`). However, when you run `terraform plan`, Terraform completely ignores your change and sticks to an unexpected value.

This lab teaches you how Terraform prioritizes variable inputs and how to fix evaluation issues.

---

## 🛠️ The Disputed Code Setup
Imagine you have this configuration spread across multiple files in your directory:

#### File 1: `variables.tf` (The Definition)
```hcl
variable "instance_type" {
  type        = string
  description = "The sizing of the compute server"
  default     = "t2.micro" # Baseline default value
}
```

#### File 2: `terraform.tfvars` (The Intended Value)
You wrote your production scaling requirement here:
```hcl
instance_type = "m5.large"
```

#### File 3: `secret.auto.tfvars` (The Hidden Overrider)
A teammate left this auto-loading file in the folder weeks ago:
```hcl
instance_type = "t3.medium"
```

If you run `terraform plan` now, the instance type will evaluate to `t3.medium` because `*.auto.tfvars` files silently override standard `terraform.tfvars` files!

---

## 📋 The Troubleshooting & Fix Strategies

### Step 1: Check the Current Evaluation via Output
To see exactly what value Terraform is *actually* processing without digging through an massive plan, add an output block to `outputs.tf`:

```hcl
output "debug_instance_type" {
  value = var.instance_type
}
```
Run `terraform refresh` to print out the active assignment value.

---

### Step 2: Enforce the Correct Value via CLI Flag (Highest Priority)
If you need to bypass all background configuration files immediately to guarantee your value takes effect, use the explicit `-var` argument in your terminal command line:

```bash
terraform plan -var="instance_type=m5.large"
```
**Result:** Because terminal arguments have the absolute highest priority in the hierarchy, this forces Terraform to drop the definitions in `secret.auto.tfvars` and assign `m5.large` instead.

---

### Step 3: Check Your Shell Environment Variables
Sometimes, your shell environment is quietly overriding your code settings via backend variables. Check if you have a variable set in your current terminal session:

**For Linux / macOS:**
```bash
printenv | grep TF_VAR_
```
If you see `TF_VAR_instance_type=t2.micro` listed in the output, your terminal session is hardcoding the values. Clear it out by running:
```bash
unset TF_VAR_instance_type
```

**For Windows (PowerShell):**
```powershell
Get-ChildItem env:TF_VAR_*
# To clear it out:
Remove-Item env:TF_VAR_instance_type
```
---

### Step 4: Validate Types and Formats
If your variable is receiving an empty or broken value, ensure you aren't passing a string where an object or a list is expected. Always use variable validation blocks to catch bad values early:

```hcl
variable "instance_type" {
  type = string
  
  validation {
    condition     = can(regex("^[t|m|c]", var.instance_type))
    error_message = "The instance_type value must begin with a valid instance family letter (t, m, or c)."
  }
}
```
