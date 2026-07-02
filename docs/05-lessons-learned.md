# 5. Lessons Learned

A running log of what broke, why, and how I fixed it — the most useful part of
any homelab for interviews. Replace/extend these with your own real notes as you
build.

## What went wrong (and the fix)

| Problem | Root cause | Fix |
| --- | --- | --- |
| Windows installer didn't see the disk | VirtIO SCSI driver not loaded | Attached the VirtIO drivers ISO and loaded the storage driver during setup |
| `Install-ADDSForest` warned about DNS delegation | No parent DNS zone (expected in a lab) | Safe to ignore — the warning doesn't block promotion |
| Ubuntu `realm join` failed with a clock-skew error | Kerberos requires time sync within 5 min | Pointed NTP at the DC / gateway; re-ran the join |
| AD users couldn't log into Ubuntu | Logins not permitted by default after join | `realm permit -g Engineers@corp.local` |
| GPO didn't apply to a client | Slow replication / cached policy | `gpupdate /force`, then verified with `gpresult /r` |
| DC couldn't resolve external names | DNS forwarders not set | Added the router as a forwarder in DNS Manager |

## Key concepts I can now explain

- **Why a DC points DNS at itself:** AD is built on DNS; clients find domain
  services via SRV records the DC hosts.
- **OU vs. group:** OUs are for *organization and GPO targeting*; groups are for
  *permissions/membership*. They solve different problems.
- **GPO precedence (LSDOU):** Local → Site → Domain → OU, with the last writer
  winning unless *Enforced*/*Block Inheritance* changes it.
- **Kerberos and time:** tickets are time-stamped, so >5 min skew breaks auth —
  hence the Ubuntu join failure.
- **realmd/SSSD:** how Linux delegates authentication to AD without a local
  account per user.

## What I'd do differently next time

- Snapshot each VM right after a clean install to speed up rebuilds.
- Use a dedicated `Disabled` OU + the `Get-InactiveADAccounts.ps1` script from
  the [it-automation-toolkit](https://github.com/) repo for account hygiene.
- Automate user creation with `New-BulkADUsers.ps1` instead of the inline loop.
- Add the [lab-monitoring](https://github.com/) stack earlier so I have metrics
  while building, not after.

## Interview talking points

- "Walk me through building an AD domain from scratch" → guides 1–2.
- "How does Group Policy precedence work?" → LSDOU + the two GPOs I built.
- "How would you integrate Linux with AD?" → guide 4 (realmd/SSSD/Kerberos).
- "How do you find and clean up stale accounts?" → the inactive-accounts script.
