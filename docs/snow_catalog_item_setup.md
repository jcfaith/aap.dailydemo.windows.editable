# ServiceNow Setup Guide

## What Gets Built

| Component | Table | Purpose |
|---|---|---|
| Catalog Item | `sc_cat_item` | The 7-variable request form users fill out |
| Script Include | `sys_script_include` | `SaveRitmVariables` — reads and saves RITM variables from client scripts |
| Business Rule | `sys_script` | Fires on `sysapproval_approver` approval → calls AAP REST API |
| UI Action | `sys_ui_action` | "Approve" button on the RITM form — saves edits and approves in one click |

---

## 1. Catalog Item

Create a new catalog item in **Service Catalog → Maintain Items**.

- **Name:** Windows VM Provisioning
- **Category:** Your choice
- **Approval:** Set to require approval before fulfillment

### Variables (create in this order)

| Order | Label | Name | Type | Mandatory |
|---|---|---|---|---|
| 1 | Datacenter | `datacenter` | Select Box | Yes |
| 2 | VM Name | `vm_name` | Single Line Text | Yes |
| 3 | Instance Type | `instance_type` | Select Box | Yes |
| 4 | Windows Version | `windows_version` | Select Box | Yes |
| 5 | Environment | `environment` | Select Box | Yes |
| 6 | Contact Email | `contact_email` | Single Line Text | Yes |
| 7 | Include Website Setup | `include_website` | Checkbox | No |

### Variable Choices

**Datacenter (`datacenter`)**
| Label | Value |
|---|---|
| US East - N. Virginia | `us-east-1` |
| US West - Oregon | `us-west-2` |
| Europe - Ireland | `eu-west-1` |
| Asia Pacific - Singapore | `ap-southeast-1` |

**Instance Type (`instance_type`)**
| Label | Value |
|---|---|
| Small (t3.medium — 2 vCPU / 4 GB) | `t3.medium` |
| Medium (t3.large — 2 vCPU / 8 GB) | `t3.large` |
| Large (m5.large — 2 vCPU / 8 GB) | `m5.large` |
| XLarge (m5.xlarge — 4 vCPU / 16 GB) | `m5.xlarge` |

**Windows Version (`windows_version`)**
| Label | Value |
|---|---|
| Windows Server 2022 | `2022` |
| Windows Server 2019 | `2019` |

**Environment (`environment`)**
| Label | Value |
|---|---|
| Daily Demo | `windows-dailydemo` |
| Development | `dev` |
| Production | `prod` |

---

## 2. Script Include — `SaveRitmVariables`

Navigate to **System Definition → Script Includes → New**.

| Field | Value |
|---|---|
| Name | `SaveRitmVariables` |
| API Name | `global.SaveRitmVariables` |
| Client callable | ✓ |
| Access | Public |
| Active | ✓ |

**Script:**
```javascript
var SaveRitmVariables = Class.create();
SaveRitmVariables.prototype = Object.extendsObject(AbstractAjaxProcessor, {

  getVariables: function() {
    var ritmSysId = this.getParameter("ritm_sys_id");
    var varNames = ["datacenter","windows_version","instance_type","vm_name","environment","contact_email","include_website"];
    var result = {};
    var mtom = new GlideRecord("sc_item_option_mtom");
    mtom.addQuery("request_item", ritmSysId);
    mtom.query();
    while (mtom.next()) {
      var opt = mtom.sc_item_option.getRefRecord();
      var nm = opt.item_option_new.name.toString();
      if (varNames.indexOf(nm) > -1) {
        result[nm] = opt.value.toString().trim();
      }
    }
    return JSON.stringify(result);
  },

  saveVariablesAndApproveByRitm: function() {
    var ritmSysId = this.getParameter("sysparm_ritm_sys_id");
    var varsJson = this.getParameter("sysparm_variables_json");
    try {
      var vars = JSON.parse(varsJson);
      var ritm = new GlideRecord("sc_req_item");
      ritm.addQuery("sys_id", ritmSysId);
      ritm.query();
      if (!ritm.next()) {
        return JSON.stringify({success: false, error: "RITM not found: " + ritmSysId});
      }
      for (var k in vars) { ritm.variables[k] = vars[k]; }
      ritm.update();
      var appr = new GlideRecord("sysapproval_approver");
      appr.addQuery("source_id", ritmSysId);
      appr.addQuery("state", "requested");
      appr.query();
      var count = 0;
      while (appr.next()) {
        appr.state = "approved";
        appr.update();
        count++;
      }
      if (count === 0) {
        return JSON.stringify({success: false, error: "No pending approvals found"});
      }
      return JSON.stringify({success: true});
    } catch(e) {
      return JSON.stringify({success: false, error: String(e)});
    }
  },

  type: "SaveRitmVariables"
});
```

---

## 3. Business Rule — Trigger AAP on Approval

Navigate to **System Definition → Business Rules → New**.

| Field | Value |
|---|---|
| Name | `DDW - Trigger AAP Workflow on Approval` |
| Table | `Approval [sysapproval_approver]` |
| Active | ✓ |
| Advanced | ✓ |
| When | After |
| Insert | ✓ |
| Update | ✓ |
| Filter Condition | State = Approved |

**Script** (replace `<AAP_HOST>`, `<WORKFLOW_JOB_TEMPLATE_ID>`, `<AAP_TOKEN>`):
```javascript
(function executeRule(current, previous) {
    var ritm = new GlideRecord("sc_req_item");
    if (!ritm.get(current.source_id)) { return; }
    if (ritm.cat_item.sys_id.toString() !== "<CATALOG_ITEM_SYS_ID>") { return; }

    var ritmNumber = ritm.getValue("number");
    var endpoint = "https://<AAP_HOST>/api/controller/v2/workflow_job_templates/<WORKFLOW_JOB_TEMPLATE_ID>/launch/";
    var body = {"extra_vars": {"ticket_number": ritmNumber}};

    var req = new sn_ws.RESTMessageV2();
    req.setEndpoint(endpoint);
    req.setHttpMethod("POST");
    req.setRequestHeader("Authorization", "Bearer <AAP_TOKEN>");
    req.setRequestHeader("Content-Type", "application/json");
    req.setRequestBody(JSON.stringify(body));
    req.execute();
})(current, previous);
```

> Store the AAP token in a SNOW credential and retrieve it via `GlideCredentialResolver` rather than hardcoding it in production.

---

## 4. UI Action — "Approve" Button on RITM Form

Navigate to **System Definition → UI Actions → New**.

| Field | Value |
|---|---|
| Name | `Approve` |
| Table | `Requested Item [sc_req_item]` |
| Active | ✓ |
| Client | ✓ |
| Form button | ✓ |
| Isolate script | unchecked |
| Condition | `current.getValue("approval") == "requested" && current.getValue("cat_item") == "<CATALOG_ITEM_SYS_ID>"` |
| onclick | `editVarsRitm(); return false;` |

**Script:**
```javascript
function editVarsRitm() {
  var ritmSysId = g_form.getUniqueValue();
  var varNames = ["vm_name", "datacenter", "windows_version", "instance_type", "environment", "contact_email", "include_website"];
  var updated = {};
  for (var i = 0; i < varNames.length; i++) {
    updated[varNames[i]] = g_form.getValue(varNames[i]) || "";
  }
  var ga = new GlideAjax("SaveRitmVariables");
  ga.addParam("sysparm_name", "saveVariablesAndApproveByRitm");
  ga.addParam("sysparm_ritm_sys_id", ritmSysId);
  ga.addParam("sysparm_variables_json", JSON.stringify(updated));
  ga.getXMLAnswer(function(ans) {
    var res = {};
    try { res = JSON.parse(ans); } catch(e) { g_form.addErrorMessage("Parse error: " + ans); return; }
    if (res.success) {
      g_form.addInfoMessage("Approved! Reloading...");
      setTimeout(function() { window.location.reload(); }, 1500);
    } else {
      g_form.addErrorMessage("Failed: " + (res.error || "Unknown"));
    }
  });
}
```

**How it works:** The approver goes to My Approvals, opens the RITM, changes any variable using the native ServiceNow dropdowns, and clicks **Approve**. The button reads whatever is currently on the form, saves those values to `sc_item_option`, approves all pending approval records, and reloads the page. The Business Rule fires on each approval record update and calls AAP.

---

## 5. Verify the Setup

1. Submit a catalog item request
2. Go to My Approvals → open the pending request
3. Change a variable (e.g., switch the datacenter)
4. Click **Approve**
5. Confirm "Approved! Reloading..." banner appears and the page refreshes
6. In AAP, confirm the workflow launched with the correct `ticket_number`
7. In the `DDW - Get Requested Item` job output, confirm it read the variable you changed
