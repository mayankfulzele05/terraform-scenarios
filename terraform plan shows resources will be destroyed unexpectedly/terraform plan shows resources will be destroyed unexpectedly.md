# Lab: Debugging Unexpected Resource Destruction in Terraform

## 🚨 The Situation
You run `terraform plan` after making a small change, and your terminal lights up with red text warning you that a critical production resource (like a database) **will be destroyed and recreated**. 

This lab teaches you why this happens and how to fix it safely without causing data loss or downtime.

---

## 🛠️ The Setup Code
Imagine you have this initial configuration for a database (`main.tf`):

```hcl
resource "aws_db_instance" "prod_db" {
  allocated_storage    = 20
  engine               = "postgres"
  engine_version       = "15.4"
  instance_class       = "db.m5.large"
  db_name              = "production_db"
  username             = "db_admin"
  password             = "SuperSecretPassword123!"
  skip_final_snapshot  = true
  identifier           = "prod-db" 
}
```

---

## 📋 Scenario 1: The "Forces Replacement" Trap

### The Mistake
You change the database `identifier` string from `"prod-db"` to `"prod-db-cluster"`. When you run `terraform plan`, it says it must **destroy and recreate** the database.

### Why it happens
Cloud providers (like AWS) do not allow you to change certain core settings on a live resource. Terraform sees that this setting is immutable (unchangeable) and is forced to delete the old resource to create the new one. In the terminal output, you will see `# forces replacement` next to the changed line.

### How to Fix / Protect It
Add a `lifecycle` block inside your resource block to act as a safety guard.

```hcl
resource "aws_db_instance" "prod_db" {
  # ... your other settings ...
  identifier = "prod-db" 

  lifecycle {
    prevent_destroy = true # Stops Terraform from executing any plan that deletes this resource
  }
}
```
*If you absolutely must change the identifier, you would need to set up a new database, migrate the data manually, change the traffic pointer, and only then delete the old one.*

---

## 📋 Scenario 2: The Code Cleanup Ghost

### The Mistake
You decide to clean up your code formatting. You don't change any settings, but you rename the Terraform structural block address from `"prod_db"` to `"prod_database"`:

```hcl
# Before: resource "aws_db_instance" "prod_db"
# After:
resource "aws_db_instance" "prod_database" {
  # ... settings stay exactly the same ...
}
```
When you run `terraform plan`, Terraform says it wants to **destroy 1 resource** and **create 1 resource**.

### Why it happens
Terraform tracks your real infrastructure using the name inside the code code (`aws_db_instance.prod_db`). When you change that name, Terraform thinks you deleted the old configuration entirely and wrote a brand-new one from scratch. It doesn't know it's the same physical database.

### How to Fix It
Add a `moved` block anywhere in your `.tf` files. This explicitly instructs Terraform to update its internal records without touching the actual cloud resource.

```hcl
moved {
  from = aws_db_instance.prod_db
  to   = aws_db_instance.prod_database
}
```

Run `terraform plan` again after adding this block, and you will see:
`Plan: 0 to add, 1 to change, 0 to destroy.` (Success!)
