# AAP Setup Guide

## What Gets Built

| Component | Purpose |
|---|---|
| Organization | Logical container for all demo resources |
| Project | Git repo sync — pulls playbooks from this repo |
| Inventories (×2) | `AAP Managed Inventory` for provisioned VMs; `Demo Inventory` (localhost) for utility jobs |
| Custom Credential Type | `ServiceNow ITSM Credential` — injects `SN_HOST`, `SN_USERNAME`, `SN_PASSWORD` as env vars |
| Credentials (×4) | AWS, Machine (Windows/WinRM), ServiceNow ITSM, AAP (self-referential) |
| Execution Environments (×3) | `amazon_aws_ee`, `windows workshop execution environment`, `Default execution environment` |
| Job Templates (×14) | One per workflow node plus cleanup |
| Workflow Job Template | `Run workflow for the Windows Daily Demo (Editable)` — orchestrates all nodes |
| AAP API Token | Long-lived bearer token for ServiceNow to call the AAP launch endpoint |

---

## 1. Organization

Use an existing org or create one: **Resources → Organizations → Add**.

Name it something like `IT Service Automation`. All resources below go into this org.

---

## 2. Execution Environments

Three EEs are needed. Pull them before creating job templates.

**Resources → Execution Environments → Add** for each:

| Name | Image |
|---|---|
| `amazon_aws_ee` | `quay.io/zigfreed/amazon_aws_ee:latest` |
| `windows workshop execution environment` | `quay.io/acme_corp/windows_ee:latest` |
| `Default execution environment` | `registry.redhat.io/ansible-automation-platform-26/ee-supported-rhel9` (or your platform default) |

> `Default execution environment` is typically pre-installed. `amazon_aws_ee` contains the `amazon.aws` collection. `windows workshop execution environment` contains `ansible.windows` and WinRM support.

---

## 3. Project

**Resources → Projects → Add**

| Field | Value |
|---|---|
| Name | `aap.dailydemo.windows.editable` |
| Organization | _(your org)_ |
| Source Control Type | Git |
| Source Control URL | `https://github.com/jcfaith/aap.dailydemo.windows.editable.git` |
| Branch | `main` |

Save → **Sync**. Wait for status to show `Successful` before creating job templates.

---

## 4. Inventories

### AAP Managed Inventory

**Resources → Inventories → Add → Add inventory**

| Field | Value |
|---|---|
| Name | `AAP Managed Inventory` |
| Organization | _(your org)_ |

No hosts needed at creation time — the `DDW - Inventory Update` job adds them dynamically when a VM is provisioned.

### Demo Inventory

**Resources → Inventories → Add → Add inventory**

| Field | Value |
|---|---|
| Name | `Demo Inventory` |
| Organization | _(your org)_ |

After saving, go to the **Hosts** tab → **Add** and add a host named `localhost` with these host variables:

```yaml
ansible_connection: local
```

This inventory is used only by the cleanup job (which runs entirely on the controller, not on any remote host).

---

## 5. Custom Credential Type — ServiceNow ITSM Credential

**Administration → Credential Types → Add**

| Field | Value |
|---|---|
| Name | `ServiceNow ITSM Credential` |

**Input Configuration:**
```yaml
fields:
  - id: instance
    type: string
    label: Instance
  - id: username
    type: string
    label: username
  - id: password
    type: string
    label: password
    secret: true
required:
  - instance
  - username
  - password
```

**Injector Configuration:**
```yaml
env:
  SN_HOST: '{{instance}}'
  SN_USERNAME: '{{username}}'
  SN_PASSWORD: '{{password}}'
```

Save.

---

## 6. Credentials

**Resources → Credentials → Add** for each:

### AWS

| Field | Value |
|---|---|
| Name | `AWS` |
| Credential Type | Amazon Web Services |
| Access Key | _(your AWS access key ID)_ |
| Secret Key | _(your AWS secret access key)_ |

### Machine (Windows/WinRM)

| Field | Value |
|---|---|
| Name | `Daily Demo Windows` |
| Credential Type | Machine |
| Username | `Administrator` (or the Windows user created by `DDW - Provision Access`) |
| Password | _(Windows account password)_ |

> WinRM connection is used — not SSH. The playbooks set `ansible_connection: winrm` in inventory variables. No SSH key needed here.

### ServiceNow ITSM

| Field | Value |
|---|---|
| Name | `ServiceNow ITSM Credential` |
| Credential Type | ServiceNow ITSM Credential _(the custom type created above)_ |
| Instance | `https://<your-instance>.service-now.com` |
| username | _(ServiceNow username with `itil` role)_ |
| password | _(ServiceNow password)_ |

### AAP (self-referential)

Used by `DDW - Inventory Update` and the cleanup job to call the AAP API.

| Field | Value |
|---|---|
| Name | `AAP Credential` |
| Credential Type | Red Hat Ansible Automation Platform |
| Red Hat Ansible Automation Platform | `https://<your-aap-host>` |
| Username | `admin` |
| Password | _(AAP admin password)_ |

> This injects `CONTROLLER_HOST`, `CONTROLLER_USERNAME`, `CONTROLLER_PASSWORD` as env vars — used by `ansible.builtin.uri` calls in the inventory update and cleanup playbooks.

---

## 7. Job Templates

**Resources → Templates → Add → Add job template** for each row below.

All templates use **Organization** = your org and **Project** = `aap.dailydemo.windows.editable`.

### Workflow node templates

| Name | Playbook | Inventory | Execution Environment | Credentials |
|---|---|---|---|---|
| `DDW - Get Requested Item` | `playbooks/servicenow/get_requested_item.yml` | AAP Managed Inventory | Default execution environment | ServiceNow ITSM Credential |
| `DDW - Network Create` | `playbooks/01_create_vpc.yml` | AAP Managed Inventory | amazon_aws_ee | AWS, ServiceNow ITSM Credential |
| `DDW - VM Create` | `playbooks/02_create_instance.yml` | AAP Managed Inventory | amazon_aws_ee | AWS, Daily Demo Windows |
| `DDW - Inventory Update` | `playbooks/03_create_inventory.yml` | AAP Managed Inventory | amazon_aws_ee | AAP Credential, AWS |
| `DDW - Get Instance Info` | `playbooks/04_get_instance_info.yml` | AAP Managed Inventory | amazon_aws_ee | AWS |
| `DDW - Powershell Improvement` | `playbooks/05_powershell_improve.yml` | AAP Managed Inventory | windows workshop execution environment | Daily Demo Windows |
| `DDW - Website Setup` | `playbooks/06_website_setup.yml` | AAP Managed Inventory | Default execution environment | Daily Demo Windows |
| `DDW - Provision Access` | `playbooks/06_windows_account_create.yml` | AAP Managed Inventory | Default execution environment | Daily Demo Windows |
| `DDW - Patching` | `playbooks/07_windows_patching.yml` | AAP Managed Inventory | windows workshop execution environment | Daily Demo Windows |
| `DDW - Create a CMDB record` | `playbooks/servicenow/create_ci.yml` | AAP Managed Inventory | Default execution environment | ServiceNow ITSM Credential |
| `DDW - Create CMDB relationship` | `playbooks/servicenow/create_cmdb_relationship.yml` | AAP Managed Inventory | Default execution environment | ServiceNow ITSM Credential |
| `DDW - Create Incident` | `playbooks/servicenow/incident_create.yml` | AAP Managed Inventory | Default execution environment | ServiceNow ITSM Credential |
| `DDW - Update request ticket - success` | `playbooks/servicenow/update_sn_req_itm.yml` | AAP Managed Inventory | Default execution environment | ServiceNow ITSM Credential |
| `DDW - Update request ticket - failure` | `playbooks/servicenow/update_sn_req_itm.yml` | AAP Managed Inventory | Default execution environment | ServiceNow ITSM Credential |

### Cleanup template

| Name | Playbook | Inventory | Execution Environment | Credentials |
|---|---|---|---|---|
| `DDW - Cleanup All Demo Instances` | `playbooks/cleanup_all_demo_instances.yml` | **Demo Inventory** | amazon_aws_ee | AAP Credential, AWS |

> **Cleanup must use Demo Inventory (localhost), not AAP Managed Inventory.** If it runs against AAP Managed Inventory it holds a lock on that inventory while deleting hosts from it, causing HTTP 409 errors on the host deletion API calls.

---

## 8. Workflow Job Template

**Resources → Templates → Add → Add workflow job template**

| Field | Value |
|---|---|
| Name | `Run workflow for the Windows Daily Demo (Editable)` |
| Organization | _(your org)_ |
| Description | `Edit-before-approval: reads SNOW RITM variables first, then provisions Windows VM` |
| Extra Variables | _(leave blank — `ticket_number` is passed at launch time by ServiceNow)_ |

Save, then click **Visualizer** to build the node graph:

```
DDW - Get Requested Item
  └─ (success) DDW - Network Create
       └─ (success) DDW - VM Create
            ├─ (success) DDW - Inventory Update
            │    └─ (success) DDW - Get Instance Info
            │         ├─ (success) DDW - Create a CMDB record
            │         │    └─ (success) DDW - Create CMDB relationship
            │         └─ (success) DDW - Powershell Improvement
            │              ├─ (success) DDW - Website Setup
            │              │    └─ (success) DDW - Patching
            │              │         └─ (always) DDW - Update request ticket - success
            │              └─ (success) DDW - Provision Access
            │                   └─ (success) DDW - Patching (same node)
            └─ (failure) DDW - Create Incident
                 └─ (always) DDW - Update request ticket - failure
```

> In the visualizer, click a node's **+** to add a downstream node. Select **On Success**, **On Failure**, or **Always** for each connection. `DDW - Patching` → `DDW - Update request ticket - success` uses **Always** so the ticket is closed even if patching partially succeeds.

---

## 9. AAP API Token (for ServiceNow)

ServiceNow's Business Rule calls the AAP workflow launch endpoint using a bearer token. Generate one:

```
POST https://<aap-host>/api/controller/v2/tokens/
Authorization: Basic <base64 admin:password>
Content-Type: application/json

{"description": "ServiceNow integration token", "application": null, "scope": "write"}
```

Copy the `token` value from the response. Paste it into the ServiceNow Business Rule (`AAP - Update RITM on Approval`) in the `Authorization: Bearer <token>` header line.

> The token does not expire by default — it is valid indefinitely unless manually revoked or the AAP instance is rebuilt.

---

## 10. Verify the Setup

1. Sync the project — status must be `Successful`
2. Launch `DDW - Get Requested Item` manually with `ticket_number` set to a real RITM — confirm it reads catalog variables from ServiceNow
3. Submit a catalog item in ServiceNow, approve it, and confirm the workflow job `Run workflow for the Windows Daily Demo (Editable)` starts automatically
4. Watch the workflow visualizer in real time as nodes turn green
5. When complete, confirm the RITM in ServiceNow is closed with hostname, IP, and AMI ID in the work notes
