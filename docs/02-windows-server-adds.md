# 2. Windows Server 2022 — Active Directory Domain Services

Install Windows Server, promote it to the first Domain Controller for the
`corp.local` forest, and create the OU/user/group structure used by the rest of
the lab.

## Prerequisites

- A VM (`DC01`) created per [guide 1](01-proxmox-host.md).
- Windows Server 2022 **evaluation** ISO (free, 180 days):
  https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022

## Install Windows Server

1. Boot the ISO, choose **Windows Server 2022 Standard (Desktop Experience)**.
2. Custom install to the virtio disk (load the VirtIO SCSI driver from the
   drivers ISO if the disk isn't listed).
3. Set the local Administrator password at first boot.
4. Install **QEMU guest agent** / VirtIO drivers so Proxmox sees the IP.

## Configure networking (before promotion)

A DC must have a **static IP** and point DNS at itself.

```powershell
# Run in an elevated PowerShell on DC01
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.10.10 `
    -PrefixLength 24 -DefaultGateway 192.168.10.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
Rename-Computer -NewName "DC01" -Restart
```

## Promote to a Domain Controller

```powershell
# Install the AD DS role
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# Create a brand-new forest: corp.local
Install-ADDSForest `
    -DomainName "corp.local" `
    -DomainNetbiosName "CORP" `
    -InstallDns `
    -ForestMode "WinThreshold" `
    -DomainMode "WinThreshold" `
    -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password") `
    -Force
```

The server reboots and comes up as `DC01.corp.local`, a Domain Controller with
integrated DNS.

## Create OUs, users, and groups

```powershell
$dn = "DC=corp,DC=local"

# Organizational Units
"Users","Engineering","Research","Finance","Disabled" | ForEach-Object {
    New-ADOrganizationalUnit -Name $_ -Path $dn -ProtectedFromAccidentalDeletion $true
}

# Security groups
New-ADGroup -Name "Engineers"   -GroupScope Global -Path "OU=Engineering,$dn"
New-ADGroup -Name "Researchers" -GroupScope Global -Path "OU=Research,$dn"
New-ADGroup -Name "Finance"     -GroupScope Global -Path "OU=Finance,$dn"

# Five test users (see the it-automation-toolkit repo for a bulk CSV version)
$pw = ConvertTo-SecureString "P@ssw0rd-Lab!" -AsPlainText -Force
$users = @(
    @{ First="Ada";       Last="Lovelace"; OU="Engineering"; Grp="Engineers"   }
    @{ First="Grace";     Last="Hopper";   OU="Engineering"; Grp="Engineers"   }
    @{ First="Alan";      Last="Turing";   OU="Research";    Grp="Researchers" }
    @{ First="Katherine"; Last="Johnson";  OU="Finance";     Grp="Finance"     }
    @{ First="Linus";     Last="Torvalds"; OU="Engineering"; Grp="Engineers"   }
)
foreach ($u in $users) {
    $sam = ($u.First.Substring(0,1) + $u.Last).ToLower()
    New-ADUser -Name "$($u.First) $($u.Last)" -GivenName $u.First -Surname $u.Last `
        -SamAccountName $sam -UserPrincipalName "$sam@corp.local" `
        -Path "OU=$($u.OU),$dn" -AccountPassword $pw -Enabled $true `
        -ChangePasswordAtLogon $true
    Add-ADGroupMember -Identity $u.Grp -Members $sam
}
```

## Verification

```powershell
Get-ADDomain | Select-Object Forest, DNSRoot, DomainMode
Get-ADOrganizationalUnit -Filter * | Select-Object Name
Get-ADUser -Filter * | Select-Object Name, SamAccountName
dcdiag /q          # should report no errors
```

## Screenshots

- `screenshots/adds-install-role.png`
- `screenshots/adds-promotion.png`
- `screenshots/aduc-ou-structure.png` — Active Directory Users and Computers
- `screenshots/dcdiag.png`
