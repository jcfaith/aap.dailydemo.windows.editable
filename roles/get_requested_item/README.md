# get_requested_item

Reads a ServiceNow RITM's catalog item variables and shares them as workflow-level facts via `set_stats`. This enables the **edit-before-approval** pattern: an approver edits the RITM in SNOW before approving, and AAP reads the saved values at runtime.

## Catalog Item Variables Expected

| SNOW Variable Name | Type | Description | Example Values |
|---|---|---|---|
| `datacenter` | Select Box | AWS region to deploy into | `us-east-1`, `us-west-2`, `eu-west-1`, `ap-southeast-1` |
| `vm_name` | Single Line Text | Name for the EC2 instance | `windows-demo-prod` |
| `instance_type` | Select Box | EC2 instance size | `t3.medium`, `t3.large`, `m5.large`, `m5.xlarge` |
| `windows_version` | Select Box | Windows Server version | `2022`, `2019` |
| `environment` | Select Box | Environment tag applied to EC2 | `windows-dailydemo`, `dev`, `prod` |
| `contact_email` | Single Line Text | Owner email applied as EC2 tag | `user@example.com` |
| `include_website` | Checkbox | Whether to run website setup step | `yes`, `no` |

## Variables Set Downstream (via set_stats)

| Variable | Source |
|---|---|
| `vm_region` | `datacenter` catalog variable |
| `vm_name` | `vm_name` catalog variable |
| `vm_instance_type` | `instance_type` catalog variable |
| `vm_environment_tag` | `environment` catalog variable |
| `vm_image` | Looked up from `get_requested_item_ami_map` using `windows_version` + `datacenter` |
| `vm_my_email_address` | `contact_email` catalog variable |
| `include_website_setup` | `include_website` catalog variable |

## Required Variables

```yaml
ticket_number: ''   # Set as extra_var on workflow template
```

## Required Credentials

- ServiceNow ITSM Credential (custom credential type with SN_HOST, SN_USERNAME, SN_PASSWORD)

## AMI Map

Update `defaults/main.yml` → `get_requested_item_ami_map` with current AMI IDs from your AWS account. AMIs are region-specific and change with AWS patch releases.
