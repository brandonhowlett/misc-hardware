# Proxmox Headless EDID Fix (Intel iGPU)

Minimal, working steps to resolve:

`EDID block 0 is all zeroes`

---

## 1. Create a valid EDID

```bash
mkdir -p /lib/firmware/edid

printf '\x00\xff\xff\xff\xff\xff\xff\x00\x31\xd8\x00\x00\x00\x00\x00\x00\x05\x16\x01\x03\x6d\x32\x1c\x78\xea\x5e\xc0\xa4\x59\x4a\x98\x25\x20\x50\x54\x00\x00\x00\xd1\xc0\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x02\x3a\x80\x18\x71\x38\x2d\x40\x58\x2c\x45\x00\xf4\x19\x11\x00\x00\x1e\x00\x00\x00\xff\x00\x4c\x69\x6e\x75\x78\x20\x23\x30\x0a\x20\x20\x20\x20\x00\x00\x00\xfd\x00\x3b\x3d\x42\x44\x0f\x00\x0a\x20\x20\x20\x20\x20\x20\x00\x00\x00\xfc\x00\x4c\x69\x6e\x75\x78\x20\x46\x48\x44\x0a\x20\x20\x20\x00\x05' > /lib/firmware/edid/1920x1080.bin
```

---

## 2. Verify EDID

```bash
hexdump -C /lib/firmware/edid/1920x1080.bin | head
# Must start with: 00 ff ff ff ff ff ff 00

stat -c %s /lib/firmware/edid/1920x1080.bin
# Must be 128
```

---

## 3. Set kernel cmdline (connector-specific)

```bash
nano /etc/kernel/cmdline
```

Example:
```
root=ZFS=rpool/ROOT/pve-1 boot=zfs i915.fastboot=1 drm.edid_firmware=edid/1920x1080.bin video=card0-VGA-1:e
```

---

## 4. Rebuild boot artifacts

```bash
update-initramfs -u -k all
proxmox-boot-tool refresh
```

---

## 5. Reboot and validate

```bash
dmesg | grep -i edid
# No errors expected

hexdump -C /sys/class/drm/card0-VGA-1/edid | head
# Should not be all zeroes
```

---

## Notes

- EDID must exist in `/lib/firmware/edid/` and be included in initramfs  
- Connector-specific binding is required for reliability  
- Replace `HDMI-A-3` if your connector differs (`/sys/class/drm/`)
