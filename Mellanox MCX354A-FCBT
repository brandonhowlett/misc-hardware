# Mellanox MCX354A-FCBT — Ethernet Mode Setup
**Platform:** Proxmox VE | **Kernel:** 6.17+ | **Verified:** March 2026

---

## Before You Start

> **ConnectX-3 does not use `mlxconfig`.** That tool only supports ConnectX-4 and newer.  
> Port type on CX3 is set via a kernel module option. The procedure below is the only confirmed working method on kernel 6.17+.

This procedure requires **two reboots**.  
Do not put the MCX354A ports on your Proxmox management network — they will disappear during the blacklist phase.

---

## Step 1 — Install Packages

```bash
apt-get install -y proxmox-headers-$(uname -r) gcc make dkms unzip
```

> Use `proxmox-headers-$(uname -r)` — not `linux-headers-generic`. That package does not exist on Proxmox.

---

## Step 2 — Install MFT 4.34.1

MFT 4.27 fails to build on kernel 6.17. Use 4.34.1.

```bash
cd ~
wget https://content.mellanox.com/MFT/mft-4.34.1-10-x86_64-deb.tgz
tar xzf mft-4.34.1-10-x86_64-deb.tgz
cd mft-4.34.1-10-x86_64-deb
./install.sh
```

Verify the DKMS module built correctly:

```bash
dkms status
# Expected: kernel-mft-dkms/4.34.1, 6.17.x-pve, amd64: installed

find /lib/modules/$(uname -r) -name "mst_pci*"
# Expected: mst_pci.ko and mst_pciconf.ko both listed
```

---

## Step 3 — Blacklist mlx4 and Reboot *(Reboot 1 of 2)*

`mst` cannot access the device while `mlx4_core` holds it.

```bash
echo "blacklist mlx4_core" > /etc/modprobe.d/blacklist-mlx4.conf
echo "blacklist mlx4_en" >> /etc/modprobe.d/blacklist-mlx4.conf
update-initramfs -u
reboot
```

---

## Step 4 — Register the Device with MST

After reboot, run these commands **in exact order**. Each step is required.

```bash
# 1. Create the MST device directory — must exist before mst add will work
mkdir -p /dev/mst

# 2. Enable PCI memory regions on the card
setpci -s 02:00.0 COMMAND=0x02

# 3. Load MST kernel modules
modprobe mst_pci
modprobe mst_pciconf

# 4. Register the device — do NOT use mst start, use mst add
mst add 02:00.0

# 5. Verify device nodes were created
ls -la /dev/mst/
```

Expected output from `ls -la /dev/mst/`:
```
crw------- 1 root root 511, 0 ... mt4099_pciconf0
crw------- 1 root root 234, 0 ... mt4099_pci_cr0
```

> Replace `02:00.0` with your actual PCIe address from: `lspci | grep -i mellanox`

> **Why this order matters:**
> - `mkdir -p /dev/mst` must come first — `mst add` fails silently if the directory doesn't exist
> - `setpci COMMAND=0x02` must come before `modprobe` — PCI memory regions must be enabled before the kernel module can bind
> - Use `mst add` not `mst start` — `mst start` scans, finds nothing, and unloads the modules itself

---

## Step 5 — Update Firmware

Cards ship with firmware `2.34.5000` or `2.35.5100`. Update to `2.42.5000`.

```bash
cd ~
wget https://www.mellanox.com/downloads/firmware/fw-ConnectX3-rel-2_42_5000-MCX354A-FCB_A2-A5-FlexBoot-3.4.752.bin.zip
unzip fw-ConnectX3-rel-2_42_5000-MCX354A-FCB_A2-A5-FlexBoot-3.4.752.bin.zip

# Verify card identity before burning
flint -d /dev/mst/mt4099_pci_cr0 query
# Confirm: PSID = MT_1090120019

# Burn — type y when prompted, takes ~30 seconds
flint -d /dev/mst/mt4099_pci_cr0 \
  -i fw-ConnectX3-rel-2_42_5000-MCX354A-FCB_A2-A5-FlexBoot-3.4.752.bin \
  burn
```

Successful output ends with:
```
Burning FS2 FW image without signatures - OK
Restoring signature                     - OK
```

---

## Step 6 — Set Ethernet Mode and Reboot *(Reboot 2 of 2)*

```bash
# Set both ports to Ethernet permanently
echo "options mlx4_core port_type_array=2,2" > /etc/modprobe.d/mlx4_core.conf

# Remove the blacklist
rm /etc/modprobe.d/blacklist-mlx4.conf

# Rebuild initramfs and reboot
update-initramfs -u
reboot
```

---

## Step 7 — Verify

```bash
# Confirm ports loaded as Ethernet
dmesg | grep -i mlx4 | grep -i "port"
# Look for: Activating port:1 and Activating port:2

# Check interfaces
ip link show | grep enp2
# enp2s0 and enp2s0d1 should appear in DOWN state

# Once DAC cable is connected
ethtool enp2s0
# Expected: Speed: 40000Mb/s, Link detected: yes
```

`DOWN` state with no cable is correct — interfaces come up automatically when cables are inserted.

---

## Network Configuration

Add to `/etc/network/interfaces`:

```
# Port 1 — 40Gb direct fabric link
auto enp2s0
iface enp2s0 inet static
    address 10.40.0.x/30
    mtu 9000

# Port 2 — 10Gb to CRS354 via 655874-B21 QSA adapter
auto enp2s0d1
iface enp2s0d1 inet static
    address <your management IP>/<prefix>
    mtu 1500
```

---

## Hardware Notes

**655874-B21 QSA adapter** (HP P/N = Mellanox MAM1Q00A-QSA)  
Plugs into the QSFP+ port and accepts any standard SFP+ module or DAC cable. Use on Port 2 to connect to your existing 10Gb SFP+ switch plant.

**DAC cable**  
10Gtek 40GBASE-CR4 passive DAC 2m confirmed working. QSFP+ to QSFP+ — no adapter needed for CX3-to-CX3 direct connect. Passive DAC shows no link in IB mode — complete both reboots before connecting cables.

---

## What Does Not Work

| Approach | Why it fails |
|---|---|
| `mstflint` from Debian repos | `mstconfig` segfaults on CX3; `mlxconfig` not included |
| MFT 4.27.0 | DKMS build fails on kernel 6.17 |
| MFT 4.31.0 | Does not exist — 404 |
| `mst start` without blacklist | `mlx4_core` holds device; MST loads then unloads itself |
| `mst start` after blacklist | Scans, finds nothing, unloads modules — use `mst add` instead |
| Unbind `mlx4_core` via sysfs | Single PCI function; write returns `No such device` |
| `mlxconfig set LINK_TYPE_P1=2` | Unsupported device — CX3 not supported by mlxconfig |
| `flint set_key DEFAULT_PORT_TYPE 2` | Wrong syntax — requires at most 1 argument |
| `mst add` without `mkdir -p /dev/mst` first | Directory doesn't exist after reboot; device nodes cannot be created |
| `modprobe` before `setpci COMMAND=0x02` | PCI memory regions disabled; module cannot bind to device |
