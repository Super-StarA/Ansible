Logstash Ansible Playbook

# Logstash Ansible Playbook

This directory contains the **Ansible configuration for Logstash**.
Terraform packages this folder, uploads it to a shared S3 bucket, and triggers AWS Systems Manager (SSM) to apply the playbook directly on the Logstash EC2 instance.

---

## 📂 Directory Layout

```
ansible/logstash/
├── configure-logstash.yml     # Playbook for Logstash
├── inventory.ini              # Minimal inventory (always localhost)
├── templates/
│   └── logstash.conf.j2       # Logstash pipeline config
└── README.md                  # This file
```

---

## ➕ Adding a New Logstash Host

Unlike traditional Ansible usage, you **do not edit `inventory.ini`** with private IPs.
Instead:

1. **Provision a new EC2 instance** (via Terraform).
2. **Create an `aws_ssm_association`** resource in Terraform that targets the new instance:

```hcl
resource "aws_ssm_association" "configure_logstash" {
  name = "AWS-ApplyAnsiblePlaybooks"

  targets {
    key    = "InstanceIds"
    values = [module.logstash-ec2.logstash_instance_id]
  }

  parameters = {
    SourceType          = "S3"
    SourceInfo          = jsonencode({ path = local.playbook_sources["logstash"]})
    InstallDependencies = "True"
    PlaybookFile        = "configure-logstash.yml"

    # Inject ES private IP dynamically
    ExtraVariables      = "elasticsearch_host=${module.elasticsearch-ec2.elasticsearch_private_ip}"

    Verbose             = "-v"
  }

  depends_on = [aws_s3_object.logstash_playbook_object]
}
```

This ensures the same playbook runs on the new host automatically.

---

## ▶️ Running the Playbook

* Terraform packages this directory into `logstash-playbook.zip`.
* Uploads it to the **shared Ansible S3 bucket**.
* Associates it with the target EC2 via **SSM**.
* SSM runs the playbook **on the instance itself**, using the following inventory:

```ini
[logstash]
localhost ansible_connection=local
```

---

## 🛠 Injecting Variables (Example)

Runtime values like the Elasticsearch private IP are passed via `ExtraVariables`.

Example (from Terraform):

```hcl
parameters = {
    SourceType          = "S3"
    SourceInfo          = jsonencode({ path = local.playbook_sources["logstash"]})
    InstallDependencies = "True"
    PlaybookFile        = "configure-logstash.yml"

    # Inject ES private IP dynamically
    ExtraVariables      = "elasticsearch_host=${module.elasticsearch-ec2.elasticsearch_private_ip}"
```

This makes the variable available inside the playbook as `{{ elasticsearch_host }}`.

---

## ✅ Workflow Summary

1. Update `templates/logstash.conf.j2` or `configure-logstash.yml` as needed.
2. Run `terraform apply`.
3. Terraform updates the ZIP, pushes it to S3, and re-applies the playbook to the targeted EC2s via SSM.
4. Verify Logstash logs on the instance to confirm the new configuration is active.

---


## 📦 Adding Another Playbook (Example: Beats)

The shared S3 bucket supports multiple playbooks.
To add one:

1. Zip the new playbook directory (e.g. `ansible/filebeat/`) in Terraform.
2. Upload it to the same S3 bucket.
3. Extend `locals.tf` like this:

```hcl
locals {
  playbook_sources = {
    logstash = "https://s3.${var.aws_region}.amazonaws.com/${module.ansible_s3_bucket.bucket_name}/${aws_s3_object.logstash_playbook_object.key}"
    filebeat = "https://s3.${var.aws_region}.amazonaws.com/${module.ansible_s3_bucket.bucket_name}/${aws_s3_object.filebeat_playbook_object.key}"
  }
}
```

4. Reference the new entry when creating an `aws_ssm_association` for Filebeat EC2 instances:

```hcl
parameters = {
  SourceType   = "S3"
  SourceInfo   = jsonencode({ path = local.playbook_sources["filebeat"] })
  PlaybookFile = "configure-filebeat.yml"
}
```

---

## ✅ Workflow Summary

1. Update `templates/logstash.conf.j2` or `configure-logstash.yml` as needed.
2. Run `terraform apply`.
3. Terraform zips the updated directory, uploads it to S3, and refreshes the URL in `local.playbook_sources`.
4. SSM applies the playbook to the targeted EC2 instances.
5. Verify Logstash logs on the instance to confirm the new configuration is active.

---

👉 This `README.md` applies only to **Logstash**.
Other components (like Beats) maintain their own documentation in their respective directories.

 
