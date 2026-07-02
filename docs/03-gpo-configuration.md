# 3. Group Policy Configuration

Two Group Policy Objects that show both a **security baseline** and a **user
experience** policy — the kind of thing asked about in interviews.

## Prerequisites

- Domain from [guide 2](02-windows-server-adds.md).
- Group Policy Management Console (installed with the AD DS management tools).

## GPO 1 — Password & account lockout policy

Enforced at the domain root so it applies to all users.

**GPMC path:** *Group Policy Management → corp.local → Default Domain Policy*
(or create a new GPO `Corp - Password Policy` linked at the domain).

*Computer Configuration → Policies → Windows Settings → Security Settings →
Account Policies*:

| Setting | Value |
| --- | --- |
| Enforce password history | 24 passwords |
| Maximum password age | 90 days |
| Minimum password length | 14 characters |
| Password must meet complexity requirements | Enabled |
| Account lockout threshold | 5 invalid attempts |
| Account lockout duration | 15 minutes |
| Reset lockout counter after | 15 minutes |

Create and link with PowerShell:

```powershell
Import-Module GroupPolicy
$gpo = New-GPO -Name "Corp - Password Policy"
New-GPLink -Name $gpo.DisplayName -Target "DC=corp,DC=local"
# Set values via the Security Settings UI in GPMC (registry-backed policies can
# be scripted with Set-GPRegistryValue; account policies are set in the editor).
```

## GPO 2 — Desktop / security baseline (linked to an OU)

Linked to `OU=Engineering` to show OU-scoped targeting.

**Create + link:**

```powershell
$gpo2 = New-GPO -Name "Corp - Engineering Baseline"
New-GPLink -Name $gpo2.DisplayName -Target "OU=Engineering,DC=corp,DC=local"
```

Settings (mix of registry-backed and UI):

| Area | Setting | Value |
| --- | --- | --- |
| Screen lock | Interactive logon: machine inactivity limit | 900 s |
| USB | Removable storage: deny write access | Enabled |
| Control Panel | Prohibit access to Control Panel/Settings | Enabled (users) |
| Mapped drive | Map `\\FS01\Shared` to `S:` | Preference item |

Example of a registry-backed setting via PowerShell:

```powershell
# Machine inactivity limit (seconds)
Set-GPRegistryValue -Name "Corp - Engineering Baseline" `
    -Key "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" `
    -ValueName "InactivityTimeoutSecs" -Type DWord -Value 900
```

## Applying and verifying

On a client (or DC01 for testing):

```powershell
gpupdate /force                       # pull latest policy
gpresult /r                           # show applied GPOs for the user/computer
gpresult /h gpreport.html; ii gpreport.html   # full HTML report
```

Confirm the password policy is enforced:

```powershell
Get-ADDefaultDomainPasswordPolicy |
    Select-Object MinPasswordLength, MaxPasswordAge, LockoutThreshold
```

## Screenshots

- `screenshots/gpo-password-policy.png`
- `screenshots/gpo-engineering-baseline.png`
- `screenshots/gpo-link-structure.png`
- `screenshots/gpresult.png`
