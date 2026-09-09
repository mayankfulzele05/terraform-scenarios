# Lab: Recovering from Missing or Corrupted Terraform State

## 🚨 The Situation
You try to run `terraform plan`, but you are hit with a catastrophic error:
*   *Scenario A:* `Error: Failed to load state: State file is not valid JSON` (Corrupted)
*   *Scenario B:* Terraform shows it wants to **Create** dozens of resources that you know are already live and running in production (Missing State).

This lab teaches you how to roll back to a healthy state backup or reconstruct a lost state file safely.

---

## 🛠️ The Recovery Strategies

### Strategy 1: The Safety Net Backup (.tfstate.backup)
Whenever Terraform modifies your state file locally, it automatically saves a snapshot of the *previous* successful state right next to it, named `terraform.tfstate.backup`.

If your current `terraform.tfstate` file gets corrupted or broken:
1. Make a copy of the broken file just in case: `cp terraform.tfstate terraform.tfstate.broken`
2. Replace the broken file with your backup:
   ```bash
   mv terraform.tfstate.backup terraform.tfstate
   ```
3. Run `terraform plan` to verify that Terraform has regained its memory.

---

### Strategy 2: Restoring from a Remote Backend (Production Standard)
In real-world production environments, you should **never** keep your state file on your local computer. It should be stored in a remote backend like **AWS S3, Azure Blob Storage, or Terraform Cloud**, which natively support **State Versioning**.

If your remote state file is corrupted or accidentally deleted:
1. Log into your cloud console (e.g., AWS S3 bucket).
2. Go to the object history/versions of your `terraform.tfstate` file.
3. Download the last known healthy version.
4. Restore/Overwrite the current object with that healthy version.

---

### Strategy 3: The Disaster Recovery (Reconstruct from Scratch)
If you lost your state file completely, have no backups, and have no remote versioning turned on, you have to reconstruct the logbook manually so Terraform doesn't try to double-provision your infrastructure.

1. **Run a targeted plan** to see what Terraform thinks is missing.
2. Use the **Modern Import Block** feature (introduced in Terraform 1.5+) to pull the existing resources back into a fresh state file without altering the cloud.

Add this to your code for every primary resource you own:
```hcl
import {
  id = "i-0123456789abcdef0" # The real AWS Instance ID running in production
  to = aws_instance.web_server
}
```

3. Run the generation/import command:
   ```bash
   terraform plan
   ```
   Terraform will read the real cloud resource and map it back into a brand new `terraform.tfstate` file automatically. Once successful, run `terraform apply` once to lock it in, then delete the `import` blocks.
