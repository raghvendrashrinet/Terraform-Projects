### Options for Drift Handling
  - `terraform plan` (The Drift Detector)
  - `terraform refresh` (Drift adopter -State Updater Only) : standalone use is mostly deprecated;

##### terraform refresh (Adopt Drift)
 * `Updates the local state file to match the real cloud resources`.
 * Does not change infrastructure — it `only syncs state`

 * Useful if you want Terraform to `“adopt” `the drift and continue from the current reality.
> [!NOTE]
> "Adopting drift" means accepting changes made to your live infrastructure outside of Terraform
> (eg manual CLI command) and updating your Terraform code or state to match those real-world modifications,
>  rather than reverting the live infrastructure back to your original code


##### terraform plan (Drift Detector)

 * Compares desired configuration vs. refreshed state.

 * Shows what changes Terraform would make to bring resources back in line.

 * Preferred if you want to revert drift and enforce IaC as the source of truth.
```
[terraform plan]
       │
       ▼
1. Refresh State (Fetch real-world attributes via Cloud APIs)
       │
       ▼
2. Parse Configuration Files (Read local .tf files & variables)
       │
       ▼
3. Build Dependency Graph (Determine resource creation/evaluation order)
       │
       ▼
4. Compare Reality vs. Code (Calculate drift and required actions)
       │
       ▼
5. Output Execution Plan (Display +, ~, - markers or output file)
```

##### terraform apply

 * Actually reconciles drift by updating resources to match configuration.

 * Risk: overwrites manual changes in the cloud.

 * Best when IaC must remain authoritative and manual edits are not allowed.

--- 
## 🛠️ The "Adopt Drift" Workflow
If you want to adopt the drift (keep the manual changes), follow these exact steps:
 - Run a Plan  
     - Run terraform plan.
         Look at the output to see what manual changes were detected.
     - Update Your CodeManually
        edit your .tf configuration files.Copy the exact values that were changed in the live cloud into your code.
     - Verify and Lock It InRun terraform plan again.Your goal is to see the message: "No changes. Your infrastructure matches the configuration."Run terraform apply to safely lock the state and code together

  ---
  ##  manually change a cloud resource in the portal and then run terraform plan.
  ### 🪜 Step‑by‑Step Example
##### 1. Manual change in portal
Suppose Terraform config says:

```hcl
resource "azurerm_storage_account" "example" {
  name                     = "tfstorageacct"
  resource_group_name      = "tf-rg"
  account_tier             = "Standard"
  account_replication_type = "LRS"
}
```
But in the Azure Portal, you manually change replication type from LRS → GRS.

##### 2. Run terraform plan
When you execute:

```bash
terraform plan
```
Terraform does the following:

1. **Refresh state** → Queries Azure to see the actual replication type (GRS).

2. **Compare config vs actual** → Config says LRS, cloud says GRS.

3. **Generate plan → Shows a diff**:

```
~ resource "azurerm_storage_account" "example" {
      account_replication_type: "GRS" => "LRS"
}

```
#### 3. Interpretation
The `~ means `Terraform `detected drift`.

Terraform` proposes to revert the portal change` (set replication back to LRS) if you run `terraform apply`.

Nothing is changed yet — plan is just a preview.

### 4. Next actions
If you want to keep the portal change (adopt drift):

-  Update your .tf config to GRS and re‑run plan.

Then apply will sync state with the new desired config.

- If you want to revert drift (enforce IaC):

Run terraform apply.

Terraform will change the resource back to LRS.

```
                ┌───────────────────────────┐
                │   Drift Detected (plan)   │
                └─────────────┬─────────────┘
                              │
                ┌─────────────┴─────────────┐
                │                           │
        ┌───────▼───────┐           ┌───────▼───────┐
        │ Adopt Drift   │           │ Revert Drift  │
        └───────┬───────┘           └───────┬───────┘
                │                           │
   ┌────────────▼────────────┐   ┌──────────▼──────────┐
   │ Update .tf config to    │   │ Run `terraform apply`│
   │ match portal change     │   │ → Cloud reset to IaC │
   │ (e.g., GRS)             │   │ (e.g., back to LRS) │
   └────────────┬────────────┘   └──────────┬──────────┘
                │                           │
        ┌───────▼───────┐           ┌───────▼───────┐
        │ Run plan again│           │ IaC remains   │
        │ → No drift    │           │ authoritative │
        └───────────────┘           └───────────────┘
```

- **Adopt drift**→ You accept the manual portal change. Update your .tf files, run plan, then apply to sync state.
- **Revert drift**  → You enforce IaC as the source of truth. Run apply and Terraform resets the resource back to what’s defined in code.
