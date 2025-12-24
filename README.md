# Jenkins CI/CD Pipeline with Terraform & Ansible

Complete automated infrastructure deployment pipeline with Splunk installation using Jenkins multibranch pipelines, Terraform, and Ansible.

## 📋 Project Overview

**BYOD-2**: CI/CD Pipeline Setup & Infrastructure Planning (100 Marks)
**BYOD-3**: Deployment, Splunk Configuration & Cleanup (100 Marks)

### BYOD-2 Tasks (Infrastructure Planning)
- ✅ **Task 1 (20pts)**: ngrok tunnel + GitHub webhook auto-triggering
- ✅ **Task 2 (20pts)**: Environment variables & secure credentials
- ✅ **Task 3 (20pts)**: Terraform init + variable file inspection
- ✅ **Task 4 (20pts)**: Branch-specific Terraform planning
- ✅ **Task 5 (20pts)**: Conditional manual approval gate (dev branch only)

### BYOD-3 Tasks (Deployment & Configuration)
- ✅ **Task 1 (20pts)**: Terraform apply with output capture
- ✅ **Task 2 (20pts)**: Dynamic inventory management for Ansible
- ✅ **Task 3 (20pts)**: AWS health status verification
- ✅ **Task 4 (20pts)**: Splunk installation & testing via Ansible
- ✅ **Task 5 (20pts)**: Infrastructure destruction with auto-cleanup

## 🏗️ Infrastructure Components

**Terraform provisions:**
- VPC with public subnet
- EC2 instance (t3.micro)
- Security groups (SSH, HTTP, HTTPS, 8000, 8089)
- Elastic IP
- Internet Gateway and routing

**Ansible configures:**
- Splunk Enterprise 9.1.2
- Automatic service startup
- Web interface on port 8000
- Management interface on port 8089

## 🎯 Pipeline Flow

```
Checkout → Init → Plan → Approval → Apply → Inventory → 
Health Check → Install Splunk → Test → Outputs → Destroy Gate → Destroy
```

## 📁 Project Structure

```
├── Jenkinsfile          # Complete multibranch pipeline
├── main.tf             # Infrastructure code
├── *.tfvars            # Environment configs
├── ansible.cfg         # Ansible configuration
└── playbooks/          # Ansible playbooks
    ├── splunk.yml
    └── test-splunk.yml
```

## 🛠️ Technologies

- Jenkins, Terraform, Ansible, AWS, Splunk, GitHub, ngrok

## 🔄 Branch Strategy

- **dev**: Full automation
- **staging/prod**: Plan-only

## 🌐 Access After Deployment

- **Splunk**: `http://<IP>:8000`
- **User**: `admin` / **Pass**: `SplunkAdmin123!`
