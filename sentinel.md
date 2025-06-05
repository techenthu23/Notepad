```
<#
.SYNOPSIS
Export Microsoft Sentinel automation rules.

.DESCRIPTION
How to export Microsoft Sentinel automation rules at once using PowerShell and REST API.

.NOTES
File Name : Export-AutomationRules.ps1
Author    : Microsoft MVP/MCT - Charbel Nemnom
Version   : 1.0
Date      : 01-May-2024
Updated   : 02-May-2024
Requires  : PowerShell 6.2 or PowerShell 7.x.x (Core)
Module    : Az Module

.LINK
To provide feedback or for further assistance please visit:
https://charbelnemnom.com

.EXAMPLE
.\Export-AutomationRules.ps1 -SubscriptionId <SUB-ID> -ResourceGroup <RG-Name> `
    -WorkspaceName <Log-Analytics-Name> -Status <Enabled> -Verbose
This example will connect to your Azure account using the subscription Id specified, and then export all automation rules to JSON templates.
By default, the automation rules with the state "Enabled" will be exported. Optionally, you can export both Enabled and Disabled automation rules.
#>

param (
    [Parameter(Position = 0, Mandatory = $true, HelpMessage = 'Enter Azure Subscription ID')]
    [string]$subscriptionId,
    [Parameter(Position = 1, Mandatory = $true, HelpMessage = 'Enter Resource Group Name where Microsoft Sentinel is deployed')]
    [string]$resourceGroupName,
    [Parameter(Position = 2, Mandatory = $true, HelpMessage = 'Enter Log Analytics Workspace Name')]
    [string]$workspaceName,
    [Parameter(Position = 3, Mandatory = $true, HelpMessage = 'Select the status of the automation rules to export')]
    [ValidateSet('Enabled', 'Disabled', 'All')]
    [String]$status = 'Enabled'
)

#! Install Az Module If Needed
function Install-Module-If-Needed {
    param([string]$ModuleName)

    if (Get-Module -ListAvailable -Name $ModuleName) {
        Write-Host "Module '$($ModuleName)' already exists, continue..." -ForegroundColor Green
    }
    else {
        Write-Host "Module '$($ModuleName)' does not exist, installing..." -ForegroundColor Yellow
        Install-Module $ModuleName -Force -AllowClobber -ErrorAction Stop
        Write-Host "Module '$($ModuleName)' installed." -ForegroundColor Green
    }
}

function Export-AutomationRules {
    param (
        $subscriptionId,
        $apiVersion,
        $resourceGroupNam,
        $workspaceName
    )

    #! Get Az Access Token
    $token = Get-AzAccessToken #This will default to Azure Resource Manager endpoint
    $authHeader = @{
        'Content-Type'  = 'application/json'
        'Authorization' = 'Bearer ' + $token.Token
    }

    # Define Automation Rules URI
    $ruleURI = "https://management.azure.com/subscriptions/$subscriptionId/resourceGroups/$resourceGroupName/providers/Microsoft.OperationalInsights/workspaces/$workspaceName/providers/Microsoft.SecurityInsights/automationRules$($apiVersion)"
    $ruleResult = Invoke-RestMethod $ruleURI -Method 'GET' -Headers $authHeader       
    return $ruleResult
}

#! Install Az Accounts Module If Needed
Install-Module-If-Needed Az.Accounts

#! Check Azure Connection
Try {
    Write-Verbose "Connecting to Azure Cloud..."
    Connect-AzAccount -ErrorAction Stop | Out-Null
}
Catch {
    Write-Warning "Cannot connect to Azure Cloud. Please check your credentials. Exiting!"
    Break
}

# Define the latest API Version to use for Microsoft Sentinel
$apiVersion = "?api-version=2024-03-01"

If ($status -eq 'Enabled') {   
    Write-Verbose "Getting $($status) Automation Rules..."
    # Get all Enabled Automation Rules  
    $ruleResult = (Export-AutomationRules -apiVersion $apiVersion -subscriptionId $subscriptionId -resourceGroupName $resourceGroupName -workspaceName $workspaceName).value
    $activeRules = $ruleResult | Where-Object { $_.properties.triggeringLogic.isEnabled -eq $true }
    Write-Verbose "Exporting $($status) Automation Rules..." 
}
elseif ($status -eq 'Disabled') {
    Write-Verbose "Getting $($status) Automation Rules..."
    # Get all Disabled Automation Rules    
    $ruleResult = (Export-AutomationRules -apiVersion $apiVersion -subscriptionId $subscriptionId -resourceGroupName $resourceGroupName -workspaceName $workspaceName).value
    $activeRules = $ruleResult | Where-Object { $_.properties.triggeringLogic.isEnabled -eq $false }
    Write-Verbose "Exporting $($status) Automation Rules..." 
}
else {
    Write-Verbose "Getting $($status) Automation Rules..."
    # Get all Automation Rules    
    $ruleResult = (Export-AutomationRules -apiVersion $apiVersion -subscriptionId $subscriptionId -resourceGroupName $resourceGroupName -workspaceName $workspaceName).value
    $activeRules = $ruleResult
    Write-Verbose "Exporting $($status) Automation Rules..." 
}

try {    
    if ($activeRules.count -eq 0) {
        throw "No $($status) Automation Rules were found!"
    }
    else {
        Write-Output "$($activeRules.count) $($status) Automation Rules were found!"
    }
}
catch {
    Write-Error $_ -ErrorAction Stop
}

$exportedActiveRules = @()

foreach ($activeRule in $activeRules) {
    $ruleExport = $activeRule | ConvertTo-Json -Depth 100

    Write-Verbose "Exporting [$($activeRule.properties.displayName).json] to C:\temp\"
    $ruleExport | Out-File -FilePath "C:\temp\$($activeRule.properties.displayName).json" -Force
    $exportedActiveRules += $rule
}

try {    
    if ($exportedActiveRules.count -eq 0) {
        throw "No $($status) Automation Rules were exported!"
    }
    else {
        Write-Output "$($exportedActiveRules.count) $($status) Automation Rules were exported!"
    }
}
catch {
    Write-Error $_ -ErrorAction Stop
}

```

```python
#!/usr/bin/env python3
"""
Export Microsoft Sentinel automation rules using Python and Azure REST API.

This script connects to Azure, retrieves all automation rules for a given
Sentinel workspace, filters them by status (Enabled, Disabled, or All),
and exports each rule to a JSON file.

Requirements:
    - Python 3.6+
    - azure-identity (`pip install azure-identity`)
    - requests (`pip install requests`)

Usage example:
    python export_automation_rules.py \
        --subscription-id <SUBSCRIPTION_ID> \
        --resource-group <RESOURCE_GROUP_NAME> \
        --workspace-name <WORKSPACE_NAME> \
        --status Enabled \
        --output-dir ./exported_rules
"""

import os
import argparse
import json
import sys

import requests
from azure.identity import DefaultAzureCredential


API_VERSION = "2024-03-01"
ARM_SCOPE = "https://management.azure.com/.default"


def get_bearer_token() -> str:
    """
    Acquire an Azure AD access token for the Azure Resource Manager scope
    using DefaultAzureCredential.
    """
    try:
        credential = DefaultAzureCredential()
        token = credential.get_token(ARM_SCOPE)
        return token.token
    except Exception as e:
        print(f"[ERROR] Failed to acquire Azure AD token: {e}", file=sys.stderr)
        sys.exit(1)


def fetch_all_automation_rules(
    subscription_id: str, resource_group: str, workspace_name: str, bearer_token: str
) -> list:
    """
    Retrieve all automation rules under the specified Sentinel workspace.
    Returns the list of rule objects as Python dicts.
    """

    base_url = (
        f"https://management.azure.com/"
        f"subscriptions/{subscription_id}/"
        f"resourceGroups/{resource_group}/"
        f"providers/Microsoft.OperationalInsights/"
        f"workspaces/{workspace_name}/"
        f"providers/Microsoft.SecurityInsights/"
        f"automationRules"
    )
    params = {"api-version": API_VERSION}

    headers = {
        "Authorization": f"Bearer {bearer_token}",
        "Content-Type": "application/json",
    }

    try:
        response = requests.get(base_url, headers=headers, params=params, timeout=30)
        response.raise_for_status()
    except requests.exceptions.HTTPError as http_err:
        print(f"[ERROR] HTTP error while fetching automation rules: {http_err}", file=sys.stderr)
        sys.exit(1)
    except Exception as err:
        print(f"[ERROR] Unexpected error while fetching automation rules: {err}", file=sys.stderr)
        sys.exit(1)

    payload = response.json()
    # The REST response structure: { "value": [ { rule1 }, { rule2 }, ... ] }
    return payload.get("value", [])


def filter_rules_by_status(rules: list, status: str) -> list:
    """
    Filter the list of rule dicts based on the 'status' parameter.
    - If status == "Enabled": return only rules with properties.triggeringLogic.isEnabled == True
    - If status == "Disabled": return only rules with properties.triggeringLogic.isEnabled == False
    - If status == "All": return all rules unchanged
    """
    status_lower = status.lower()
    if status_lower == "all":
        return rules

    desired = True if status_lower == "enabled" else False
    filtered = []
    for rule in rules:
        # Some older/rare rules might not have triggeringLogic; skip if missing
        is_enabled = (
            rule.get("properties", {})
            .get("triggeringLogic", {})
            .get("isEnabled", None)
        )
        if is_enabled is not None and is_enabled == desired:
            filtered.append(rule)

    return filtered


def sanitize_filename(name: str) -> str:
    """
    Replace any characters that are invalid in filenames with underscores.
    """
    invalid_chars = '<>:"/\\|?*'
    for ch in invalid_chars:
        name = name.replace(ch, "_")
    return name


def export_rules_to_files(rules: list, output_dir: str) -> None:
    """
    For each rule dict in 'rules', serialize to JSON and write to a file
    named '<displayName>.json' under 'output_dir'. Creates 'output_dir' if needed.
    """
    if not os.path.isdir(output_dir):
        os.makedirs(output_dir, exist_ok=True)

    exported_count = 0
    for rule in rules:
        display_name = rule.get("properties", {}).get("displayName") or rule.get("name")
        if not display_name:
            # Fall back to the resource name if displayName is missing
            display_name = rule.get("name", f"rule_{exported_count}")

        filename = sanitize_filename(display_name) + ".json"
        filepath = os.path.join(output_dir, filename)

        try:
            with open(filepath, "w", encoding="utf-8") as f:
                json.dump(rule, f, indent=4, ensure_ascii=False)
            print(f"[INFO] Exported rule '{display_name}' to: {filepath}")
            exported_count += 1
        except Exception as e:
            print(f"[WARNING] Failed to write rule '{display_name}' to file: {e}", file=sys.stderr)

    if exported_count == 0:
        print(f"[WARNING] No rules were exported. Check if the directory '{output_dir}' is writable.", file=sys.stderr)
    else:
        print(f"\n[INFO] Total exported rules: {exported_count}")


def main():
    parser = argparse.ArgumentParser(
        description="Export Microsoft Sentinel automation rules to JSON files."
    )
    parser.add_argument(
        "--subscription-id",
        "-s",
        required=True,
        help="Azure Subscription ID",
    )
    parser.add_argument(
        "--resource-group",
        "-g",
        required=True,
        help="Resource Group name containing the Sentinel workspace",
    )
    parser.add_argument(
        "--workspace-name",
        "-w",
        required=True,
        help="Log Analytics Workspace name (Sentinel workspace)",
    )
    parser.add_argument(
        "--status",
        "-t",
        required=False,
        choices=["Enabled", "Disabled", "All"],
        default="Enabled",
        help="Filter automation rules by status (Enabled, Disabled, or All)",
    )
    parser.add_argument(
        "--output-dir",
        "-o",
        required=False,
        default="./",
        help="Directory to export JSON files into (default: current directory)",
    )

    args = parser.parse_args()

    print("[INFO] Acquiring Azure AD token...")
    token = get_bearer_token()

    print(f"[INFO] Fetching all automation rules for workspace '{args.workspace_name}'...")
    all_rules = fetch_all_automation_rules(
        subscription_id=args.subscription_id,
        resource_group=args.resource_group,
        workspace_name=args.workspace_name,
        bearer_token=token,
    )

    if not all_rules:
        print(f"[ERROR] No automation rules found in workspace '{args.workspace_name}'. Exiting.", file=sys.stderr)
        sys.exit(1)

    filtered_rules = filter_rules_by_status(all_rules, args.status)
    count_filtered = len(filtered_rules)

    if count_filtered == 0:
        print(f"[ERROR] No '{args.status}' automation rules were found. Exiting.", file=sys.stderr)
        sys.exit(1)

    print(f"[INFO] Found {count_filtered} automation rule(s) with status '{args.status}'.")
    print(f"[INFO] Exporting to directory: {args.output_dir}\n")

    export_rules_to_files(filtered_rules, args.output_dir)


if __name__ == "__main__":
    main()
```

Explanation of the Python script:

1. Dependencies

azure-identity: Used to acquire a valid Azure AD access token via DefaultAzureCredential.

requests: Used to call the Azure REST API endpoint for automation rules.

argparse, os, sys, json: Standard Python libraries for CLI parsing, file I/O, and JSON serialization.



2. Authentication (get_bearer_token)
We call DefaultAzureCredential() from azure.identity, which will attempt to authenticate in this order by default:

Environment variables (e.g. AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_CLIENT_SECRET)

Managed Identity (if running in Azure)

Azure CLI cached credentials (if you have already run az login)
This token is then used in the Authorization: Bearer <token> header for our REST calls.



3. Fetching Automation Rules (fetch_all_automation_rules)
Constructs the REST API URI:

https://management.azure.com/
  subscriptions/{subscription_id}/
  resourceGroups/{resource_group}/
  providers/Microsoft.OperationalInsights/
  workspaces/{workspace_name}/
  providers/Microsoft.SecurityInsights/
  automationRules?api-version=2024-03-01

It then does an HTTP GET with the Bearer token. The response JSON has a value array containing all the rules.


4. Filtering by Status (filter_rules_by_status)

If status == "Enabled", we keep only rules where properties.triggeringLogic.isEnabled == True.

If status == "Disabled", we keep only those where it is False.

If status == "All", we return the entire list unchanged.



5. Exporting to Files (export_rules_to_files)

We create the output directory if it does not exist.

For each rule, we look up rule["properties"]["displayName"]. If that is missing, we fall back to the resource name or a synthetic placeholder.

We sanitize the display name to remove any filesystem-invalid characters.

We write the pretty-printed JSON representation (indent=4) to "<displayName>.json" under the specified output directory.



6. Command-Line Interface

The script is invoked with these required arguments:

--subscription-id    / -s   Azure Subscription ID
--resource-group     / -g   Name of the Resource Group
--workspace-name     / -w   Name of the Log Analytics (Sentinel) Workspace
--status             / -t   One of [Enabled, Disabled, All] (default: Enabled)
--output-dir         / -o   Where to save the JSON files (default: "./")

Example:

python export_automation_rules.py \
  --subscription-id 11111111-2222-3333-4444-555555555555 \
  --resource-group MySentinelRG \
  --workspace-name MySentinelWorkspace \
  --status Enabled \
  --output-dir C:/temp




Save this script as export_automation_rules.py, ensure you have authenticated via az login or set up environment credentials, then run it with the appropriate parameters. It will print informational messages as it acquires a token, fetches rules, filters them, and writes each rule’s JSON to a separate file.


https://techcommunity.microsoft.com/blog/microsoftsentinelblog/transferring-microsoft-sentinel-scheduled-alert-rules-between-different-workspac/3269113

To export and import Microsoft Sentinel automation rules and components (like analytic rules, playbooks, hunting queries, workbooks, etc.) across different Azure tenants, you can use Azure REST API or the Azure Resource Manager (ARM) templates.

Here's a comprehensive guide with:

1. REST API endpoints to export Sentinel components


2. Python logic for exporting and modifying resources


3. Python logic to import into another tenant




---

🔹 1. Key REST API Endpoints for Microsoft Sentinel Components

Sentinel Component	REST API Endpoint

Automation Rules	/subscriptions/{subId}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{ws}/providers/Microsoft.SecurityInsights/automationRules?api-version=2024-03-01
Analytic Rules	/subscriptions/{subId}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{ws}/providers/Microsoft.SecurityInsights/alertRules?api-version=2024-03-01
Hunting Queries	/subscriptions/{subId}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{ws}/providers/Microsoft.SecurityInsights/huntingQueries?api-version=2024-03-01
Watchlists	/subscriptions/{subId}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{ws}/providers/Microsoft.SecurityInsights/watchlists?api-version=2024-03-01
Workbooks	/subscriptions/{subId}/resourceGroups/{rg}/providers/Microsoft.Insights/components/{ws}/workbooks?api-version=2023-10-01



---

🔹 2. Python Script – Export Sentinel Components to JSON (with tenant-agnostic format)

Here’s a modular Python script that:

Authenticates via azure-identity

Calls REST API

Removes tenant/subscription-specific values

Saves in import-ready JSON format


🧩 export_sentinel_components.py

import os
import json
import requests
from azure.identity import DefaultAzureCredential

ARM_SCOPE = "https://management.azure.com/.default"
API_VERSION = "2024-03-01"
EXPORT_DIR = "./export"

# Define which components to export
SENTINEL_COMPONENTS = {
    "automationRules": "automationRules",
    "alertRules": "alertRules",  # Analytic rules
    "huntingQueries": "huntingQueries",
}

def get_token():
    credential = DefaultAzureCredential()
    return credential.get_token(ARM_SCOPE).token

def sanitize_and_export(resource, resource_type, output_dir):
    # Remove tenant-specific fields
    for field in ["id", "etag", "systemData"]:
        resource.pop(field, None)

    file_name = f"{resource.get('name')}.json"
    file_path = os.path.join(output_dir, resource_type, file_name)

    os.makedirs(os.path.dirname(file_path), exist_ok=True)
    with open(file_path, "w") as f:
        json.dump(resource, f, indent=2)
    print(f"[✓] Exported {resource_type}: {file_name}")

def export_resources(subscription_id, rg_name, workspace_name):
    token = get_token()
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }

    for component, path in SENTINEL_COMPONENTS.items():
        print(f"\n[INFO] Exporting {component}...")
        url = f"https://management.azure.com/subscriptions/{subscription_id}/resourceGroups/{rg_name}" \
              f"/providers/Microsoft.OperationalInsights/workspaces/{workspace_name}/providers/Microsoft.SecurityInsights/{path}?api-version={API_VERSION}"

        response = requests.get(url, headers=headers)
        response.raise_for_status()

        for item in response.json().get("value", []):
            sanitize_and_export(item, component, EXPORT_DIR)

# Example usage
if __name__ == "__main__":
    SUBSCRIPTION_ID = "<SOURCE_SUBSCRIPTION_ID>"
    RESOURCE_GROUP = "<SOURCE_RG>"
    WORKSPACE = "<SOURCE_WORKSPACE>"

    export_resources(SUBSCRIPTION_ID, RESOURCE_GROUP, WORKSPACE)


---

🔹 3. Python Script – Import into Target Tenant

> You’ll need appropriate RBAC permissions to create Sentinel components in the destination tenant.



🧩 import_sentinel_components.py

import os
import json
import requests
from azure.identity import DefaultAzureCredential

ARM_SCOPE = "https://management.azure.com/.default"
API_VERSION = "2024-03-01"
IMPORT_DIR = "./export"  # Use the directory from export

COMPONENTS = {
    "automationRules": "automationRules",
    "alertRules": "alertRules",
    "huntingQueries": "huntingQueries"
}

def get_token():
    credential = DefaultAzureCredential()
    return credential.get_token(ARM_SCOPE).token

def import_resource(resource, component_type, subscription_id, rg_name, workspace_name, headers):
    name = resource.get("name")
    url = f"https://management.azure.com/subscriptions/{subscription_id}/resourceGroups/{rg_name}" \
          f"/providers/Microsoft.OperationalInsights/workspaces/{workspace_name}/providers/Microsoft.SecurityInsights/{component_type}/{name}?api-version={API_VERSION}"

    response = requests.put(url, headers=headers, json=resource)
    if response.status_code in [200, 201]:
        print(f"[✓] Imported: {component_type}/{name}")
    else:
        print(f"[✗] Failed to import {name}: {response.status_code}, {response.text}")

def import_all(subscription_id, rg_name, workspace_name):
    token = get_token()
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }

    for component_type in COMPONENTS:
        dir_path = os.path.join(IMPORT_DIR, component_type)
        if not os.path.isdir(dir_path):
            continue
        for file in os.listdir(dir_path):
            with open(os.path.join(dir_path, file), "r") as f:
                resource = json.load(f)
                import_resource(resource, component_type, subscription_id, rg_name, workspace_name, headers)

# Example usage
if __name__ == "__main__":
    TARGET_SUBSCRIPTION_ID = "<TARGET_SUBSCRIPTION_ID>"
    TARGET_RG = "<TARGET_RESOURCE_GROUP>"
    TARGET_WORKSPACE = "<TARGET_WORKSPACE>"

    import_all(TARGET_SUBSCRIPTION_ID, TARGET_RG, TARGET_WORKSPACE)


---

🔹 4. Tips for Cross-Tenant Use

Use az login --tenant <tenant-id> or DefaultAzureCredential with environment variables to authenticate into the target tenant.

Consider using a Service Principal with delegated permissions to automate the import/export.

Avoid exporting system-managed resources, like built-in playbooks or managed rules unless needed.



---

🔹 5. Optional Enhancements

Add support for Workbooks and Watchlists

Add CLI parameters (e.g., argparse) to make scripts dynamic

Add parallel processing or batching for large environments



---

Would you like these scripts packaged together or modified for a specific CI/CD environment (e.g., Azure DevOps, GitHub Actions)?

Yes! Below is a Terraform-based approach to deploy sanitized Microsoft Sentinel Automation Rules (or other components) in a new tenant using the sanitized JSON files you exported earlier. This assumes you've used the Python export script I shared earlier and saved all the resources in individual JSON files.


---

✅ Assumptions

You have a set of JSON files (one per rule/component) stored locally.

You are authenticated to the target Azure tenant using Terraform (via az login, service principal, etc.).

Your sanitized JSON files do not include fields like id, etag, systemData, etc.



---

📁 Directory Structure Example

terraform-sentinel-import/
│
├── main.tf
├── variables.tf
├── automation_rules/
│   ├── rule1.json
│   ├── rule2.json
│   └── ...


---

🔧 Step-by-Step Terraform Code

🔹 variables.tf

variable "subscription_id" {
  type = string
}

variable "resource_group_name" {
  type = string
}

variable "workspace_name" {
  type = string
}

variable "automation_rule_files" {
  type        = list(string)
  description = "List of JSON files for Sentinel Automation Rules"
}


---

🔹 main.tf

provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}

locals {
  automation_rules = [
    for file_path in var.automation_rule_files :
    jsondecode(file(file_path))
  ]
}

resource "azurerm_sentinel_automation_rule" "rules" {
  for_each = {
    for rule in local.automation_rules : rule.name => rule
  }

  name                = each.value.name
  display_name        = each.value.properties.displayName
  log_analytics_workspace_id = "/subscriptions/${var.subscription_id}/resourceGroups/${var.resource_group_name}/providers/Microsoft.OperationalInsights/workspaces/${var.workspace_name}"
  order               = each.value.properties.order
  triggering_logic {
    triggers_on          = each.value.properties.triggeringLogic.triggersOn
    triggers_when        = each.value.properties.triggeringLogic.triggersWhen
    is_enabled           = each.value.properties.triggeringLogic.isEnabled
    conditions           = try(each.value.properties.triggeringLogic.conditions, [])
  }

  actions = [
    for action in each.value.properties.actions : {
      order      = action.order
      action_type = action.actionType
      logic_app_resource_id = try(action.logicAppResourceId, null)
    }
  ]
}

> You may need to adapt actions and triggering_logic.conditions parsing depending on how complex your automation rules are.




---

🔹 Example terraform.tfvars

subscription_id       = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
resource_group_name   = "sentinel-rg"
workspace_name        = "sentinel-workspace"
automation_rule_files = [
  "${path.module}/automation_rules/rule1.json",
  "${path.module}/automation_rules/rule2.json"
]


---

🧪 To Deploy:

cd terraform-sentinel-import
terraform init
terraform plan -var-file="terraform.tfvars"
terraform apply -var-file="terraform.tfvars"


---

📝 Notes:

Terraform does not natively support all Sentinel components like analytic rules or watchlists, so for those you must either:

Use the ARM template module via azurerm_resource_group_template_deployment

OR manage via custom provider or local-exec scripts


The Terraform provider azurerm currently (as of mid-2025) supports:

azurerm_sentinel_automation_rule

azurerm_sentinel_alert_rule_* (e.g., scheduled, fusion)

Others may need workarounds




---

📦 Optional: Automatically Load All JSONs

You can use this instead of manually listing files in terraform.tfvars:

variable "automation_rule_folder" {
  type = string
  default = "${path.module}/automation_rules"
}

locals {
  automation_rule_files = fileset(var.automation_rule_folder, "*.json")
}

# and then loop over local.automation_rule_files after prefixing with folder path


---

Would you like me to generate a full template including analytic rules, or would you prefer a ready-to-use Terraform module with CLI support to point to folders?

Great question — for Microsoft Sentinel Workbooks, Hunting Queries, and Watchlists, Terraform currently does not have first-class native support via resources like it does for automation rules or analytic rules. However, you can still deploy them using Terraform with the azurerm_resource_group_template_deployment resource and ARM template JSONs that you previously exported and sanitized.


---

✅ Summary of Deployment Methods

Sentinel Component	Terraform Native Support?	Recommended Terraform Method

Automation Rules	✅ Yes	azurerm_sentinel_automation_rule
Analytic Rules	✅ Yes	azurerm_sentinel_alert_rule_* resources
Workbooks	❌ No	azurerm_resource_group_template_deployment with ARM JSON
Hunting Queries	❌ No	ARM template deployment
Watchlists	❌ Partial	ARM template or REST API via null_resource/script



---

🔧 Deploy Workbooks, Hunting Queries, Watchlists using ARM JSON

🔹 1. Structure Example

sentinel-import/
├── main.tf
├── variables.tf
├── terraform.tfvars
├── arm_templates/
│   ├── workbook1.json
│   ├── hunting_query1.json
│   ├── watchlist1.json


---

🔹 2. main.tf

provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}

resource "azurerm_resource_group_template_deployment" "workbooks" {
  for_each = {
    for file_path in var.workbook_files :
    basename(file_path) => file_path
  }

  name                = "workbook-deployment-${each.key}"
  resource_group_name = var.resource_group_name
  deployment_mode     = "Incremental"

  template_body = file(each.value)

  parameters = {}  # If your ARM templates require parameters, add them here

  lifecycle {
    ignore_changes = [template_body]
  }
}

resource "azurerm_resource_group_template_deployment" "hunting_queries" {
  for_each = {
    for file_path in var.hunting_query_files :
    basename(file_path) => file_path
  }

  name                = "hunting-query-deployment-${each.key}"
  resource_group_name = var.resource_group_name
  deployment_mode     = "Incremental"

  template_body = file(each.value)

  parameters = {}
}


---

🔹 3. variables.tf

variable "subscription_id" {
  type = string
}

variable "resource_group_name" {
  type = string
}

variable "workbook_files" {
  type = list(string)
}

variable "hunting_query_files" {
  type = list(string)
}


---

🔹 4. terraform.tfvars Example

subscription_id     = "xxxx-xxxx-xxxx-xxxx"
resource_group_name = "sentinel-rg"

workbook_files = [
  "${path.module}/arm_templates/workbook1.json"
]

hunting_query_files = [
  "${path.module}/arm_templates/hunting_query1.json"
]


---

🧠 Notes

1. Make sure your exported JSONs from Python have this structure for ARM deployment:

{
  "$schema": "...",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.Insights/workbooks",
      ...
    }
  ]
}


2. For Watchlists, include type:

"type": "Microsoft.SecurityInsights/watchlists",
"apiVersion": "2024-03-01"


3. If the exported files don’t follow ARM schema, you’ll need to wrap the exported resource in an ARM wrapper like this:

{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    << your exported object >>
  ]
}




---

🚀 Want Full Terraform Automation?

I can generate a single main.tf that automatically:

Loads all .json files from a directory

Deploys all workbooks, watchlists, and huntingQueries

Uses fileset() and jsondecode() to handle inline resources


Let me know if you want this auto-wrapping + deploy capability.



param (
    [Parameter(Mandatory)]
    [string]$SubscriptionId,

    [Parameter(Mandatory)]
    [string]$ResourceGroupName,

    [Parameter(Mandatory)]
    [string]$WorkspaceName
)

$ErrorActionPreference = 'Stop'

# Ensure modules
Import-Module Az.Accounts -ErrorAction Stop
Import-Module Az.Resources -ErrorAction Stop
Import-Module Az.OperationalInsights -ErrorAction Stop

# Login and Set Context
Write-Host "Connecting to Azure..." -ForegroundColor Cyan
Connect-AzAccount -ErrorAction Stop
Set-AzContext -SubscriptionId $SubscriptionId

# Token for REST API
$token = (Get-AzAccessToken).Token
$authHeader = @{
    'Authorization' = "Bearer $token"
    'Content-Type'  = 'application/json'
}

# Base URI
$baseUri = "https://management.azure.com/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.OperationalInsights/workspaces/$WorkspaceName/providers/Microsoft.SecurityInsights"
$apiVersion = "?api-version=2024-03-01"

# Resource list
$resourceTypes = @{
    "analyticRules"   = "Microsoft.SecurityInsights/alertRules"
    "automationRules" = "Microsoft.SecurityInsights/automationRules"
    "workbooks"       = "Microsoft.Insights/workbooks"
    "huntingQueries"  = "Microsoft.SecurityInsights/huntingQueries"
    "watchlists"      = "Microsoft.SecurityInsights/watchlists"
    "dataConnectors"  = "Microsoft.SecurityInsights/dataConnectors"
}

$allResources = @()

foreach ($key in $resourceTypes.Keys) {
    $uri = if ($key -eq "workbooks") {
        # Workbook is at RG level, not under SecurityInsights
        "https://management.azure.com/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Insights/workbooks$apiVersion"
    } else {
        "$baseUri/$($key)$apiVersion"
    }

    Write-Host "Fetching $key..." -ForegroundColor Yellow
    $result = Invoke-RestMethod -Uri $uri -Method GET -Headers $authHeader
    foreach ($res in $result.value) {
        # Remove non-exportable fields
        $res.PSObject.Properties.Remove("etag")
        $res.PSObject.Properties.Remove("id")
        $res.PSObject.Properties.Remove("systemData")
        $res.PSObject.Properties.Remove("name") # Will be re-added
        $res.PSObject.Properties.Remove("type") # Will be re-added

        $resourceObj = @{
            type       = $res.type
            name       = "[concat('$($res.name)')]"
            apiVersion = "2024-03-01"
            properties = $res.properties
        }

        if ($key -eq "workbooks") {
            $resourceObj["location"] = $res.location
            $resourceObj["kind"]     = $res.kind
            $resourceObj["identity"] = $res.identity
        }

        $allResources += $resourceObj
    }
}

# ARM Template output
$template = @{
    '$schema'         = "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#"
    contentVersion    = "1.0.0.0"
    parameters        = @{}
    variables         = @{}
    resources         = $allResources
    outputs           = @{}
}

# Output path
$outputFolder = "C:\SentinelExport"
$outputFile   = Join-Path $outputFolder "Sentinel-ARM-Template.json"

if (-not (Test-Path $outputFolder)) {
    New-Item -ItemType Directory -Path $outputFolder -Force
}

$template | ConvertTo-Json -Depth 10 | Out-File -FilePath $outputFile -Force -Encoding utf8

Write-Host "`nExport complete! ARM Template saved to:`n$outputFile" -ForegroundColor Green


Here's a PowerShell script that does the following:

✅ Connects to Azure
✅ Collects Microsoft Sentinel content types:

Analytic Rules

Automation Rules

Workbooks

Hunting Queries

Watchlists

Data Connectors
✅ Exports each as JSON
✅ Wraps them all in a valid ARM template structure for deployment in a new tenant/subscription



---

✅ Prerequisites:

Az PowerShell Module: Az.Accounts, Az.Resources, Az.OperationalInsights

Logged into Azure: Connect-AzAccount

Microsoft Sentinel enabled in the workspace



---

📂 Output Structure:

Exports into C:\SentinelExport\Resources.json (full ARM template)


---

🧾 PowerShell Script: Export-Sentinel-ToARM.ps1
```
param (
    [Parameter(Mandatory)]
    [string]$SubscriptionId,

    [Parameter(Mandatory)]
    [string]$ResourceGroupName,

    [Parameter(Mandatory)]
    [string]$WorkspaceName
)

$ErrorActionPreference = 'Stop'

# Ensure modules
Import-Module Az.Accounts -ErrorAction Stop
Import-Module Az.Resources -ErrorAction Stop
Import-Module Az.OperationalInsights -ErrorAction Stop

# Login and Set Context
Write-Host "Connecting to Azure..." -ForegroundColor Cyan
Connect-AzAccount -ErrorAction Stop
Set-AzContext -SubscriptionId $SubscriptionId

# Token for REST API
$token = (Get-AzAccessToken).Token
$authHeader = @{
    'Authorization' = "Bearer $token"
    'Content-Type'  = 'application/json'
}

# Base URI
$baseUri = "https://management.azure.com/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.OperationalInsights/workspaces/$WorkspaceName/providers/Microsoft.SecurityInsights"
$apiVersion = "?api-version=2024-03-01"

# Resource list
$resourceTypes = @{
    "analyticRules"   = "Microsoft.SecurityInsights/alertRules"
    "automationRules" = "Microsoft.SecurityInsights/automationRules"
    "workbooks"       = "Microsoft.Insights/workbooks"
    "huntingQueries"  = "Microsoft.SecurityInsights/huntingQueries"
    "watchlists"      = "Microsoft.SecurityInsights/watchlists"
    "dataConnectors"  = "Microsoft.SecurityInsights/dataConnectors"
}

$allResources = @()

foreach ($key in $resourceTypes.Keys) {
    $uri = if ($key -eq "workbooks") {
        # Workbook is at RG level, not under SecurityInsights
        "https://management.azure.com/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Insights/workbooks$apiVersion"
    } else {
        "$baseUri/$($key)$apiVersion"
    }

    Write-Host "Fetching $key..." -ForegroundColor Yellow
    $result = Invoke-RestMethod -Uri $uri -Method GET -Headers $authHeader
    foreach ($res in $result.value) {
        # Remove non-exportable fields
        $res.PSObject.Properties.Remove("etag")
        $res.PSObject.Properties.Remove("id")
        $res.PSObject.Properties.Remove("systemData")
        $res.PSObject.Properties.Remove("name") # Will be re-added
        $res.PSObject.Properties.Remove("type") # Will be re-added

        $resourceObj = @{
            type       = $res.type
            name       = "[concat('$($res.name)')]"
            apiVersion = "2024-03-01"
            properties = $res.properties
        }

        if ($key -eq "workbooks") {
            $resourceObj["location"] = $res.location
            $resourceObj["kind"]     = $res.kind
            $resourceObj["identity"] = $res.identity
        }

        $allResources += $resourceObj
    }
}

# ARM Template output
$template = @{
    '$schema'         = "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#"
    contentVersion    = "1.0.0.0"
    parameters        = @{}
    variables         = @{}
    resources         = $allResources
    outputs           = @{}
}

# Output path
$outputFolder = "C:\SentinelExport"
$outputFile   = Join-Path $outputFolder "Sentinel-ARM-Template.json"

if (-not (Test-Path $outputFolder)) {
    New-Item -ItemType Directory -Path $outputFolder -Force
}

$template | ConvertTo-Json -Depth 10 | Out-File -FilePath $outputFile -Force -Encoding utf8

Write-Host "`nExport complete! ARM Template saved to:`n$outputFile" -ForegroundColor Green
```

---

✅ Deployment to New Tenant

1. Copy the exported Sentinel-ARM-Template.json to the new environment.


2. Deploy with Azure CLI, PowerShell, or Terraform:



🔹 Option 1: PowerShell
```
New-AzResourceGroupDeployment -Name "SentinelImport" `
  -ResourceGroupName "<new-RG>" `
  -TemplateFile "Sentinel-ARM-Template.json"
```
🔹 Option 2: Azure CLI
```
az deployment group create \
  --resource-group <new-RG> \
  --template-file Sentinel-ARM-Template.json
```

---

🔐 Notes

Make sure you sanitize sensitive fields like logicAppResourceId, tenantId, etc., before reusing in new tenant.

Some resources (like certain Data Connectors) may require manual setup due to dependency on external sources (like AAD, Defender, etc.).

You may optionally split the script into multiple files per component for modular use.



---

Would you like a version of the script that exports each resource to its own JSON file for easier editing or version control (Git)?




Great! Below is a modular PowerShell script that:

✅ Exports each Microsoft Sentinel component to its own individual JSON file
✅ Saves in organized folders (AnalyticsRules, AutomationRules, etc.)
✅ Prepares each file for ARM deployment (removes read-only/system fields)
✅ You can later assemble these JSONs into an ARM template or deploy them via Terraform


---

📂 Output Folder Structure (under C:\SentinelExport)
```
C:\SentinelExport\
├── AnalyticsRules\
├── AutomationRules\
├── Workbooks\
├── HuntingQueries\
├── Watchlists\
└── DataConnectors\
```

---
```
🧾 Script: Export-Sentinel-Components.ps1

param (
    [Parameter(Mandatory)]
    [string]$SubscriptionId,

    [Parameter(Mandatory)]
    [string]$ResourceGroupName,

    [Parameter(Mandatory)]
    [string]$WorkspaceName
)

$ErrorActionPreference = 'Stop'

# Login & context
Connect-AzAccount -ErrorAction Stop
Set-AzContext -SubscriptionId $SubscriptionId

# Token for REST API
$token = (Get-AzAccessToken).Token
$authHeader = @{
    'Authorization' = "Bearer $token"
    'Content-Type'  = 'application/json'
}

# Resource definitions
$resourceMap = @{
    "AnalyticsRules"   = "alertRules"
    "AutomationRules"  = "automationRules"
    "HuntingQueries"   = "huntingQueries"
    "Watchlists"       = "watchlists"
    "DataConnectors"   = "dataConnectors"
}
$apiVersion = "?api-version=2024-03-01"

$outputBase = "C:\SentinelExport"
if (-not (Test-Path $outputBase)) {
    New-Item -Path $outputBase -ItemType Directory -Force
}

# Workbooks handled separately
$workbookUri = "https://management.azure.com/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.Insights/workbooks$apiVersion"
$workbookFolder = Join-Path $outputBase "Workbooks"
New-Item -ItemType Directory -Path $workbookFolder -Force | Out-Null

Write-Host "Exporting Workbooks..." -ForegroundColor Cyan
$workbooks = Invoke-RestMethod -Uri $workbookUri -Method GET -Headers $authHeader
foreach ($item in $workbooks.value) {
    $export = @{
        type       = $item.type
        name       = $item.name
        apiVersion = "2024-03-01"
        location   = $item.location
        kind       = $item.kind
        identity   = $item.identity
        properties = $item.properties
    }
    $export | ConvertTo-Json -Depth 10 | Out-File -FilePath "$workbookFolder\$($item.name).json" -Encoding utf8 -Force
}

# Export Sentinel Resources
foreach ($folderName in $resourceMap.Keys) {
    $typeUri = $resourceMap[$folderName]
    $uri = "https://management.azure.com/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroupName/providers/Microsoft.OperationalInsights/workspaces/$WorkspaceName/providers/Microsoft.SecurityInsights/$typeUri$apiVersion"

    $outputFolder = Join-Path $outputBase $folderName
    New-Item -ItemType Directory -Path $outputFolder -Force | Out-Null

    Write-Host "Exporting $folderName..." -ForegroundColor Cyan

    $results = Invoke-RestMethod -Uri $uri -Method GET -Headers $authHeader

    foreach ($item in $results.value) {
        $item.PSObject.Properties.Remove("etag")
        $item.PSObject.Properties.Remove("id")
        $item.PSObject.Properties.Remove("systemData")
        $item.PSObject.Properties.Remove("kind")

        $export = @{
            type       = $item.type
            name       = $item.name
            apiVersion = "2024-03-01"
            properties = $item.properties
        }

        if ($item.location) { $export["location"] = $item.location }

        $filename = "$($outputFolder)\$($item.name).json"
        $export | ConvertTo-Json -Depth 10 | Out-File -FilePath $filename -Encoding utf8 -Force
    }
}

Write-Host "`nExport completed! Resources saved under: $outputBase" -ForegroundColor Green

```
---

✅ Benefits

Easy to version-control in Git

Easy to update or edit specific resources (e.g., Edit-C:\SentinelExport\AnalyticsRules\HighCPUAlert.json)

Compatible with ARM, Terraform, or Azure CLI



---

🔄 Next Steps

Let me know if you want:

A reassembly script to combine these back into one template.json for deployment

Terraform wrapper to deploy these *.json files

A script that also exports Logic Apps attached to Automation Rules


I can generate all of those for you.


Great! Here's everything you need for a full Sentinel export/import pipeline:


---

✅ 1. Modular PowerShell Script

→ Already provided above:
Exports Analytics Rules, Automation Rules, Workbooks, Hunting Queries, Watchlists, and Data Connectors as individual JSONs.


---

🔁 2. Script to Reassemble into a Unified ARM Template
```
param (
    [string]$InputFolder = "C:\SentinelExport",
    [string]$OutputFile = "C:\SentinelExport\Sentinel-ARM-Template.json"
)

$allResources = @()

Get-ChildItem -Path $InputFolder -Recurse -Filter *.json | ForEach-Object {
    $content = Get-Content $_.FullName -Raw | ConvertFrom-Json
    $allResources += $content
}

$template = @{
    '$schema'         = 'https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#'
    contentVersion    = '1.0.0.0'
    parameters        = @{}
    variables         = @{}
    resources         = $allResources
    outputs           = @{}
}

$template | ConvertTo-Json -Depth 20 | Out-File -FilePath $OutputFile -Encoding utf8 -Force
Write-Host "`nUnified ARM template created at: $OutputFile" -ForegroundColor Green
```

---

🧱 3. Terraform Wrapper to Deploy the JSON Files

Create a folder with:

📄 main.tf
```
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group_template_deployment" "sentinel_import" {
  name                = "sentinel-import"
  resource_group_name = var.resource_group_name
  deployment_mode     = "Incremental"

  template_content    = file("${path.module}/Sentinel-ARM-Template.json")
}

📄 variables.tf

variable "resource_group_name" {
  type = string
}

📄 terraform.tfvars

resource_group_name = "your-new-RG-name"
```
Run:
```
terraform init
terraform apply

```
---

🔌 4. Export Logic Apps for Automation Rules (If Any)

📄 Export-LogicApps.ps1
```
param (
    [Parameter(Mandatory)]
    [string]$ResourceGroupName
)

$logicApps = Get-AzLogicApp -ResourceGroupName $ResourceGroupName
$outputDir = "C:\SentinelExport\LogicApps"
New-Item -Path $outputDir -ItemType Directory -Force | Out-Null

foreach ($app in $logicApps) {
    $definition = Get-AzLogicAppDefinition -ResourceGroupName $ResourceGroupName -Name $app.Name
    $filePath = Join-Path $outputDir "$($app.Name).json"
    $definition.Definition | ConvertTo-Json -Depth 20 | Out-File -FilePath $filePath -Encoding utf8 -Force
}

✅ You can later re-import them with:

New-AzLogicApp -ResourceGroupName <newRG> -Name <LogicAppName> -Location <region> -DefinitionFile <path>
```

---

🧾 Summary of All Deliverables

Task	Script/Tool

Export Sentinel components	Export-Sentinel-Components.ps1
Rebuild single ARM template	Reassemble-ARM.ps1
Deploy via Terraform	main.tf, variables.tf, Sentinel-ARM-Template.json
Export Logic Apps	Export-LogicApps.ps1



---

Would you like me to bundle these in a downloadable ZIP or GitHub-ready structure?

Great follow-up! Here's how you can import/deploy Logic Apps (Standard or Consumption) in a new tenant/resource group, using PowerShell or ARM/Terraform.


---

🔁 1. Import Logic App using PowerShell

After exporting a Logic App’s definition (from Export-LogicApps.ps1), you can import it using:

📄 PowerShell Script to Import Logic App
```
param (
    [string]$ResourceGroupName,
    [string]$LogicAppName,
    [string]$Location,
    [string]$DefinitionPath
)

$definition = Get-Content -Path $DefinitionPath -Raw | ConvertFrom-Json

New-AzLogicApp -ResourceGroupName $ResourceGroupName `
               -Name $LogicAppName `
               -Location $Location `
               -Definition $definition

Write-Host "Logic App [$LogicAppName] deployed successfully in [$ResourceGroupName]" -ForegroundColor Green
```

> 📝 Note: This works for Logic Apps Consumption Plan (under Microsoft.Logic/workflows), which Sentinel typically uses for playbooks.




---

🧱 2. Deploy Logic Apps via ARM Template

You can include logic app files like this:

📄 ARM Template Snippet
```
{
  "type": "Microsoft.Logic/workflows",
  "apiVersion": "2019-05-01",
  "name": "[parameters('logicAppName')]",
  "location": "[parameters('location')]",
  "properties": {
    "definition": {
      "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
      "actions": {
        ...
      },
      "contentVersion": "1.0.0.0",
      "outputs": {},
      "parameters": {},
      "triggers": {
        ...
      }
    }
  }
}
```

Use parameters or static names for easy ARM deployment.


---

🌍 3. Deploy Logic App via Terraform

> You can also use Terraform if you're using it for Sentinel.



📄 Terraform Resource

resource "azurerm_logic_app_workflow" "example" {
  name                = "MyImportedLogicApp"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  definition          = jsondecode(file("logicapps/MyImportedLogicApp.json"))
}


---

✅ Recommendations for Logic App Imports

Requirement	Method	Notes

Quick, one-time import	PowerShell	Good for dev/test
Part of full infrastructure as code	ARM or Terraform	Ideal for consistent deployment
Includes connections/authentication	ARM + Parameters	Secure credential injection needed



---

Would you like:

A starter Terraform module that includes logic app import?

A packaged ARM template that combines Logic App + Sentinel rules?


Let me know how you want to manage deployments (PowerShell, ARM, Terraform) and I’ll tailor it to that.






Excellent and very relevant question. When it comes to Data Connectors in Microsoft Sentinel, the export/import story is quite different from other resources like analytics or automation rules.


---

🔎 What Are Sentinel Data Connectors?

Data connectors are pre-built integration templates for bringing in data from sources like:

Azure resources (e.g., Azure AD, Defender, Activity Logs)

Microsoft 365

Non-Microsoft sources (e.g., AWS, Cisco, Palo Alto, etc.)

Custom logs


They mostly configure diagnostic settings, API connections, and parsers.


---

🤔 Should You Export Data Connectors?

❌ NOT Worth It in Most Cases

Here’s why:

Reason	Explanation

Predefined	Data connectors are built-in. You don’t need to export them. You just enable them again in the new tenant.
Tenant-Specific Settings	Many connectors rely on things like tenant ID, subscription, Azure AD app registrations, etc. These won’t work cross-tenant unless reconfigured.
Manual/Portal Setup Often Required	Enabling connectors like Microsoft 365 Defender or MCAS often requires consent or setup via the portal.
Authentication-Dependent	Connectors use secure credentials, tokens, or service principals that don’t transfer.



---

✅ When It Does Make Sense

Scenario	Recommendation

Custom Data Connectors (via APIs, Event Hubs, Syslog)	✅ Export the ARM/Terraform for these
Log Analytics Diagnostic Settings for Azure services	✅ Export diagnostic settings using PowerShell or ARM
Large-scale infrastructure automation	✅ Consider ARM or Terraform deployment of diagnostics/log piping, not connector GUIs



---

🧱 Exporting Diagnostic Settings (Instead of Connectors)

Here’s how to export diagnostic settings (what most connectors actually configure):

PowerShell

$resourceId = "/subscriptions/<subId>/resourceGroups/<rg>/providers/Microsoft.OperationalInsights/workspaces/<workspaceName>"
Get-AzDiagnosticSetting -ResourceId $resourceId | ConvertTo-Json -Depth 10

You can then convert this into an ARM or Terraform file for reuse.


---

📝 Summary

Connector Type	Export/Import Feasible?	Alternative Approach

Microsoft-native (M365, Defender)	❌ No – reconfigure manually	Re-enable in portal
Custom Event Hub/Syslog/Data Collector	✅ Yes	Use ARM/Terraform
Azure Diagnostic Piping	✅ Yes	Export diagnostic settings



---

✔️ Recommendation

> Don’t waste time trying to export/import built-in data connectors.
Instead, focus on:



Re-enabling them in the new tenant (manually or scripted)

Exporting diagnostic settings + custom collector configs



---

If you want, I can:

Generate ARM/Terraform for a custom Event Hub or Syslog connector

Script the diagnostic setting export/import process


Let me know how deep you want to go into this.



