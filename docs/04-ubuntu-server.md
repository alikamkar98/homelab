# 4. Ubuntu Server — Domain Join & Service

Add a Linux member to the lab: install Ubuntu Server, join it to `corp.local`,
and host a simple service (Nginx). Demonstrates cross-platform administration
and AD integration with `realmd`/`sssd`.

## Prerequisites

- VM `UBUNTU01` per [guide 1](01-proxmox-host.md).
- Ubuntu Server 22.04 LTS ISO: https://ubuntu.com/download/server
- Working `corp.local` domain and DNS on DC01 (192.168.10.10).

## Install Ubuntu Server

1. Boot the ISO, run the guided install, enable **OpenSSH server**.
2. Create an admin user (e.g. `labadmin`).
3. Set a static IP and point DNS at the DC (netplan):

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    ens18:
      addresses: [192.168.10.30/24]
      routes:
        - to: default
          via: 192.168.10.1
      nameservers:
        addresses: [192.168.10.10]   # DC01 (must resolve corp.local)
        search: [corp.local]
```

```bash
sudo netplan apply
ping -c1 dc01.corp.local          # verify DNS resolves the domain
```

## Join the domain

```bash
sudo apt update
sudo apt install -y realmd sssd sssd-tools adcli samba-common-bin \
    oddjob oddjob-mkhomedir packagekit krb5-user

# Discover and join (prompts for a domain admin password)
realm discover corp.local
sudo realm join --user=Administrator corp.local

# Verify the join
realm list
```

Enable home-directory creation on first login and allow AD logins:

```bash
sudo pam-auth-update --enable mkhomedir
# Optionally restrict logins to a group:
sudo realm permit -g Engineers@corp.local
```

Test with an AD account:

```bash
id alovelace@corp.local
su - alovelace@corp.local
```

## Host a service (Nginx)

```bash
sudo apt install -y nginx
echo "<h1>UBUNTU01 - homelab</h1>" | sudo tee /var/www/html/index.html
sudo systemctl enable --now nginx
curl -s http://localhost | grep homelab
```

Browse to `http://192.168.10.30` from another lab machine to confirm.

## Verification

- `realm list` shows `corp.local` and configured principals.
- `id <ADuser>@corp.local` returns AD UID/GID and group membership.
- Nginx page loads across the lab network.

## Screenshots

- `screenshots/ubuntu-netplan.png`
- `screenshots/ubuntu-realm-join.png`
- `screenshots/ubuntu-id-aduser.png`
- `screenshots/ubuntu-nginx.png`
