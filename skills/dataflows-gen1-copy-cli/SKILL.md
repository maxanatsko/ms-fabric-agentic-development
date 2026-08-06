---
name: dataflows-gen1-copy-cli
description: "Use when: copying, cloning, duplicating, or bulk-copying Power BI Dataflow Gen1 artifacts between workspaces while preserving Gen1. Uses Azure CLI, PowerShell, and the Power BI REST export/import APIs. Covers Gen1 discovery, exclusions, target conflict checks, export cleanup, multipart model.json upload, import polling, and Gen1 verification. Do not use for Gen1-to-Gen2 migration or saveAsNativeArtifact. Triggers: copy Gen1 dataflow, clone Gen1 dataflow, move Gen1 dataflow between workspaces, export import dataflow JSON, Dataflow Gen1 copy with az cli."
argument-hint: "Source workspace ID or URL, target workspace ID or URL, and optional exclusions"
---

# Copy Power BI Dataflow Gen1 Between Workspaces

Copy Dataflow Gen1 definitions from one Power BI workspace to another while retaining Dataflow Gen1. This is an export/import clone workflow, not `saveAsNativeArtifact` and not a Gen2.1 migration.

## Scope and Boundaries

- Source dataflows must be confirmed with `generation: 1`.
- Create a new Gen1 dataflow in the target; the target has a new `objectId`.
- Leave source dataflows unchanged. Never delete a source as part of this workflow.
- Do not use `saveAsNativeArtifact`; it produces a Gen2.1 artifact.
- Do not refresh, alter credentials, configure gateways, or enable schedules unless the user separately and explicitly requests it.
- The copy does not transfer operational configuration reliably. Credentials, gateway mapping, privacy settings, schedules, downstream dependencies, and workspace-specific parameters need validation after the copy.

## Prerequisites

### Access

The signed-in delegated user must have access to both workspaces and enough Power BI permissions to read the source dataflow definition and create a dataflow in the target. Use delegated user authentication; do not attempt an unattended service-principal import.

### Tools

- Azure CLI (`az`) installed and authenticated with the target tenant.
- PowerShell 5.1 or later. PowerShell 7 is optional; this workflow does not depend on `Invoke-RestMethod -Form`.
- `curl.exe`, included with supported current Windows versions, for the required multipart file upload.

Verify the tools and identity before any API request:

```powershell
az --version
curl.exe --version
az account show --query "{user:user.name, tenant:tenantId, subscription:name}" -o json
```

If Azure CLI is absent on Windows, install it with:

```powershell
winget install --exact --id Microsoft.AzureCLI
```

Sign in interactively if needed:

```powershell
az login
```

Never request or capture passwords, access tokens, secrets, or connection credentials in chat.

## API Setup

Use the Power BI REST API audience for every `az rest` call:

```powershell
$sourceWorkspaceId = '<source-workspace-guid>'
$targetWorkspaceId = '<target-workspace-guid>'
$pbiResource = 'https://analysis.windows.net/powerbi/api'
```

A workspace URL such as `https://app.powerbi.com/groups/<workspace-id>/list` contains the required workspace GUID.

## Phase 1: Inventory and Plan

List both workspaces before writing anything. This establishes which source artifacts are Gen1 and prevents target-name duplicates.

```powershell
$sourceDataflows = az rest --method get `
  --resource $pbiResource `
  --url "https://api.powerbi.com/v1.0/myorg/groups/$sourceWorkspaceId/dataflows" `
  -o json | ConvertFrom-Json

$targetDataflows = az rest --method get `
  --resource $pbiResource `
  --url "https://api.powerbi.com/v1.0/myorg/groups/$targetWorkspaceId/dataflows" `
  -o json | ConvertFrom-Json

$sourceDataflows.value |
  Select-Object name, objectId, generation, configuredBy

$targetDataflows.value |
  Select-Object name, objectId, generation, configuredBy
```

Select only Gen1 candidates and apply user-requested exclusions exactly by name. For example, exclude `AR Comments`:

```powershell
$candidates = $sourceDataflows.value |
  Where-Object {
    $_.generation -eq 1 -and
    $_.name -ne 'AR Comments'
  }
```

### Decision Rules

- If a source item is not `generation: 1`, stop for that item. Do not substitute a Gen2 endpoint or convert it.
- If a target already contains the source name, do not upload it. Report it as skipped and ask the user to choose a conflict policy.
- If the user asked for a bulk copy, show the candidate plan and obtain approval before target writes unless approval was explicit in the request.
- Preserve source names. Unlike `saveAsNativeArtifact`, this import workflow supports names containing `&`.

Create a plan with conflict status:

```powershell
$plan = foreach ($candidate in $candidates) {
  [PSCustomObject]@{
    Name = $candidate.name
    SourceId = $candidate.objectId
    Generation = $candidate.generation
    TargetExists = $candidate.name -in $targetDataflows.value.name
    Action = if ($candidate.name -in $targetDataflows.value.name) { 'Skip: target name exists' } else { 'Copy' }
  }
}

$plan | Format-Table -AutoSize
```

## Phase 2: Copy One Gen1 Dataflow

### 1. Export the Gen1 definition

```powershell
$sourceDataflowId = '<source-dataflow-guid>'
$definition = az rest --method get `
  --resource $pbiResource `
  --url "https://api.powerbi.com/v1.0/myorg/groups/$sourceWorkspaceId/dataflows/$sourceDataflowId" `
  -o json | ConvertFrom-Json
```

### 2. Remove persisted partitions

Exports contain snapshot `partitions` metadata tied to the source storage. The import API can reject these with `CdsaZeroPartitionsRequiredOnPut`. Remove `partitions` from every exported entity while preserving the Power Query logic and all other properties:

```powershell
$definition.entities | ForEach-Object {
  $_.PSObject.Properties.Remove('partitions')
}
```

Start with this minimal normalization. Only consider further changes in response to a specific import error, and report that error before changing the definition more broadly.

### 3. Upload `model.json` as multipart form data

Use `az` to obtain a Power BI delegated token, then use `curl.exe` for the API's multipart form-data upload. `az rest` is used for Azure-authenticated REST discovery but is not suitable for this multipart upload.

```powershell
$tempFile = Join-Path $env:TEMP 'model.json'
$responseFile = Join-Path $env:TEMP 'dataflow-import-response.json'

$definition | ConvertTo-Json -Depth 100 -Compress |
  Set-Content -LiteralPath $tempFile -Encoding utf8 -NoNewline

$token = az account get-access-token `
  --resource $pbiResource `
  --query accessToken -o tsv

$importUrl = "https://api.powerbi.com/v1.0/myorg/groups/$targetWorkspaceId/imports?datasetDisplayName=model.json&nameConflict=Abort"

try {
  $httpStatus = curl.exe --silent --show-error `
    --output $responseFile `
    --write-out '%{http_code}' `
    -X POST `
    -H "Authorization: Bearer $token" `
    -F "file=@$tempFile;type=application/json" `
    $importUrl

  $importResponse = Get-Content -LiteralPath $responseFile -Raw | ConvertFrom-Json

  if ($httpStatus -ne '202' -or $null -eq $importResponse.id) {
    throw "Import submission failed (HTTP $httpStatus): $($importResponse | ConvertTo-Json -Compress)"
  }

  $importId = $importResponse.id
}
finally {
  Remove-Item -LiteralPath $tempFile, $responseFile -Force -ErrorAction SilentlyContinue
}
```

Do not retry a failed or ambiguous submission automatically. First list the target and determine whether the named item was created.

## Phase 3: Validate the Import

The Import API returns `202 Accepted`; it is asynchronous. Poll the import operation until it reaches `Succeeded` or `Failed`.

```powershell
$import = az rest --method get `
  --resource $pbiResource `
  --url "https://api.powerbi.com/v1.0/myorg/groups/$targetWorkspaceId/imports/$importId" `
  -o json | ConvertFrom-Json

$import | Select-Object id, name, importState, error, dataflows
```

- If `importState` is `Failed`, report `error.code` and stop for that dataflow.
- If it is still pending, poll again with a bounded retry loop; do not submit another upload.
- If it is `Succeeded`, verify the target dataflow list. The final list is authoritative; the import response can report an internal generation value that differs from the user-facing dataflow listing.

```powershell
$targetAfterImport = az rest --method get `
  --resource $pbiResource `
  --url "https://api.powerbi.com/v1.0/myorg/groups/$targetWorkspaceId/dataflows" `
  -o json | ConvertFrom-Json

$targetAfterImport.value |
  Where-Object { $_.name -eq $definition.name } |
  Select-Object name, objectId, generation, configuredBy
```

A completed copy must appear in this listing with `generation: 1`.

## Phase 4: Bulk Copy

1. Inventory source and target once, then build the planned name list.
2. Exclude non-Gen1 sources and user-specified names.
3. Skip existing target names. Do not overwrite or generate alternate names without explicit approval.
4. Process candidates sequentially: export, remove `partitions`, upload, poll, verify final `generation: 1`.
5. Before continuing after an interrupted run, refresh the target inventory and copy only still-missing names.
6. Produce a report containing source name and ID, target name and ID, import ID, final state, and any error.

Recommended result object:

```powershell
[PSCustomObject]@{
  Name = $sourceItem.name
  SourceId = $sourceItem.objectId
  TargetId = $targetItem.objectId
  ImportId = $importId
  ImportState = $import.importState
  Generation = $targetItem.generation
  Result = 'Copied'
}
```

## Troubleshooting

| Symptom | Cause | Response |
|---|---|---|
| `CdsaZeroPartitionsRequiredOnPut` | Export includes persisted source snapshot partitions | Remove `partitions` from every entity, then submit once after confirming no target item exists. |
| HTTP `202` but no target item yet | Import is asynchronous | Poll the import ID; do not re-upload. |
| Target already has the name | Copy would conflict or duplicate intent | Skip it; ask the user whether to keep, replace manually, or use a different name. |
| `401` or `403` | Wrong API audience, expired login, or insufficient workspace rights | Use the Power BI audience, run `az login` again if needed, and verify source/target permissions. |
| Import fails after normalization | Definition has a dataflow-specific import incompatibility | Report the complete import error and preserve the source. Use the Power BI UI's Export JSON/Import workflow only after user approval. |

## Completion Criteria

Report the following for every requested non-excluded source:

- Source name and ID.
- Whether it was copied, skipped, or failed.
- Target name and ID for copied items.
- The target list confirms `generation: 1`.
- `AR Comments` or other requested exclusions were not copied.
- No source was changed or deleted.
- No refresh was executed.
- Follow-up required: credentials, gateways, schedules, parameters, and dependencies must be validated before the first refresh.
