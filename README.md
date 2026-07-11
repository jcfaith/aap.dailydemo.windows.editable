# AAP + ServiceNow: Windows VM Provisioning Demo

## Overview

This demo shows end-to-end IT automation between Red Hat Ansible Automation Platform (AAP) and ServiceNow. A user submits a **Windows VM Provisioning** request through the ServiceNow Service Catalog. The request routes to an approver who can adjust any variable before approving — wrong region, wrong instance size, wrong VM name, all correctable inline. On approval, AAP automatically provisions a Windows EC2 instance in the correct AWS region, registers it in the CMDB, optionally deploys a web server, applies patches, and closes the ticket with a summary.

---

## Demo Flow

1. **Submit a request** — Go to ServiceNow → Service Catalog → Windows VM Provisioning. Fill in VM name, datacenter (AWS region), instance type, Windows version, environment, and email. Submit.

2. **Approve and optionally edit** — Go to My Approvals. Click the pending request. If any field needs to be corrected (e.g., the requester picked the wrong region), change it directly on the form. Click **Approve**. The button saves whatever values are on the form and approves the request in one click.

3. **AAP runs the workflow** — The workflow launches automatically. Each job node is visible in real time: VPC creation → instance provisioning → inventory registration → website setup → patching → CMDB update → ticket closure.

4. **Review the result** — The RITM in ServiceNow is closed with the instance hostname, IP address, and AMI ID. The web server (if selected) is reachable at the public IP.

---

## How It Works

```
User submits catalog item in SNOW
          │
          ▼
RITM created with catalog variables
          │
          ▼
Approver opens My Approvals → edits any field → clicks Approve
          │
          ▼
SNOW Business Rule fires → calls AAP REST API
          │
          ▼
AAP workflow launches with ticket_number as the only input
          │
          ▼
DDW - Get Requested Item reads the RITM and extracts all variables
(picks up whatever the approver last saved)
          │
          ▼
All downstream nodes receive: region, vm_name, instance_type,
windows_version, environment, contact_email, include_website
          │
          ├── Create VPC + subnet + security group + IGW in target region
          ├── Launch Windows EC2 instance with WinRM enabled via user_data
          ├── Register instance in AAP inventory
          ├── Wait for WinRM (port 5986) to respond
          ├── Improve PowerShell execution environment
          ├── Create Windows user account
          ├── (optional) Deploy IIS website
          ├── Apply Windows patches
          ├── Create/update CMDB CI and relationship
          └── Close RITM with hostname, IP, AMI ID
```

---

## Catalog Item Variables

| Label | Name | Type | Options |
|---|---|---|---|
| Datacenter | `datacenter` | Select Box | US East (us-east-1), US West (us-west-2), EU Ireland (eu-west-1), AP Singapore (ap-southeast-1) |
| VM Name | `vm_name` | Single Line Text | e.g. `customer-demo-vm` |
| Instance Type | `instance_type` | Select Box | t3.medium, t3.large, m5.large, m5.xlarge |
| Windows Version | `windows_version` | Select Box | 2022, 2019 |
| Environment | `environment` | Select Box | windows-dailydemo, dev, prod |
| Contact Email | `contact_email` | Single Line Text | Applied as EC2 Contact tag |
| Include Website Setup | `include_website` | Checkbox | Default: yes |

---

## Cost Management

Run the **Cleanup All Demo Instances** job template in AAP to terminate all demo EC2 instances and tear down all VPC resources across all four supported regions (us-east-1, us-west-2, eu-west-1, ap-southeast-1). Run it as often as needed — it is safe to run with no instances present.

---

## Day 2 Operations

After provisioning, the following job templates are available to demonstrate ongoing management:

- **Audit** — Scans registry entries and repairs drift. Documents results in a CSV.
- **Patching** — Survey-driven Windows patching with optional reboot.
- **.NET Patch Report** — Inventory of installed .NET versions.
- **SMB Server** — Configure a Windows file share.
- **Update IMDSv2** — Harden the instance metadata service.
- **Gather Facts** — Ad hoc fact collection for any inventory host.

---

## AAP Credentials Required

| Credential Type | Used For |
|---|---|
| AWS | EC2, VPC, ENI, IGW operations |
| Machine (Windows) | WinRM connection to provisioned instance |
| ServiceNow | CMDB CI creation, RITM update, incident management |
| Red Hat Ansible Automation Platform | Inventory registration via REST API |

---

## ServiceNow Setup

See [docs/snow_catalog_item_setup.md](docs/snow_catalog_item_setup.md) for the full guide to setting up:
- The catalog item and its 7 variables
- The approval configuration
- The Business Rule that triggers the AAP workflow on approval
- The Script Include (`SaveRitmVariables`) that powers the edit-before-approval button
- The `Approve` UI Action on the RITM form

---

## Automated Incident Management

If any job node in the workflow fails, AAP captures the error message, job ID, and template name, then opens a ServiceNow incident automatically with full context — no manual ticket creation required.

```yaml
- name: Run task with incident handling
  block:
    - name: Your task here
      ...
  rescue:
    - name: Capture error details
      ansible.builtin.set_stats:
        data:
          my_error: "{{ ansible_failed_result.msg }}"
          my_job_id: "{{ tower_job_id }}"
          my_job_template_name: "{{ tower_job_template_name }}"
    - ansible.builtin.fail:
        msg: failing so we create the incident ticket
```

---

## ServiceNow Credential (AAP Custom Credential Type)

Input configuration:
```yaml
fields:
  - id: instance
    type: string
    label: Instance
  - id: username
    type: string
    label: Username
  - id: password
    type: string
    label: Password
    secret: true
required:
  - instance
  - username
  - password
```

Injector configuration:
```yaml
env:
  SN_HOST: '{{instance}}'
  SN_USERNAME: '{{username}}'
  SN_PASSWORD: '{{password}}'
```
