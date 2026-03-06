# SONiC Generic AMD64 ONIE Rescue Live

## Overview

The **SONiC Rescue Live** artifact (`sonic-generic-rescue.bin`) boots a
fully in-memory SONiC rescue environment on generic amd64 hardware via ONIE.
No data is written to disk; all changes exist only in RAM and are lost on
reboot.

### What happens at boot

1. ONIE boots and runs the rescue installer (`sonic-generic-rescue.bin`).
2. `installer/install.sh` detects `demo_type=RESCUE` and **skips** all disk
   partitioning and file installation.
3. `fs.squashfs` is extracted from the installer payload into a tmpfs.
4. The squashfs is mounted read-only as the lower layer of an **overlayfs**.
5. A tmpfs is used as the writable upper layer (no persistence).
6. The rescue shell (`sonic-rescue`) is exec'd via `chroot` into the new root.
7. Kernel modules are loaded, network interfaces are brought up, and
   **dhclient** is run on each interface (best-effort, 10 s timeout per NIC).
8. An interactive shell is presented.

---

## Building the rescue artifact

```bash
# Configure the build for generic amd64 (if not already done)
make configure PLATFORM=generic

# Build only the rescue image
make target/sonic-generic-rescue.bin
```

The output file will be at:

```
target/sonic-generic-rescue.bin
```

---

## Booting from ONIE

### Method 1 – HTTP (most common)

On the ONIE shell, set the installer URL and trigger installation:

```bash
onie-nos-install http://<server>/sonic-generic-rescue.bin
```

### Method 2 – USB / local file

Copy `sonic-generic-rescue.bin` to a USB drive or any ONIE-accessible path,
then:

```bash
onie-nos-install file:///mnt/usb/sonic-generic-rescue.bin
```

> **Note:** In RESCUE mode the installer does **not** write to disk and does
> **not** set the system into NOS mode. ONIE remains in installer mode after
> the rescue shell exits.

---

## What to expect

- The root filesystem is the standard SONiC squashfs image, mounted
  **read-only**.
- A **tmpfs overlayfs** sits on top, so any writes (files, configs) survive
  only until the next reboot.
- All standard SONiC tools and utilities are available.
- The session begins in a Bash (or sh) rescue shell as root.

---

## Networking (DHCP)

On startup, `sonic-rescue` automatically attempts DHCP on every non-loopback
interface found in `/sys/class/net/` with a **10-second timeout per NIC**.

### Check current IP assignments

```bash
ip addr show
```

### Manually request DHCP on a specific interface

```bash
dhclient eth0
```

### Set a static IP address

```bash
ip addr add 192.168.1.100/24 dev eth0
ip route add default via 192.168.1.1
```

---

## Notes

- Changes are **not** persisted. The overlay lives entirely in RAM.
- Kernel modules for common amd64 NICs and storage controllers are loaded
  automatically (e1000e, igb, ixgbe, i40e, ice, mlx4/mlx5, virtio_net, etc.).
- To add more modules: `modprobe <module_name>`.
- If `fs.squashfs` cannot be found or mounted, the rescue setup will print a
  clear error message and abort (no silent failures).
- The rescue installer does **not** call `onie-nos-mode -s`, so the device
  remains in ONIE installer mode when the shell exits.
