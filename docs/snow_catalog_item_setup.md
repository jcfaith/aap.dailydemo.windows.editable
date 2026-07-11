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
- **Group:** Set to an approval group containing only yourself (e.g. create a new group via System Security → Groups, add yourself as the only member, then select it here). Without this, SNOW falls back to global approval groups and creates approver records for every member.

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
| Test | `windows-dailydemo` |
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
      var varNames = ["datacenter","windows_version","instance_type","vm_name","environment","contact_email","include_website"];

      // Pass 1: read originals
      var originals = {};
      var mtom = new GlideRecord("sc_item_option_mtom");
      mtom.addQuery("request_item", ritmSysId);
      mtom.query();
      while (mtom.next()) {
        var opt = mtom.sc_item_option.getRefRecord();
        var nm = opt.item_option_new.name.toString();
        if (varNames.indexOf(nm) > -1) {
          originals[nm] = opt.value.toString().trim();
        }
      }

      // Pass 2: write approver's values
      var mtom2 = new GlideRecord("sc_item_option_mtom");
      mtom2.addQuery("request_item", ritmSysId);
      mtom2.query();
      while (mtom2.next()) {
        var opt2 = mtom2.sc_item_option.getRefRecord();
        var nm2 = opt2.item_option_new.name.toString();
        if (vars.hasOwnProperty(nm2)) {
          opt2.setValue("value", vars[nm2]);
          opt2.update();
        }
      }

      // Requester info from RITM
      var ritmGr = new GlideRecord("sc_req_item");
      ritmGr.get(ritmSysId);
      var requesterName = ritmGr.requested_for.getDisplayValue();
      var requesterGr = new GlideRecord("sys_user");
      requesterGr.get(ritmGr.getValue("requested_for"));
      var requesterEmail = requesterGr.getValue("email") || "";

      var approverName = gs.getUser().getFullName();
      var approverEmail = "faith@redhat.com";

      // Track what changed
      var changedFields = [];
      var unchangedFields = [];
      for (var i = 0; i < varNames.length; i++) {
        var vn = varNames[i];
        if (originals[vn] !== vars[vn]) {
          changedFields.push({field: vn, from: originals[vn] || "", to: vars[vn] || ""});
        } else {
          unchangedFields.push(vn);
        }
      }

      var note = "Approval Summary\n\n";
      note += "Requester: " + requesterName + " (" + requesterEmail + ")\n";
      note += "Approver: " + approverName + " (" + approverEmail + ")\n";

      note += "\nOriginal request variables:\n";
      for (var a = 0; a < varNames.length; a++) {
        note += "  " + varNames[a] + ": " + (originals[varNames[a]] || "") + "\n";
      }

      note += "\nApproved variables:\n";
      for (var b = 0; b < varNames.length; b++) {
        note += "  " + varNames[b] + ": " + (vars[varNames[b]] || "") + "\n";
      }

      if (changedFields.length > 0) {
        note += "\nVariables changed before approval:\n";
        for (var c = 0; c < changedFields.length; c++) {
          note += "  " + changedFields[c].field + ": [" + changedFields[c].from + "] -> [" + changedFields[c].to + "]\n";
        }
      } else {
        note += "\nNo variables were changed from the original request.\n";
      }

      if (unchangedFields.length > 0) {
        note += "\nUnchanged: " + unchangedFields.join(", ");
      }

      note += "\n\nContact the approver at " + approverEmail + " with any questions.";

      var currentUserId = gs.getUserID();
      var appr = new GlideRecord("sysapproval_approver");
      appr.addQuery("document_id", ritmSysId);
      appr.addQuery("state", "requested");
      appr.addQuery("approver", currentUserId);
      appr.orderByDesc("sys_created_on");
      appr.setLimit(1);
      appr.query();
      if (!appr.next()) {
        return JSON.stringify({success: false, error: "No pending approval found for current user"});
      }
      appr.state = "approved";
      appr.update();

      // Insert journal entry directly — setWorkflow(false) below suppresses
      // journal field processing on the RITM update, so we write here instead.
      var jf = new GlideRecord("sys_journal_field");
      jf.setValue("name", "sc_req_item");
      jf.setValue("element", "work_notes");
      jf.setValue("element_id", ritmSysId);
      jf.setValue("value", note);
      jf.insert();

      var ritm = new GlideRecord("sc_req_item");
      if (ritm.get(ritmSysId)) {
        ritm.setValue("approval", "approved");
        ritm.setWorkflow(false);
        ritm.update();
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

## 3. Outbound REST Message — AAP Windows Demo Launch

Navigate to **System Web Services → Outbound → REST Messages → New**.

| Field | Value |
|---|---|
| Name | `AAP Windows Demo Launch` |
| Endpoint | `https://<AAP_HOST>/api/controller/v2/` |
| Authentication type | No authentication |

Save, then open the **HTTP Methods** tab → **New**.

| Field | Value |
|---|---|
| Name | `launch` |
| Function name | `launch` |
| HTTP method | POST |
| Endpoint | `https://<AAP_HOST>/api/controller/v2/workflow_job_templates/<WORKFLOW_JOB_TEMPLATE_ID>/launch/` |
| Content | `{"extra_vars": {"ticket_number": "${ticket_number}"}}` |

**Leave the HTTP Request Headers tab completely empty.** Headers are set programmatically in the Business Rule. Any header added here will be merged with the code headers on every call — duplicates of Content-Type or Authorization cause HTTP 415 from AAP and silently kill the integration.

Save the function.

---

## 4. Business Rule — Trigger AAP on Approval

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

**Script** (replace `<CATALOG_ITEM_SYS_ID>` and `<AAP_TOKEN>`):
```javascript
(function executeRule(current, previous) {
    var ritmGr = new GlideRecord("sc_req_item");
    if (!ritmGr.get(current.getValue("document_id"))) { return; }
    if (ritmGr.getValue("cat_item") !== "<CATALOG_ITEM_SYS_ID>") { return; }
    try {
        var rm = new sn_ws.RESTMessageV2("AAP Windows Demo Launch", "launch");
        rm.setStringParameterNoEscape("ticket_number", ritmGr.getValue("number"));
        // Set headers in code, not in the REST Message UI.
        // IMPORTANT: leave the REST Message Headers tab empty.
        // Any stored headers merge with these and will cause HTTP 415.
        rm.setRequestHeader("Content-Type", "application/json");
        rm.setRequestHeader("Authorization", "Bearer <AAP_TOKEN>");
        var response = rm.execute();
        gs.log("AAP launched for " + ritmGr.getValue("number") + " | HTTP: " + response.getStatusCode(), "AAPIntegration");
    } catch(e) {
        gs.logError("AAP launch failed for " + ritmGr.getValue("number") + ": " + e.message, "AAPIntegration");
    }
})(current, previous);
```

> **Critical:** The REST Message "AAP Windows Demo Launch" must have **zero headers** in its UI (both at the message level and function level). Headers set via `setRequestHeader()` in the BR are merged with stored headers — duplicates cause HTTP 415. Generate an AAP token via `POST /api/controller/v2/tokens/` and put it here. The token created for this demo expires in 3025.

---

## 5. UI Action — "Approve" Button on RITM Form

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

## 6. Verify the Setup

1. Submit a catalog item request
2. Go to My Approvals → open the pending request
3. Change a variable (e.g., switch the datacenter)
4. Click **Approve**
5. Confirm "Approved! Reloading..." banner appears and the page refreshes
6. In AAP, confirm the workflow launched with the correct `ticket_number`
7. In the `DDW - Get Requested Item` job output, confirm it read the variable you changed
