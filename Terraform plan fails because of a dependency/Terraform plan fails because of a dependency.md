# Lab: Fixing Dependency Failures in Terraform

## 🚨 The Situation
You run `terraform plan` or `terraform apply`, and it crashes with a dependency error:
`Error: Cycle: aws_instance.web depends on aws_security_group.sg; aws_security_group.sg depends on aws_instance.web`
or it fails because a child resource tries to spin up before its parent is ready.

This lab teaches you how to map out resource relationships and break deadlocks.

---

## 📋 Scenario 1: The Deadlock (Cyclic Dependency)

### The Mistake
Look at this broken configuration where a virtual machine (`web`) and a security group (`sg`) try to reference each other at the exact same moment:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_class = "t2.micro"
  
  # The server needs the security group's ID
  vpc_security_group_ids = [aws_security_group.sg.id]
}

resource "aws_security_group" "sg" {
  name = "web-sg"
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    # The security group rule needs the server's private IP!
    cidr_blocks = ["\${aws_instance.web.private_ip}/32"] 
  }
}
```
Terraform throws a **Cycle Error** because neither resource can finish calculating its variables until the other one exists.

### How to Fix It
Break the dependency loop by extracting the overlapping setting into its own independent, standalone resource block (like a security group rule).

```hcl
# 1. Create the instance cleanly
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_class = "t2.micro"
  vpc_security_group_ids = [aws_security_group.sg.id]
}

# 2. Create the empty container group
resource "aws_security_group" "sg" {
  name = "web-sg"
}

# 3. Inject the rule afterward to break the circular logic loop!
resource "aws_security_group_rule" "allow_web" {
  type              = "ingress"
  from_port         = 80
  to_port           = 80
  protocol          = "tcp"
  security_group_id = aws_security_group.sg.id
  cidr_blocks       = ["\${aws_instance.web.private_ip}/32"]
}
```

---

## 📋 Scenario 2: The Hidden Sequence (Explicit Dependency)

### The Mistake
Sometimes, you have an application running inside an EC2 instance that needs to read files from an S3 bucket as soon as it boots up. Terraform doesn't see any direct code links between them, so it tries to build them both at the same exact time. If the server finishes booting before the bucket is ready, the app crashes.

```hcl
resource "aws_s3_bucket" "app_storage" {
  bucket = "my-heavy-app-storage-bucket"
}

resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_class = "t2.micro"
  # No variable linking to the bucket exists here!
}
```

### How to Fix It
Use the **`depends_on`** configuration option to force a strict chronological order. This tells Terraform: *"Do not touch the server until the bucket is 100% complete."*

```hcl
resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_class = "t2.micro"

  # Explicitly forces the S3 bucket to finish building first
  depends_on = [
    aws_s3_bucket.app_storage
  ]
}
```
