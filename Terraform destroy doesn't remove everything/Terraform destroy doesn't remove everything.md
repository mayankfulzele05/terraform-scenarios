# Lab: Troubleshooting Incomplete "terraform destroy" Executions

## 🚨 The Situation
You run `terraform destroy` to tear down your staging environment. The command finishes, but errors pop up stating that certain resources could not be removed. Alternatively, you check the AWS console and notice a VPC or a Database is still lingering behind.

This lab teaches you how to break deletion deadlocks and cleanly wipe your cloud footprint.

---

## 🛠️ Scenario 1: The Database Snapshot Lock

### The Mistake
You try to destroy an RDS Database instance using this code:

```hcl
resource "aws_db_instance" "app_db" {
  allocated_storage = 20
  engine            = "postgres"
  instance_class    = "db.t3.micro"
  db_name           = "appdb"
  username          = "admin"
  password          = "Password123!"

  # If this is omitted or set to false, destroy WILL fail or stall!
  # skip_final_snapshot = false 
}
```
When you run `terraform destroy`, AWS refuses to drop the database because it wants to take a final storage backup first, which can cause timeout failures or leave the database running if configuration parameters aren't explicitly declared.

### How to Fix It
Update your configuration file to explicitly bypass the final backup snapshot requirement before executing the teardown:

```hcl
resource "aws_db_instance" "app_db" {
  # ... other settings ...
  
  # Set this to true to allow immediate, unhindered deletion
  skip_final_snapshot = true 
}
```
Run `terraform apply` to save this setting, and then run `terraform destroy`.

---

## 📋 Scenario 2: The Foreign Dependency Trap (VPC Deletion Fails)

### The Error
`Error: Sep 09, 2026: VPCResourceInUse: The VPC contains one or more instances or network interfaces and cannot be deleted.`

### Why it happens
You used Terraform to build a Virtual Private Cloud (VPC) and a subnet. Later, someone manually logged into the AWS console or ran a Kubernetes service that spawned an Elastic Network Interface (ENI) inside that subnet. 

Because Terraform didn't create that ENI, it doesn't know it exists. When you run `terraform destroy`, Terraform tries to delete the VPC. AWS blocks it because you cannot delete a VPC that has active network items inside it.

### How to Fix It
1. **Locate the Rogue Asset:** Log into your AWS Console and search for Network Interfaces (ENIs) or Elastic IPs attached to the subnets of the target VPC.
2. **Manual Cleanup:** Manually detach and delete the external resources (e.g., terminate the manual EC2 instance or delete the manual load balancer).
3. **Re-run Destroy:** Once the hidden dependencies are wiped from the cloud, return to your terminal and run the clean command:
   ```bash
   terraform destroy
   ```

---

## 📋 Scenario 3: Force Clearing Residual State (The Nuclear Option)

If a resource has already been physically deleted from AWS by you or an admin, but Terraform's brain is stuck thinking it still exists and keeps failing during the destroy phase, you can manually slice it out of your logbook.

Run the state remove command to tell Terraform to just forget about it:
```bash
terraform state rm <resource_type>.<resource_name>
```

*Example:*
```bash
terraform state rm aws_db_instance.app_db
```
**Result:** Terraform removes the object from its tracker file. The next time you run `terraform plan` or `terraform destroy`, it will completely ignore that resource.
