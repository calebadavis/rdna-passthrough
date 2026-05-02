# AMD RX 7900 XTX GPU Passthrough Setup Guide
## Host: Fedora 44, AMD Ryzen 9800X3D, Gigabyte Aorus Elite X870m WiFi7

---

## Overview

This guide documents the complete host-side setup for AMD GPU passthrough on Fedora
with an RX 7900 XTX (reference VBIOS). The setup requires a custom patched kernel to
work around a kernel bug where AMD GPU firmware flags (`force_iommu`/`require_direct`)
prevent VFIO container setup on kernel >= 6.10.

**Patch file:** `vfio-amd-passthrough.patch`  
**Kernel version:** 6.19.14-200.vfio_amd.fc44

---

## 1. BIOS/UEFI Settings

- **SVM Mode** → Enabled
- **IOMMU** → Enabled (AMD-Vi)
- **Above 4G Decoding** → Enabled
- **Resizable BAR (ReBAR)** → Enabled
- **Primary Display** → IGD/Integrated (host uses iGPU)

---

## 2. Kernel Boot Parameters

Add to `GRUB_CMDLINE_LINUX` in `/etc/default/grub`:

```
amd_iommu=on amd_iommu_force_isolation=0 vfio-pci.ids=1002:744c,1002:ab30,1002:7446,1002:7444
```

- `amd_iommu=on` — enable AMD IOMMU
- `amd_iommu_force_isolation=0` — disable strict IOMMU isolation check
- `vfio-pci.ids=...` — bind all 4 GPU functions to vfio-pci at boot

Regenerate GRUB config:
```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

---

## 3. Custom Patched Kernel

The kernel requires two patches to `drivers/iommu/iommu.c` and
`drivers/vfio/vfio_iommu_type1.c` to work around the `require_direct`/`force_iommu`
VBIOS flag issue.

### Building the kernel RPM

```bash
# Install build dependencies
sudo dnf install -y rpm-build make gcc flex bison openssl-devel \
  elfutils-libelf-devel bc dwarves python3-devel asciidoc xmlto python3-sphinx

# Set up rpmbuild tree
mkdir -p ~/rpmbuild/{SOURCES,SPECS,RPMS,BUILD}

# Download Fedora kernel SRPM
dnf download --source kernel

# Install SRPM
rpm -ivh kernel-*.src.rpm

# Copy patch file
cp vfio-amd-passthrough.patch ~/rpmbuild/SOURCES/

# Edit kernel.spec:
# 1. Add: %define buildid .vfio_amd  (after the commented-out buildid line)
# 2. Add: Patch2: vfio-amd-passthrough.patch  (after Patch1)
# 3. Add: ApplyOptionalPatch vfio-amd-passthrough.patch  (before linux-kernel-test line)

# Build
cd ~/rpmbuild
rpmbuild -bb --target=x86_64 \
  --without debug \
  --without debuginfo \
  --without perf \
  --without tools \
  --without selftests \
  --without cross_headers \
  --without ynl \
  SPECS/kernel.spec

# Install
sudo dnf install \
  ~/rpmbuild/RPMS/x86_64/kernel-*.vfio_amd.*.rpm \
  ~/rpmbuild/RPMS/x86_64/kernel-core-*.vfio_amd.*.rpm \
  ~/rpmbuild/RPMS/x86_64/kernel-modules-*.vfio_amd.*.rpm \
  ~/rpmbuild/RPMS/x86_64/kernel-modules-core-*.vfio_amd.*.rpm \
  ~/rpmbuild/RPMS/x86_64/kernel-modules-extra-*.vfio_amd.*.rpm
```

---

## 4. Systemd Service — GPU Function Binding

Functions `03:00.2` (USB) and `03:00.3` (Serial) need to be manually bound to
vfio-pci since they're driven by built-in kernel drivers that load before modprobe.

Create `/etc/systemd/system/vfio-bind-gpu.service`:

```ini
[Unit]
Description=Bind AMD GPU functions to vfio-pci and set IOMMU group type
After=systemd-udev-settle.service
Before=libvirtd.service virtqemud.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/sh -c '\
  echo "auto" > /sys/bus/pci/devices/0000:03:00.0/iommu_group/type; \
  echo "0000:03:00.2" > /sys/bus/pci/drivers/xhci_hcd/unbind 2>/dev/null || true; \
  echo "0000:03:00.3" > /sys/bus/pci/drivers/i2c-designware-pci/unbind 2>/dev/null || true; \
  echo "vfio-pci" > /sys/bus/pci/devices/0000:03:00.2/driver_override; \
  echo "vfio-pci" > /sys/bus/pci/devices/0000:03:00.3/driver_override; \
  echo "0000:03:00.2" > /sys/bus/pci/drivers/vfio-pci/bind 2>/dev/null || true; \
  echo "0000:03:00.3" > /sys/bus/pci/drivers/vfio-pci/bind 2>/dev/null || true'

[Install]
WantedBy=multi-user.target
```

Enable:
```bash
sudo systemctl enable --now vfio-bind-gpu.service
```

---

## 5. libvirt Configuration

Edit `/etc/libvirt/qemu.conf` and add:
```
iommufd = 0
```

This forces QEMU to use the legacy vfio-pci container interface instead of iommufd.

Restart libvirt:
```bash
sudo systemctl restart virtqemud virtnetworkd
```

---

## 6. virtqemud Memory Lock Limit

Create `/etc/systemd/system/virtqemud.service.d/override.conf`:

```ini
[Service]
LimitMEMLOCK=infinity
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart virtqemud
```

---

## 7. VBIOS ROM File

The GPU VBIOS ROM is required for OVMF/UEFI to initialize the GPU display output
(GOP) during VM boot. Download the matching VBIOS for your card from TechPowerUp
(search for RX 7900 XTX, VBIOS version `022.001.002.009.000001`).

Place in a QEMU-accessible location:
```bash
sudo cp AMD.RX7900XTX.24576.221115.rom /usr/share/qemu/
sudo chmod 644 /usr/share/qemu/AMD.RX7900XTX.24576.221115.rom
```

---

## 8. VM XML Configuration

Key elements of the libvirt VM definition:

```xml
<!-- UEFI firmware with secure boot -->
<os firmware="efi">
  <type arch="x86_64" machine="q35">hvm</type>
  <firmware>
    <feature enabled="yes" name="enrolled-keys"/>
    <feature enabled="yes" name="secure-boot"/>
  </firmware>
</os>

<!-- CPU passthrough with AMD SMT fix -->
<cpu mode="host-passthrough" check="none" migratable="off">
  <topology sockets="1" dies="1" clusters="1" cores="6" threads="2"/>
  <feature policy="require" name="topoext"/>
</cpu>

<!-- Hypervisor hiding -->
<features>
  <hyperv mode="custom">
    <vendor_id state="on" value="AuthenticAMD"/>
  </hyperv>
  <kvm><hidden state="on"/></kvm>
  <ioapic driver="qemu"/>
  <smm state="on"/>
</features>

<!-- GPU passthrough with VBIOS ROM -->
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x03" slot="0x00" function="0x0"/>
  </source>
  <rom bar="on" file="/usr/share/qemu/AMD.RX7900XTX.24576.221115.rom"/>
</hostdev>

<!-- GPU audio, USB controller, serial bus -->
<hostdev mode="subsystem" type="pci" managed="yes">
  <source><address domain="0x0000" bus="0x03" slot="0x00" function="0x1"/></source>
</hostdev>
<hostdev mode="subsystem" type="pci" managed="yes">
  <source><address domain="0x0000" bus="0x03" slot="0x00" function="0x2"/></source>
</hostdev>
<hostdev mode="subsystem" type="pci" managed="yes">
  <source><address domain="0x0000" bus="0x03" slot="0x00" function="0x3"/></source>
</hostdev>
```

---

## 9. Verification

After setup, verify all 4 GPU functions are bound to vfio-pci:

```bash
lspci -nnk | grep -A 2 "03:00"
# All should show: Kernel driver in use: vfio-pci
```

Check IOMMU groups:
```bash
for d in /sys/kernel/iommu_groups/*/devices/*; do
  n=${d#*/iommu_groups/*}; n=${n%%/*}
  printf 'IOMMU Group %s ' "$n"
  lspci -nns "${d##*/}"
done | grep "03:00"
```

Start VM:
```bash
sudo virsh start win11
```

---

## 10. The Kernel Patch — Technical Background

The RX 7900 XTX reference VBIOS sets a `force_iommu` flag that causes the kernel
to assign an identity IOMMU domain to the GPU and set `require_direct=1`. On
kernel >= 6.10, this prevents VFIO from setting up the container needed for
passthrough.

**Patches applied:**

1. `iommu.c`: Don't re-set `require_direct=1` from RESV_DIRECT regions
2. `iommu.c`: Skip identity mapping of address 0 (conflicts with VFIO DMA)  
3. `iommu.c`: Allow overriding identity domains with DMA_FQ domain type
4. `iommu.c`: Disable firmware 1:1 mapping rejection for VFIO/iommufd
5. `vfio_iommu_type1.c`: Skip RLIMIT_MEMLOCK check for multi-group passthrough
6. `vfio_iommu_type1.c`: Always allow memory locking for VFIO containers
7. `vfio_iommu_type1.c`: Don't exclude address 0 from IOVA range
8. `vfio_iommu_type1.c`: Clear `require_direct` before VFIO domain allocation

---

## License
Copyright (C) 2026 Caleb Davis
Licensed under the GNU General Public License v3 or later. See [LICENSE](LICENSE).
