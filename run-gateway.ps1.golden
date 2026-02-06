$ErrorActionPreference = "Stop"
Set-StrictMode -Version Latest

Write-Host "=== OpenClaw Gateway bootstrap starting ==="
Write-Host "PowerShell version:" $PSVersionTable.PSVersion
Write-Host "Working dir:" (Get-Location)

# ---- CONFIG ----
$SecretId = "openclaw/prod/secrets"
$Region = "us-east-1"
$NodeExe = "C:\Program Files\nodejs\node.exe"
$OpenClaw = "C:\dev\openclaw\dist\index.js"
$Port = "18789"
# ----------------

Write-Host "Checking node executable..."
if (!(Test-Path $NodeExe)) {
  throw "Node.exe not found at $NodeExe"
}

Write-Host "Checking OpenClaw entrypoint..."
if (!(Test-Path $OpenClaw)) {
  throw "OpenClaw index.js not found at $OpenClaw"
}

Write-Host "Fetching secrets from AWS Secrets Manager..."
$secretString = & aws secretsmanager get-secret-value `
  --secret-id $SecretId `
  --region $Region `
  --query SecretString `
  --output text

if (-not $secretString) {
  throw "Failed to retrieve secrets from AWS Secrets Manager"
}

Write-Host "Parsing secrets..."
$secrets = $secretString | ConvertFrom-Json

Write-Host "Injecting secrets into process environment..."
foreach ($p in $secrets.PSObject.Properties) {
  $name = $p.Name
  $value = [string]$p.Value
  if ($value -and $value.Length -gt 0) {
    [Environment]::SetEnvironmentVariable($name, $value, "Process")
    Write-Host "  injected $name"
  }
}

Write-Host "Starting OpenClaw Gateway on port $Port..."
Write-Host "Command:"
Write-Host "  $NodeExe $OpenClaw gateway --port $Port"

& $NodeExe $OpenClaw gateway --port $Port

Write-Host "=== THIS LINE SHOULD NEVER PRINT ==="