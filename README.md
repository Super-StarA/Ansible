# Ansible
This directory contains the Ansible configuration for Logstash. Terraform packages this folder, uploads it to a shared S3 bucket, and triggers AWS Systems Manager (SSM) to apply the playbook directly on the Logstash EC2 instance.
