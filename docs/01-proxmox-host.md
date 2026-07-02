# 1. Proxmox VE Host Setup

Proxmox VE is a free, open-source Type-1 hypervisor. It hosts every VM in this
lab.

> Alternative: if you can't install bare metal, VMware Workstation Player or
> Hyper-V on Windows work too — the VM guides still apply.

## Prerequisites

- A spare PC/laptop (or a nested VM) with virtualization (VT-x/AMD-V) enabled in
  BIOS, 16 GB+ RAM recommended, 100 GB+ disk.
- Proxmox VE ISO: https://www.proxmox.com/en/downloads
- A USB stick (8 GB+) and [Rufus](https://rufus.ie) or balenaEtcher to write it.

## Steps

1. **Write the ISO to USB** with Rufus (DD mode) and boot the target machine
   from it.
2. **Install Proxmox VE:** accept the license, choose the target disk, set
   country/timezone, a strong `root` password, and an admin email.
3. **Network:** assign a static management IP on your LAN, e.g.
   `192.168.10.5/24`, gateway `192.168.10.1`, DNS `192.168.10.1`.
   Record the hostname (e.g. `pve01.corp.local`).
4. **First boot → web UI:** browse to `https://192.168.10.5:8006` and log in as
   `root`. Dismiss the "no subscription" notice (the free repo is fine for a
   lab).
5. **Use the no-subscription repo** (optional but avoids update errors):
   *Datacenter → pve01 → Updates → Repositories* — disable the enterprise repo
   and add `pve-no-subscription`, then **Refresh** and **Upgrade**.
6. **Confirm the bridge** `vmbr0` exists under *pve01 → System → Network*. This
   is the virtual switch all VMs attach to.

## Uploading install ISOs

*pve01 → local (pve01) → ISO Images → Upload* — upload the Windows Server 2022
and Ubuntu Server ISOs here so you can attach them when creating VMs.

## Creating a VM (pattern used for every guest)

*Create VM* (top right), then:

- **General:** name (e.g. `DC01`).
- **OS:** select the uploaded ISO; for Windows also attach the VirtIO drivers ISO.
- **System:** BIOS default (OVMF/UEFI optional), QEMU Agent enabled.
- **Disk:** 60 GB (Windows) / 25 GB (Ubuntu), `virtio-scsi`.
- **CPU:** 2 cores. **Memory:** 4096 MB (Windows) / 2048 MB (Ubuntu).
- **Network:** bridge `vmbr0`, model VirtIO.

Start the VM and open its **Console** to run the OS installer.

## Verification

- Web UI reachable at `https://<host-ip>:8006`.
- `vmbr0` shows *Active: Yes*.
- A test VM boots to its installer from the attached ISO.

## Screenshots

- `screenshots/proxmox-summary.png` — datacenter summary
- `screenshots/proxmox-network.png` — vmbr0 configuration
- `screenshots/proxmox-create-vm.png` — VM creation wizard
