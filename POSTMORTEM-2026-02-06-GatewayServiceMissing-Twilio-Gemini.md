# OpenClaw Gateway Post-Mortem (Windows EC2) — “Gateway service missing” + Twilio/Gemini chaos

Date: 2026-02-06  
Host: Windows on EC2  
OpenClaw: 2026.1.30

## Summary
The OpenClaw gateway became unrestartable via `pnpm openclaw gateway restart` because the Windows Scheduled Task (“OpenClaw Gateway”) was deleted. The gateway *could* still run manually via `run-gateway.ps1`, but the CLI restart/status workflow depends on the Scheduled Task existing.

Separately, Twilio environment variables appeared to “still exist” due to secret injection and process/session environment behavior, not because OpenClaw JSON contained Twilio config.

Gemini config attempts failed due to schema mismatches / unsupported keys in the current OpenClaw version for Google provider configuration (as evidenced by strict config validation errors).

## What we were trying to do
1) Make Telegram group replies work more broadly (no mention requirement; allow new groups by default).  
2) Remove Twilio and stop related startup issues / confusion.  
3) Add Gemini 2.5 Flash for image understanding (either directly or via sidecar).  
4) Keep the gateway stable and restartable.

## What we did (timeline)
### Twilio “removal”
- Searched OpenClaw home config files for `twilio|TWILIO_` — found nothing.
- Observed TWILIO_* environment variables existed in the PowerShell session.
- Removed user and machine env vars:
  - `SetEnvironmentVariable(..., $null, "User")`
  - `SetEnvironmentVariable(..., $null, "Machine")`
- Confirmed a *new* PowerShell session showed no TWILIO env vars:
  - `Get-ChildItem Env: | Where-Object { $_.Name -match 'TWILIO' }` -> empty

Key insight:
- Some values were being injected from AWS Secrets Manager into the running gateway process.
- Environment variables can appear in one session and not another; always validate in a fresh shell.

### Telegram config validation errors
- Attempted keys like `groupPolicy: "allow"` or `requireMention` under `channels.telegram`.
- OpenClaw rejected invalid schema options:
  - `groupPolicy` must be one of: `"open" | "disabled" | "allowlist"`
  - `requireMention` was not recognized at the location we tried

Key insight:
- OpenClaw enforces strict JSON schema; unsupported keys will prevent config reload and/or startup.

### Gemini provider config failures
- Added `GEMINI_API_KEY` to AWS Secrets Manager.
- OpenClaw logs showed config validation failures such as:
  - `models.providers.google.baseUrl: expected string, received undefined`
  - `models.providers.google.models: expected array, received undefined`
  - `Unrecognized key: "provider"`
  - `models.providers.google.models.0: Unrecognized key: "capabilities"`

Key insight:
- The Google provider schema expected fields we didn’t provide and rejected fields we attempted.
- The current OpenClaw version appears to not accept the provider config format we tried.

### Gateway “RPC probe failed” vs reality
- Observed `gateway status` reporting RPC probe failures even while:
  - `netstat` showed LISTENING on 18789
  - `curl http://127.0.0.1:18789/` returned the control UI HTML

Key insight:
- “RPC probe failed” can happen even if the HTTP UI responds, depending on websocket/RPC readiness.
- Manual runs via `run-gateway.ps1` produced a clean “listening” state.

### The actual failure: “Gateway service missing”
- We had previously deleted the Scheduled Task:
  - `schtasks /Delete /TN "OpenClaw Gateway" /F`
- Later, `pnpm openclaw gateway restart` failed with:
  - `Gateway service missing. Start with: openclaw gateway install`

Fix:
- Reinstalled and restarted the Scheduled Task:
  - `pnpm openclaw gateway install`
  - `pnpm openclaw gateway start`
  - `pnpm openclaw gateway status` -> RPC probe ok

## Root cause
The gateway restart workflow failed because the Windows Scheduled Task that OpenClaw expects to manage the gateway was deleted. The CLI can’t restart a service that doesn’t exist.

## Contributing causes
- Confusing split between:
  - JSON config validation (hard fail)
  - Environment variables (per-session vs injected)
  - AWS Secrets Manager injection (process-time)
  - Telegram/BotFather settings (group privacy/permissions)
- Strict schema enforcement causing repeated invalid-config reload skips.

## Resolution (known-good commands)
Reinstall service:
```powershell
cd C:\dev\openclaw
pnpm openclaw gateway install
pnpm openclaw gateway start
pnpm openclaw gateway status


IMPORTANT:
Feb 7 follow-up: Secrets not updating because PID didn’t change

---

## Follow-up (2026-02-07): Why “new secrets” weren’t visible after restart

### Symptom
- Added new keys in AWS Secrets Manager
- Ran `pnpm openclaw gateway restart` or `schtasks /End` + `schtasks /Run`
- Alfred still couldn’t see the new secrets

### Root cause
AWS secrets are injected into the **process environment at gateway start** (via `run-gateway.ps1`).
If the **existing node.exe process remains alive** and keeps port `18789` bound, you did not actually restart the gateway process — so it continues using the old environment.

### Proof / mental model
> Secrets are loaded at process start. If the PID didn’t change, your secrets didn’t change.

### Fix
Ensure the old process is gone (PID on 18789 changes). In our setup, `run-gateway.ps1` now force-kills any PID holding port 18789 before launching the gateway, which makes restarts reliably pick up new AWS secrets.
