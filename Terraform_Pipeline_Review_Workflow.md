## 🚀 Terraform + Azure Pipeline Review Workflow
#### 1. Developer Stage
Generate plan artifact

```bash
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json
```
or if actual format in terrform plan output format you need to see
```
terraform show tfplan
```
Push both tfplan and tfplan.json into the Pull Request (PR).

Example resource:

```hcl
resource "azurerm_storage_account" "example" {
  name                     = "tfstorageacct"
  resource_group_name      = "tf-rg"
  account_tier             = "Standard"
  account_replication_type = "LRS"
}
```
#### 2. Automated Scans Stage
* Security scans
 - `tfsec` → flags open NSG rules or unencrypted disks.
 - `Checkov` → warns about missing Azure Storage encryption or public blob access.
 - `Trivy` → checks container images in AKS deployments.

* Compliance scans
  - `Conftest / OPA` → enforce Azure policies.

  - Example: “Production VMs must use Standard_D2s_v3 size.”

  - Example: “No deletions of production resource groups allowed.”
  >[!NOTE]
  >Azure Policy and IaC Policy-as-Code (OPA/Checkov) they operate at completely different stages of the development lifecycle.
  >Shift-Left (IaC Policies in Pipeline)
  > Catches 90% of policy violations in the PR stage.
  > Guardrails (Azure Policy in Cloud)
  EG : policy checks that every created resource has a required CostCenter tag:
```
  package main

# Deny creation if CostCenter tag is missing
deny[msg] {
    resource := input.resource_changes[_]
    resource.change.actions[_] == "create"
    not resource.change.after.tags.CostCenter
    
    msg := sprintf("Resource '%v' violates compliance: Missing required 'CostCenter' tag.", [resource.address])
}
```
 Execution & Automated Guardrails
 The policy scanner runs against the JSON plan output:
```
   conftest test tfplan.json --policy ./policies/
```
early detection and immediate action , Since in case of failure you need to address immediatley


**Reports are attached to PR.**

#### 3. Reviewer Stage
Review raw plan output (human‑readable diff):

```Code
~ azurerm_storage_account.example
      account_replication_type: "GRS" => "LRS"
+ azurerm_network_security_group.new
      rule: allow 0.0.0.0/0
```
Interpretation:
```
~ → drift detected (portal change from LRS → GRS).

+ → new NSG rule with unsafe ingress.
```
#### 4. Reviewer Checks Scan Results
  - `Checkov`: “Azure Storage must enforce HTTPS traffic.”
  - `tfsec`: “NSG allows 0.0.0.0/0 ingress — unsafe.”
  - `Conftest`: “Deletion of production RG detected — blocked by policy.”

#### 5. Reviewer Decision
Comment in PR:
- “Replication drift detected — must decide adopt vs revert.”
- “NSG ingress unsafe — restrict to internal CIDR.”
- “Policy violation: production RG deletion not allowed.”

> Reject approval until fixes are made.

#### 6. Approver Stage
Developer fixes Terraform code (e.g., restrict NSG, enforce HTTPS, remove RG deletion).

- Pipeline re‑runs → automated scans pass.
- Plan diff looks safe.
- Approver signs off in CI/CD approval gate.

#### 7. Apply Stage
Pipeline executes:

```bash
terraform apply tfplan
```
Only the reviewed plan is applied → reproducibility + audit trail.

✅ Summary
 - Developer → generates plan + JSON, raises PR.
 - Automated tools → scan JSON for Azure security/compliance.
 - Reviewer → inspects raw plan diff + scan reports.

Approver → final sign‑off before apply.

Apply → only the reviewed plan is executed.

---
###  swimlane diagram 
```
# 🏊 Swimlane Diagram: Terraform + Azure Review Workflow

| Developer                          | Security Tools / Compliance         | Approver / Ops Lead                |
|------------------------------------|-------------------------------------|------------------------------------|
| `terraform plan -out=tfplan`       |                                     |                                    |
| `terraform show -json tfplan.json` |                                     |                                    |
| Push artifacts into PR             |                                     |                                    |
|                                    | Run **tfsec** → NSG misconfig check |                                    |
|                                    | Run **Checkov** → Storage encryption|                                    |
|                                    | Run **Trivy** → AKS image scan      |                                    |
|                                    | Run **Conftest/OPA** → policy rules |                                    |
|                                    | Reports attached to PR              |                                    |
| Review raw plan diff (`+`, `~`, `-`)|                                     |                                    |
| Identify drift (e.g., LRS→GRS)     |                                     |                                    |
| Identify unsafe NSG ingress        |                                     |                                    |
| Cross‑check scan reports           |                                     |                                    |
|                                    |                                     | Review plan + scan results         |
|                                    |                                     | Approve or reject changes          |
|                                    |                                     | If approved → `terraform apply`    |

```

* Developer lane → Generates plan + JSON, raises PR.
* Security Tools lane → Automated scans consume tfplan.json and enforce Azure rules.
* Approver lane → Reviews both the human‑readable plan diff and scan reports before sign‑off.

This swimlane view makes it clear:
 - `JSON (tfplan.json)` is for tools.
 - `Raw plan diff` is for humans.
 - Approver ensures both automation and human oversight are satisfied before terraform apply.
