# ServiceNow Catalog Item Setup

## Catalog Item: Windows VM Provisioning

### Overview

This catalog item submits a ServiceNow RITM that drives the AAP workflow.  
The key feature is **edit-before-approval**: after submission, an approver can modify any catalog variable on the RITM before approving it. AAP reads the final saved values at runtime — so correcting a wrong region, instance type, or VM name before approval means the provisioning reflects those corrections automatically.

---

### Catalog Item Variables

Create the following variables on the catalog item in this order:

| Order | Label | Name | Type | Mandatory | Options / Notes |
|-------|-------|------|------|-----------|-----------------|
| 1 | Datacenter | `datacenter` | Select Box | Yes | See choices below |
| 2 | VM Name | `vm_name` | Single Line Text | Yes | e.g. `windows-demo-prod` |
| 3 | Instance Type | `instance_type` | Select Box | Yes | See choices below |
| 4 | Windows Version | `windows_version` | Select Box | Yes | `2022`, `2019` |
| 5 | Environment | `environment` | Select Box | Yes | See choices below |
| 6 | Contact Email | `contact_email` | Single Line Text | Yes | Requester's email — applied as EC2 tag |
| 7 | Include Website Setup | `include_website` | Checkbox | No | Default: `yes` |

---

### Variable Choices

#### Datacenter (`datacenter`)
| Display Label | Value |
|---|---|
| US East - N. Virginia | `us-east-1` |
| US West - Oregon | `us-west-2` |
| Europe - Ireland | `eu-west-1` |
| Asia Pacific - Singapore | `ap-southeast-1` |

#### Instance Type (`instance_type`)
| Display Label | Value |
|---|---|
| Small (t3.medium - 2 vCPU / 4 GB) | `t3.medium` |
| Medium (t3.large - 2 vCPU / 8 GB) | `t3.large` |
| Large (m5.large - 2 vCPU / 8 GB) | `m5.large` |
| XLarge (m5.xlarge - 4 vCPU / 16 GB) | `m5.xlarge` |

#### Environment (`environment`)
| Display Label | Value |
|---|---|
| Daily Demo | `windows-dailydemo` |
| Development | `dev` |
| Production | `prod` |

---

### Approval Workflow Configuration

1. On the catalog item, enable **Approval** and assign an approver group.
2. Set approval to trigger **before** the fulfillment flow that calls AAP.
3. Grant approvers **write access** to the RITM variables — this is what enables the edit-before-approval pattern.
4. After the approver edits and approves the RITM, the fulfillment flow calls AAP via webhook or REST message with `ticket_number` as the payload.

The AAP workflow node `DDW - Get Requested Item` reads the RITM at that point and uses whatever values the approver saved.

---

### AAP Workflow Trigger (Flow Designer)

In SNOW Flow Designer, add an action after approval that calls the AAP workflow launch REST endpoint:

```
POST https://<aap-host>/api/v2/workflow_job_templates/<id>/launch/
Content-Type: application/json

{
  "extra_vars": {
    "ticket_number": "${ritm_number}"
  }
}
```

Use an AAP credential stored in SNOW Connection & Credential Aliases for authentication.

---

### How the Edit-Before-Approval Pattern Works

```
User submits catalog item
        │
        ▼
RITM created in SNOW with catalog variables
        │
        ▼
Approval notification sent to approver
        │
        ▼
Approver reviews RITM — can EDIT any variable
(wrong region? wrong instance type? fix it here)
        │
        ▼
Approver clicks Approve
        │
        ▼
SNOW Flow triggers AAP workflow with ticket_number
        │
        ▼
DDW - Get Requested Item reads RITM variables
(picks up whatever the approver saved)
        │
        ▼
vm_region, vm_name, vm_instance_type, vm_image etc.
passed via set_stats to all downstream nodes
        │
        ▼
VM provisioned in correct datacenter with correct specs
        │
        ▼
RITM closed with provisioning details
```
