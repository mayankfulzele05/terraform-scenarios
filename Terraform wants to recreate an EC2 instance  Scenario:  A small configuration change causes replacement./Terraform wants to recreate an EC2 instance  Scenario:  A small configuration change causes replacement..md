# Lab: Fixing Unexpected EC2 Instance Recreations in Terraform

## 🚨 The Situation
You modify a minor property or update an image ID in your EC2 configuration file. When you run `terraform plan`, you see the dreaded message:
`+/- destroy and then create replacement`
If you apply this blindly, your live virtual machine will be terminated, causing immediate downtime and potential data loss on local volumes.

This lab teaches you how to diagnose the cause and mitigate the risks of EC2 replacements.

---

## 🛠️ The Setup Code
Imagine you have this initial virtual machine configuration (`compute.tf`):

```hcl
resource "aws_instance" "app_server" {
  ami               = "ami-0c55b159cbfafe1f0" # Ubuntu 22.04 LTS
  instance_class    = "t2.micro"
  availability_zone = "us-east-1a"

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, World Version 1" > index.html
              EOF
}
```

---

## 📋 The Troubleshooting & Fix Strategies

### Step 1: Diagnose the "Forces Replacement" Trigger
Run `terraform plan` and look carefully at the output symbols. Look for the exact line containing the text `# forces replacement`.

*   If it points to `ami`: You are trying to upgrade the OS. Replacement is **unavoidable**. Proceed to Step 2.
*   If it points to `user_data`: Replacement **can be avoided**! (See Step 3).

---

### Step 2: The Blue/Green Zero-Downtime Swap (If Replacement is Required)
If you *must* swap the AMI or move the Availability Zone, you can change the order of operations. By default, Terraform destroys the old server first, then builds the new one (causing downtime). You can reverse this using a `lifecycle` block.

Update your resource to include `create_before_destroy`:

```hcl
resource "aws_instance" "app_server" {
  ami               = "ami-0123456789abcdef0" # New upgraded AMI
  instance_class    = "t2.micro"
  availability_zone = "us-east-1a"

  lifecycle {
    # Builds the new server first, hooks it up, and only drops the old one afterward!
    create_before_destroy = true 
  }
}
```
*Note: Ensure your cloud account has enough capacity/quota to run two instances simultaneously during the deployment window.*

---

### Step 3: Stop User Data from Destroying the Server
If you only need to change a startup script layout but don't want to kill the running machine, use the `user_data_replace_on_change` argument.

By setting this flag to `false` (or omitting it depending on your provider defaults), you tell Terraform to ignore structural modifications to the script for existing instances:

```hcl
resource "aws_instance" "app_server" {
  ami            = "ami-0c55b159cbfafe1f0"
  instance_class = "t2.micro"

  # Change the script text safely
  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, World Version 2" > index.html
              EOF

  # Instructs Terraform NOT to destroy the machine just for a script change
  user_data_replace_on_change = false 
}
```
