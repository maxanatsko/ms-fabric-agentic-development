---
name: dataflow-gen1-migration-az-cli
description: "Migrate, copy, or bulk-move Power BI Dataflow Gen1 artifacts between workspaces with Azure CLI or az rest. Use for Gen1 to Dataflow Gen2 CI/CD save-as, workspace migrations, target-name conflict checks, special-character-safe renaming, connection-risk inventory, provisioning verification, and migration reporting. Triggers include migrate Gen1 dataflow, move dataflows between workspaces, copy Gen1 dataflow with az cli, saveAsNativeArtifact, and bulk Dataflow Gen1 migration."
---

# Dataflow Gen1 Migration with Azure CLI

Use the Power BI `saveAsNativeArtifact` preview API to create Dataflow Gen2 CI/CD copies of Gen1 dataflows in a target workspace. Treat this operation as a conversion, not a direct Gen1-to-Gen1 clone.

## Prerequisite: Azure CLI

Check for Azure CLI before any discovery or migration command:

```powershell
az --version
```

If `az` is unavailable, ask the user to install it from [Install the Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli). On Windows, install with `winget install --exact --id Microsoft.AzureCLI`; on macOS, use `brew install azure-cli`; on supported Linux distributions, use the commands on the Microsoft installation page. Restart the terminal after installation, then authenticate and verify the signed-in account:

```powershell
az login
az account show --query "{user:user.name, tenantId:tenantId, subscription:name}" -o json
```

Do not request, capture, or handle passwords, tokens, or other credentials in chat. Require the user to complete interactive sign-in in their own terminal.

## Safety Rules

- Establish that every source dataflow has `generation: 1` before a write operation. Do not use this workflow for Gen2 sources.
- Never delete or alter the source Gen1 dataflow.
- Use `includeSchedule: false` unless the user explicitly requests schedule migration. If the user requests it, copy the schedule but verify the result before use. Never trigger a refresh without explicit user approval.
- Check the target for the proposed name before each call. `saveAsNativeArtifact` is not idempotent, and retries create duplicates.
- Use a target name that contains only Unicode letters, digits, and spaces. Replace `&` with `and`, remove other punctuation and symbols, and require approval for the resulting name. The service can return a non-descriptive `InvalidRequest: Unexpected dataflow error` for unsupported special characters.
- Copy only after the source inventory and target conflict checks pass. Confirm that the result reaches `Active`.
- Treat `ConnectionsUpdateFailed` as a successful artifact copy that requires manual connection remediation.

## API Audiences

| API | `az rest --resource` | Purpose |
|---|---|---|
| Power BI REST API | `https://analysis.windows.net/powerbi/api` | Source Gen1 discovery, sources, dependencies, and `saveAsNativeArtifact` |
| Fabric REST API | `https://api.fabric.microsoft.com` | Target Dataflow Gen2 CI/CD inventory and artifact verification |

## Phase 1: Inventory and Preflight

Set the IDs once:

```powershell
$sourceWorkspaceId = '<source-workspace-guid>'
$targetWorkspaceId = '<target-workspace-guid>'
$pbiResource = 'https://analysis.windows.net/powerbi/api'
$fabricResource = 'https://api.fabric.microsoft.com'
```

List source dataflows and select Gen1 candidates:

```powershell
$sourceJson = az rest --method get `
  --resource $pbiResource `
  --url "https://api.powerbi.com/v1.0/myorg/groups/$sourceWorkspaceId/dataflows" `
  -o json

if ($LASTEXITCODE -ne 0 -or [string]::IsNullOrWhiteSpace($sourceJson)) {
  throw 'Could not inventory source dataflows.'
}

try {
  $source = $sourceJson | ConvertFrom-Json -ErrorAction Stop
}
catch {
  throw 'The source dataflow inventory returned invalid JSON.'
}

if ($null -eq $source.value) {
  throw 'The source dataflow inventory did not contain a value collection.'
}

$source.value |
  Where-Object { $_.generation -eq 1 } |
  Select-Object name, objectId, generation, configuredBy, modelUrl
```

Inspect the source before each migration and preserve its `objectId` for the save-as request:

```powershell
$sourceDataflow = $source.value |
  Where-Object { $_.name -eq '<source-dataflow-name>' -and $_.generation -eq 1 }

if ($null -eq $sourceDataflow) {
  throw 'A Gen1 source dataflow with the requested name was not found.'
}
```

Check the source data sources and upstream dataflows. Sources such as Snowflake, SharePoint List, and Power Platform Dataflows can need connection remediation after conversion. Treat a non-null `modelUrl` that points at custom storage as a BYOL/BYOSA migration risk.

```powershell
$sourceDataflowId = $sourceDataflow.objectId

$dataSourcesJson = az rest --method get `
  --resource $pbiResource `
  --url "https://api.powerbi.com/v1.0/myorg/groups/$sourceWorkspaceId/dataflows/$sourceDataflowId/datasources" `
  -o json

if ($LASTEXITCODE -ne 0 -or [string]::IsNullOrWhiteSpace($dataSourcesJson)) {
  throw "Could not inventory data sources for '$sourceDataflowId'."
}

try {
  $dataSources = $dataSourcesJson | ConvertFrom-Json -ErrorAction Stop
}
catch {
  throw "The data-source inventory for '$sourceDataflowId' returned invalid JSON."
}

if ($null -eq $dataSources.value) {
  throw "The data-source inventory for '$sourceDataflowId' did not contain a value collection."
}

$upstreamJson = az rest --method get `
  --resource $pbiResource `
  --url "https://api.powerbi.com/v1.0/myorg/groups/$sourceWorkspaceId/dataflows/$sourceDataflowId/upstreamDataflows" `
  -o json

if ($LASTEXITCODE -ne 0 -or [string]::IsNullOrWhiteSpace($upstreamJson)) {
  throw "Could not inventory upstream dataflows for '$sourceDataflowId'."
}

try {
  $upstream = $upstreamJson | ConvertFrom-Json -ErrorAction Stop
}
catch {
  throw "The upstream inventory for '$sourceDataflowId' returned invalid JSON."
}

if ($null -eq $upstream.value) {
  throw "The upstream inventory for '$sourceDataflowId' did not contain a value collection."
}

$upstreamDependencies = @($upstream.value)
```

Preserve `$dataSources.value` and `$upstreamDependencies` in the migration plan. Use the upstream IDs to build the dependency order. Before any write, require each upstream dependency to map to an earlier candidate or record it as an approved external dependency.

## Target Naming

Allow Unicode letters, digits, and spaces in the target name. Replace ampersands with `and`, remove other punctuation and symbols, normalize whitespace, and show the proposed name before writing.

```powershell
$targetDataflowName = $sourceDataflow.name -replace '\s*&\s*', ' and '
$targetDataflowName = $targetDataflowName -replace '[^\p{L}\p{N}\s]', ' '
$targetDataflowName = $targetDataflowName -replace '\s+', ' '
$targetDataflowName = $targetDataflowName.Trim()

if ([string]::IsNullOrWhiteSpace($targetDataflowName)) {
  throw 'The source name contains no supported characters after sanitization.'
}
```

Require one proposed target name and confirm that both inventories lack it:

```powershell
$legacyTargetJson = az rest --method get `
  --resource $pbiResource `
  --url "https://api.powerbi.com/v1.0/myorg/groups/$targetWorkspaceId/dataflows" `
  -o json

if ($LASTEXITCODE -ne 0 -or [string]::IsNullOrWhiteSpace($legacyTargetJson)) {
  throw 'Could not inventory target dataflows through the Power BI REST API.'
}

try {
  $legacyTarget = $legacyTargetJson | ConvertFrom-Json -ErrorAction Stop
}
catch {
  throw 'The Power BI target inventory returned invalid JSON.'
}

if ($null -eq $legacyTarget.value) {
  throw 'The Power BI target inventory did not contain a value collection.'
}

$fabricItems = @()
$fabricUrl = "https://api.fabric.microsoft.com/v1/workspaces/$targetWorkspaceId/items?type=Dataflow"

do {
  $fabricPageJson = az rest --method get `
    --resource $fabricResource `
    --url $fabricUrl `
    -o json

  if ($LASTEXITCODE -ne 0 -or [string]::IsNullOrWhiteSpace($fabricPageJson)) {
    throw 'Could not inventory target dataflows through the Fabric REST API.'
  }

  try {
    $fabricPage = $fabricPageJson | ConvertFrom-Json -ErrorAction Stop
  }
  catch {
    throw 'The Fabric target inventory returned invalid JSON.'
  }

  if ($null -eq $fabricPage.value) {
    throw 'The Fabric target inventory did not contain a value collection.'
  }

  $fabricItems += @($fabricPage.value)
  $fabricUrl = $fabricPage.continuationUri
}
while (-not [string]::IsNullOrWhiteSpace($fabricUrl))

$nameExists = @($legacyTarget.value | Where-Object { $_.name -eq $targetDataflowName }).Count -gt 0 -or
  @($fabricItems | Where-Object { $_.displayName -eq $targetDataflowName }).Count -gt 0

if ($nameExists) {
  throw "The target already contains '$targetDataflowName'. Refusing a non-idempotent save-as operation."
}
```

For a bulk migration, require all sanitized target names to be unique before the first write. For example, `Sales & Ops` and `Sales and Ops` both produce `Sales and Ops`. Stop and ask for an explicit naming policy when sanitization creates a collision.

If the target workspace uses folders, record the desired folder. Do not claim that this API moves an existing converted item into it. The documented save-as request has no `folderId`, and public Fabric item APIs do not provide a supported move-to-folder action for an existing dataflow. Place the copy in the folder through the Fabric UI after creation.

## Phase 2: Convert One Dataflow

Write the request body to a file. On Windows, do not add `-o json` to the `saveAsNativeArtifact` command.

```powershell
$includeSchedule = $false # Set to $true only after the user explicitly requests schedule migration.

$requestBody = @{
  displayName = $targetDataflowName
  description = 'Migrated from Gen1 dataflow via saveAsNativeArtifact'
  includeSchedule = $includeSchedule
  targetWorkspaceId = $targetWorkspaceId
} | ConvertTo-Json -Compress

$requestFile = [System.IO.Path]::GetTempFileName()
$utf8WithoutBom = [System.Text.UTF8Encoding]::new($false)

try {
  [System.IO.File]::WriteAllText($requestFile, $requestBody, $utf8WithoutBom)

  $response = az rest --method post `
    --resource $pbiResource `
    --url "https://api.powerbi.com/v1.0/myorg/groups/$sourceWorkspaceId/dataflows/$sourceDataflowId/saveAsNativeArtifact" `
    --headers 'Content-Type=application/json' `
    --body "@$requestFile"

  if ($LASTEXITCODE -ne 0 -or [string]::IsNullOrWhiteSpace($response)) {
    throw 'The save-as request failed or returned an empty response. Check the target before any retry.'
  }

  try {
    $migration = $response | ConvertFrom-Json -ErrorAction Stop
  }
  catch {
    throw 'The save-as request returned invalid JSON. Check the target before any retry.'
  }

  $migration | Select-Object `
    @{ Name = 'newId'; Expression = { $_.artifactMetadata.objectId } }, `
    @{ Name = 'name'; Expression = { $_.artifactMetadata.displayName } }, `
    @{ Name = 'provisionState'; Expression = { $_.artifactMetadata.provisionState } }, `
    errors
}
finally {
  Remove-Item -LiteralPath $requestFile -Force -ErrorAction SilentlyContinue
}
```

Do not retry automatically if the call fails. Check the target for a partial item first. If none exists, report the API response and route the source dataflow to manual UI or template migration.

## Phase 3: Verify

Require an `Active` response before reporting success. Treat any other state as incomplete, preserve the returned target ID, and do not retry the non-idempotent operation:

```powershell
$provisionState = $migration.artifactMetadata.provisionState

if ($provisionState -ne 'Active') {
  throw "The target item returned provisioning state '$provisionState'. Do not retry. Investigate target item '$($migration.artifactMetadata.objectId)'."
}
```

Then verify the returned item with the Fabric REST API:

```powershell
$newItemId = $migration.artifactMetadata.objectId

$verifiedItemJson = az rest --method get `
  --resource $fabricResource `
  --url "https://api.fabric.microsoft.com/v1/workspaces/$targetWorkspaceId/items/$newItemId" `
  -o json

if ($LASTEXITCODE -ne 0 -or [string]::IsNullOrWhiteSpace($verifiedItemJson)) {
  throw "Could not verify target item '$newItemId' through the Fabric REST API. Do not retry the save-as operation."
}

$verifiedItem = $verifiedItemJson | ConvertFrom-Json -ErrorAction Stop
$verifiedItem | Select-Object id, displayName, type, workspaceId, description
```

Report the source ID, target ID, source and target names, provisioning state, errors, whether the request included the schedule, and the fact that no refresh ran. Instruct the user to validate or rebind connections before the first refresh. If the request included the schedule, verify the copied schedule in the Fabric UI and handle `FailedToCopySchedule` as manual remediation; this skill does not define a schedule-read API call. Otherwise, configure the schedule after connection validation.

## Bulk Workflow

1. Inventory all source Gen1 dataflows.
2. Build a migration plan with the source ID, source name, sanitized target name, source types, target conflict result, and plan status. Confirm that all proposed target names are unique.
3. Present the plan for approval before creating multiple items.
4. Convert serially in dependency order, with upstream dataflows before their consumers. Stop and resolve cycles or missing upstream mappings before writing. Check the target name before each request.
5. Persist the report after every item. Include `sourceId`, `sourceName`, `targetName`, `targetId`, `provisionState`, `errors`, `includeSchedule`, verification status, connection-remediation status, and schedule-remediation status.
6. Make reruns report-driven. If the report maps a source ID to a target ID, verify that ID and expected name. Skip it only when the item exists and the prior result is `Active`. Treat a target-name match without a verified report mapping as a conflict. If a response returned a target ID, investigate that item instead of repeating the POST.
7. Never delete the Gen1 sources as part of this workflow.

## Known Outcomes

| Outcome | Meaning | Next action |
|---|---|---|
| `provisionState: Active`, no errors | Artifact copy succeeded | Validate connections; verify a requested copied schedule or configure one if it was not copied |
| `provisionState: Active`, `ConnectionsUpdateFailed` | Artifact exists; in-place connection conversion was incomplete | Configure target connections before refresh |
| `InvalidRequest: Unexpected dataflow error` | Service rejected the save-as request | Confirm the sanitized name contains only letters, digits, and spaces; check for a partial copy, then use UI or template migration if the request still fails |
| Target name exists | Retrying would create or confuse duplicates | Stop and ask for an explicit name or conflict policy |
