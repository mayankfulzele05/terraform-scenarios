# Lab: Handling Manual Infrastructure Modifications (Configuration Drift)

## 🚨 The Situation
A developer manually edits an AWS Security Group using the Web Console to open a port for troubleshooting. Later, you run `terraform plan` and notice Terraform wants to modify or revert the resource, even though no one touched the local code files.

This lab teaches you how to detect drift and safely sync your infrastructure back into alignment.

---

## 🛠️ The Initial Code Setup
Imagine your baseline security group configuration looks like this (`security.tf`):

```hcl
resource "aws_security_group" "web_sg" {
  name        = "web-server-sg"
  description = "Managed by Terraform"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## 📋 The Two Recovery Paths

### Option 1: Revert the Rogue Manual Change (Enforce Code Standard)
If the manual change violates company security policy or was just a temporary hotfix, you want to erase it.

1. Run `terraform plan`. 
   Terraform will read the real AWS state, discover the unauthorized port, and display a plan showing it will strip the manual modifications away.
2. Run the enforcement command:
   ```bash
   terraform apply
   ```
3. **Result:** Terraform hits the AWS API and deletes the manual rule. The cloud infrastructure is instantly returned to the secure state defined in your repository.

---

### Option 2: Absorb the Manual Change (Update the Code Base)
If the team agrees that the manual change is a permanent improvement, you must update your code so Terraform accepts it.

1. Run `terraform plan` to see exactly what setting was changed in the console. The output will show a diff like this:
   ```diff
   # aws_security_group.web_sg will be updated in-place
   ~ ingress {
       + from_port   = 22
       + to_port     = 22
       + protocol    = "tcp"
       + cidr_blocks = ["203.0.113.50/32"]
     }
   ```
2. Manually add that identical block directly into your local `security.tf` file:
   ```hcl
   resource "aws_security_group" "web_sg" {
     name        = "web-server-sg"
     description = "Managed by Terraform"

     ingress {
       from_port   = 80
       to_port     = 80
       protocol    = "tcp"
       cidr_blocks = ["0.0.0.0/0"]
     }

     # Added to absorb the manual production change!
     ingress {
       from_port   = 22
       to_port     = 22
       protocol    = "tcp"
       cidr_blocks = ["203.0.113.50/32"] 
     }
   }
   ```
3. Run `terraform plan` again. 
   Terraform will now output: `No changes. Your infrastructure matches the configuration.` 
4. Commit the updated code to your git repository.
