# 💻 Ryzen APU iGPU Passthrough (Renoir / Cezanne)

A practical **KVM/QEMU/libvirt** guide to passing an AMD Ryzen APU's **Vega iGPU and HDMI/DP audio** through to a Windows VM, including ROM handling and host- and guest-side reset workflows.

The documented setups use a **Ryzen 3 PRO 4350G** (Renoir, Vega 6, device ID `0x1636`) and a **Ryzen 7 5700G** (Cezanne, Vega 8, `0x1638`). Other Ryzen 4000G/5000G models are related hardware, not a verified compatibility list.

Results depend on the exact **APU, motherboard, BIOS, kernel, firmware, and Windows driver** combination. This is a record of working configurations and troubleshooting, not a universal passthrough or reset guarantee.

- [🧪 Tested configurations and limitations](#-tested-configurations-and-limitations)
- [🧩 Architecture](#-architecture-high-level)
- [📋 Step-by-step instructions](#-step-by-step-instructions)
- [🔄 Windows reset automation](#-7-windows-resetdisable-automation)
- [🩺 Troubleshooting and historical results](#-troubleshooting-and-historical-results)

---

## ⚡ Quick overview

1. Enable **SVM and IOMMU** in the host firmware and verify suitable IOMMU groups.
2. On the documented kernel baseline, add the matching iGPU ID to `vendor-reset` and install it through DKMS.
3. If the kernel 6.12 build fails on the unaligned-header include, apply the documented `atom.c` fix.
4. Place compatible GPU VBIOS and audio ROM files in `/usr/share/vgabios`.
5. Add **both GPU and audio PCI functions** to the VM XML and attach their ROMs. Missing audio/ROM configuration was associated with Error 43 in these setups.
6. Use **OVMF/UEFI**; the guide includes the tested Secure Boot and Hyper-V/KVM settings, not a proven minimum configuration.
7. Install the tested AMD **Adrenalin 25.8.1 WHQL** Windows driver.
8. If needed, configure Windows reset/disable automation. It was required on the **5700G**, but not on the **4350G** in the documented setup.

Read the limitations and full instructions before changing the host. Keep a recovery path that does not depend on the passed-through GPU.

---

## 🧪 Tested configurations and limitations

| Configuration | Documented result |
| --- | --- |
| Debian 12, kernel **6.12.12**, patched `vendor-reset` | Reported stable in the original setup |
| Debian 13, kernel **6.12.73**, patched `vendor-reset` | Working fallback after later-kernel testing |
| Debian 13, kernel **6.12.94+**, patched `vendor-reset` | Also reported working in the latest repository notes |
| Kernels **6.16 / 6.17** | Reported unsuccessful; no working configuration documented |
| Kernels **6.18 / 6.19** | Reset problems and `AMD-Vi: IO_PAGE_FAULT` events reported; not a validated path for this guide |
| Windows driver **Adrenalin 25.8.1 WHQL** | Tested driver; reverting to it resolved the reported physical-console login crash |

Other **6.12.x kernels are likely to work as well**, since Linux 6.12 is an **LTS series** whose maintenance updates focus on bug and security fixes rather than new features. **No guarantee:** the versions listed above are observations from the author's machines, not validation of every 6.12.x release. Regressions and distribution-specific patches can still affect compatibility.

These observations also do not establish that `vendor-reset` universally stops working after 6.12. The later-kernel notes attributed the failures to changed reset behavior; a working way to substitute `vendor-reset` was not found in that setup.

### Troubleshooting history

After the Debian 13 upgrade on **10 January 2026**, the VM crashed during physical-console login and under gaming load. Downgrading libvirt, QEMU, OVMF, and the kernel to 6.12.12 did not resolve the crashes. Removing `firmware-amd-graphics` was also tried; it is not a general recommendation. Reverting the Windows driver to 25.8.1 fixed the login crash, but load-related crashes remained until a **faulty power supply** was identified as their cause.

Separate later-kernel testing still showed reset/IOMMU problems, so the documented working baseline returned to **6.12.73**, with **6.12.94** subsequently reported working as well. A representative log message was:

```text
vfio-pci 0000:06:00.0: AMD-Vi: Event logged [IO_PAGE_FAULT domain=0x0002 address=0x3682ff000 flags=0x0000]
```

An IOMMU fault alone does not establish the cause of a crash. Keep hardware faults, Windows-driver behavior, and host reset compatibility separate when troubleshooting.

---

## 🧩 Architecture (high-level)

![Architecture diagram](docs/architecture.svg)

---

## 💡 Why this exists

This guide brings together the steps and reset workarounds that worked on my machines, with links to the community discussions that informed them.

Ryzen APU iGPU passthrough often fails with:
- black screen
- **Error 43**
- reset issues after VM shutdown
- “works once” behavior

The goal is to make the setup and its limitations reproducible, including behavior across VM shutdowns and restarts.

---

## 📦 What this repo contains

### VBIOS / ROM files
- [`vbios_1636.dat`](vbios_1636.dat) — Renoir Vega 6, from the 4350G setup
- [`vbios_1638.dat`](vbios_1638.dat) — Cezanne Vega 8, from the 5700G setup
- [`ATIAudioDevice_AA01.rom`](ATIAudioDevice_AA01.rom) — HDMI/DP audio ROM used with both documented setups

> Include the **audio device and its ROM** in the documented configuration. ROM compatibility is firmware/platform-specific; a matching APU name alone is not a guarantee.

### Windows helper
- [`manage-gpu.bat`](manage-gpu.bat) — enables, disables, or cycles both GPU and audio using `pnputil`; device IDs must be adapted.
- [`Archive/reset_gpu.xml`](Archive/reset_gpu.xml) and [`Archive/disable_gpu.xml`](Archive/disable_gpu.xml) — Task Scheduler examples with machine-specific accounts and paths.

### Host helpers
- [`install_vendor-reset_module.sh`](install_vendor-reset_module.sh) — unloads the module, removes its DKMS registration, reinstalls it, and loads it again. The source path is hardcoded and must be adapted.
- [`add_new_device_id.sh`](add_new_device_id.sh) — optional source/udev patch and DKMS rebuild helper. It must be placed in the `vendor-reset` source root and assumes module version `0.1.1`; review its package-install and forced-rebuild actions before use.
- [`swag-gpu.py`](swag-gpu.py) — optional **host-side** libvirt helper to switch passthrough devices and VirtIO/VNC. It shuts down, redefines, and restarts the VM, with hardcoded PCI, USB, and ROM settings. Review and adapt it before use; it is not needed for the main guide.

---

## 📋 Step-by-step instructions

The workflow combines an experimental `AMD_NAVI10` device-ID mapping in **vendor-reset** with **Windows-side reset/disable automation** where needed. It is based in part on this [Proxmox community discussion](https://forum.proxmox.com/threads/amd-ryzen-5600g-igpu-code-43-error.138665/); the commands below target Debian with libvirt, not a complete Proxmox configuration.

Before proceeding, have KVM/QEMU/libvirt, DKMS, and headers matching the running kernel available. Back up the VM XML and keep independent host access. Stop any VM using the GPU before rebuilding or unloading its reset module.

## 🔧 1) Install and patch vendor-reset (Debian)

### 1.1 Clone vendor-reset

Source: [gnif/vendor-reset](https://github.com/gnif/vendor-reset). The device-ID changes below are local patches, not a statement of upstream APU support.

```bash
git clone https://github.com/gnif/vendor-reset
cd vendor-reset
```

### 1.2 Find your iGPU device ID
```bash
lspci -nkk
```

Renoir is typically `1636`, Cezanne is typically `1638`.

### 1.3 Add your APU iGPU to `src/device-db.h`

Edit `src/device-db.h` in the cloned `vendor-reset` source tree.

Find the `AMD_NAVI10` block and add your entry (example includes Renoir):

```c
#define _AMD_NAVI10(op) \
    {PCI_VENDOR_ID_ATI, 0x1636, op, DEVICE_INFO(AMD_NAVI10)}, \
    {PCI_VENDOR_ID_ATI, 0x7310, op, DEVICE_INFO(AMD_NAVI10)}, \
    {PCI_VENDOR_ID_ATI, 0x7312, op, DEVICE_INFO(AMD_NAVI10)}, \
```

For Cezanne add:
```c
{PCI_VENDOR_ID_ATI, 0x1638, op, DEVICE_INFO(AMD_NAVI10)}, \
```

### 1.4 Install with DKMS

Review [`install_vendor-reset_module.sh`](install_vendor-reset_module.sh) first. It unloads `vendor_reset`, removes `vendor-reset/0.1.1` from DKMS, installs from a **hardcoded source path**, and loads the module. Change that path to your clone and confirm the module version before using it; copying it into the source folder alone does not change the path.

After adapting the helper, copy it into the `vendor-reset` folder and run it there, with all GPU-using VMs stopped:

```bash
chmod +x install_vendor-reset_module.sh
./install_vendor-reset_module.sh
```

### 1.5 Kernel 6.12 build fix (if needed)

The Debian 12/13 notes document this include change for kernel 6.12. If your DKMS build fails because `asm/unaligned.h` is missing, patch:

- Edit: `src/amd/amdgpu/atom.c`
- Change:
```c
#include <asm/unaligned.h>
```
to:
```c
#include <linux/unaligned.h>
```

Then rerun the adapted installer:

```bash
./install_vendor-reset_module.sh
```

### 1.6 Load the module at boot
Note: the module name is `vendor_reset` (underscore, not dash).

Add it to `/etc/modules`:
```text
vendor_reset
```

Reboot.

---

## 🔗 2) IOMMU and GRUB

### 2.1 IOMMU groups
On the documented motherboard, the GPU was already in a suitable IOMMU group. Verify the grouping of both GPU and audio on your own system before assigning them to a VM.

If unrelated host devices share the group, this guide does not establish a safe passthrough configuration for that layout. ACS override is not covered here and should not be treated as equivalent to hardware isolation.

### 2.2 GRUB options
No additional GRUB options were needed on my **headless Debian 12 / OpenMediaVault host**; the iGPU appeared to be available without further changes. This is a host-specific observation, not a general GPU-binding recipe. If the host is using the iGPU, resolve that for your platform before proceeding rather than applying unrelated kernel parameters or removing firmware packages.

---

## 💾 3) VBIOS / ROM files

### 3.1 Extract your VBIOS and HDMI audio ROM
You can extract these from:
- your GPU / system firmware
- or from a UEFI/BIOS update file

The repository includes ROMs used in the 4350G and 5700G setups:
- `vbios_1636.dat` — Vega 6 (4350G)
- `vbios_1638.dat` — Vega 8 (5700G)
- `ATIAudioDevice_AA01.rom` — HDMI audio ROM used with both

Prefer ROMs extracted from your own firmware and verify compatibility before reusing these files, even with the same APU model.

> Include the **audio PCI function and audio ROM**, as well as the GPU ROM, when following this configuration. Error 43 can have other causes too.

### 3.2 My extraction method (BIOS update file)
I used:
- UBU tool: https://winraid.level1techs.com/t/tool-guide-news-uefi-bios-updater-ubu/3035 (UBU-1.80)
- Extract the archive, run `UBU.cmd`, provide the UEFI update file when prompted.

Then convert as described here:
- https://github.com/isc30/ryzen-gpu-passthrough-proxmox?tab=readme-ov-file#configuring-the-gpu-in-the-windows-vm
- https://github.com/isc30/ryzen-gpu-passthrough-proxmox/discussions/18#discussioncomment-8627679

### 3.3 Put ROM files where libvirt can load them (Debian)
From this repository's root, copy the compatible VBIOS/ROM files to the documented location:

```bash
sudo mkdir -p /usr/share/vgabios
sudo cp vbios_1636.dat vbios_1638.dat ATIAudioDevice_AA01.rom /usr/share/vgabios/
```

This path worked on the documented Debian host; other locations failed in that setup. Ensure libvirt/QEMU can read the files under the host's access-control policy.

---

## 🔌 4) Add the PCIe devices to your VM (libvirt domain.xml)

### 4.1 Add GPU + Audio hostdev entries
Edit your VM XML (`virsh edit <vmname>` or virt-manager → XML) and add BOTH devices.

The addresses below are **examples**, not device discovery. Set each `<source>` address to the actual host PCI function, choose the matching GPU ROM (`vbios_1638.dat` for the documented 5700G), and ensure the guest PCI addresses fit your VM's controller layout. Back up the XML before editing:

```xml
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
  </source>
  <rom file="/usr/share/vgabios/vbios_1636.dat"/>
  <address type="pci" domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
</hostdev>

<hostdev mode="subsystem" type="pci" managed="yes">
  <driver name="vfio"/>
  <source>
    <address domain="0x0000" bus="0x06" slot="0x00" function="0x1"/>
  </source>
  <rom file="/usr/share/vgabios/ATIAudioDevice_AA01.rom"/>
  <address type="pci" domain="0x0000" bus="0x09" slot="0x00" function="0x0"/>
</hostdev>
```

---

## 🧩 5) VM firmware + Windows compatibility options

### 5.1 OVMF/UEFI and Secure Boot

The documented VM uses **OVMF/UEFI with Secure Boot**. Secure Boot is part of the tested configuration, not established here as a universal requirement for APU passthrough. Check the firmware actually selected by virt-manager rather than relying on defaults.

Example: adapt the Q35 machine version, installed OVMF paths, and VM-specific NVRAM path to your host:
```xml
<os firmware='efi'>
  <type arch='x86_64' machine='pc-q35-10.0'>hvm</type>
  <firmware>
    <feature enabled='yes' name='enrolled-keys'/>
    <feature enabled='yes' name='secure-boot'/>
  </firmware>
  <loader readonly='yes' secure='yes' type='pflash' format='raw'>/usr/share/OVMF/OVMF_CODE_4M.ms.fd</loader>
  <nvram template='/usr/share/OVMF/OVMF_VARS_4M.ms.fd' templateFormat='raw' format='raw'>/var/lib/libvirt/qemu/nvram/Win11_VARS.fd</nvram>
  <bootmenu enable='no'/>
</os>
```

### 5.2 Hyper-V / KVM feature flags

These flags were enabled in the working VM. Their individual necessity was not tested; this is a configuration snapshot, not a required minimum. Merge it carefully with your existing VM settings. The Hyper-V vendor ID below is an example value:

```xml
<features>
  <acpi/>
  <apic/>
  <hyperv mode='custom'>
    <relaxed state='on'/>
    <vapic state='on'/>
    <spinlocks state='on' retries='8191'/>
    <vpindex state='on'/>
    <synic state='on'/>
    <stimer state='on'/>
    <reset state='on'/>
    <vendor_id state='on' value='Example12345'/>
    <frequencies state='on'/>
    <reenlightenment state='on'/>
    <tlbflush state='on'/>
    <ipi state='on'/>
  </hyperv>
  <kvm>
    <hidden state='on'/>
  </kvm>
  <vmport state='off'/>
  <smm state='on'/>
  <ioapic driver='qemu'/>
</features>
```

---

## 💻 6) Boot Windows + install AMD drivers

Boot the VM into Windows and install the manually downloaded [AMD Adrenalin 25.8.1 WHQL driver for Vega/Polaris, Windows 10/11](https://drivers.amd.com/drivers/whql-amd-software-adrenalin-edition-25.8.1-win10-win11-aug-vega-polaris.exe).

This is the version identified in the working notes; newer drivers are not validated by this guide. Reverting to 25.8.1 resolved the reported physical-console login crash, but did not fix the separate power-supply problem.

---

## 🔄 7) Windows reset/disable automation

This was required on my **5700G**; the documented **4350G** setup worked without it. Host-side `vendor-reset` alone was not enough for the 5700G workflow.

[`manage-gpu.bat`](manage-gpu.bat) uses `pnputil` to **enable**, **disable**, or **reset** both GPU and audio. Here, `/reset` means disable, wait five seconds, then enable; it is not a replacement for the host's PCI reset mechanism. Disabling the GPU interrupts its display output, so retain another access method.

### 7.1 Set the device hardware IDs

The current script passes `/deviceid` to `pnputil`, so use **hardware IDs**, not a full device-instance path:

1. In Windows Device Manager, open the GPU → **Properties → Details → Hardware Ids**.
2. Do the same for its audio device; the Windows audio ID may begin with `HDAUDIO`, not `PCI`.
3. Replace `GPU_ID` and `AUDIO_ID` in the script with the matching values from your VM. Do not copy the script's machine-specific defaults blindly.

GPU hardware-ID format (placeholder only): `PCI\VEN_1002&DEV_1638&SUBSYS_XXXXXXXX&REV_XX`.

Test the adapted script from an elevated command prompt before scheduling it:

```bat
manage-gpu.bat /reset
manage-gpu.bat /disable
manage-gpu.bat /enable
```

Check `C:\gpu_disable_log.txt` and `C:\gpu_enable_log.txt` for the `pnputil` results; `/reset` calls both actions and does not create a separate reset log. The batch file can finish with exit code zero even after device-command failures.

### 7.2 Adapt and enable scheduled tasks

Use [`Archive/reset_gpu.xml`](Archive/reset_gpu.xml) and [`Archive/disable_gpu.xml`](Archive/disable_gpu.xml) as examples, not ready-to-import tasks for every machine:

- Set your own account/security principal, script path, and elevated execution settings.
- The reset task runs `/reset` **at user logon**, not at host or VM startup.
- The disable task runs `/disable` on **System / User32 event 1074** (a requested shutdown or restart), not on every logout. It is disabled in the exported XML; enable it after adapting and testing it.
- Confirm both actions run on your guest. A forced shutdown or crash may bypass the disable task.

---

## 🔎 Keywords

Ryzen APU, iGPU passthrough, Vega 6, Vega 8, Renoir, Cezanne, KVM, QEMU, libvirt, VFIO, vendor-reset, Error 43, Windows VM, VBIOS, ROM, OVMF, IOMMU, DKMS.

---

## 🩺 Troubleshooting and historical results

Before considering the setup reliable, check display output, driver loading, GPU/audio operation, and repeated VM shutdown/start cycles without rebooting the host.

- **Error 43:** verify that both PCI functions and their compatible ROMs are configured. This is a documented failure mode, not the only possible cause.
- **Works once, then black screen:** verify the host reset module and, on the 5700G workflow, the Windows reset/disable tasks and their logs.
- **Crashes under load:** do not assume a reset failure. The documented load-related crashes were traced to a faulty power supply; kernel and driver symptoms need separate investigation.

A QEMU hook that removed the GPU/audio PCI devices through sysfs and rescanned the bus **did not work** in these tests. Device removal is not equivalent to the `vendor_reset` reset sequence.

The [archived guide](Archive/README.md), [test notes](Archive/Tests%20Results.md), and [reset-method logs](Archive/test_results_vendor-reset_methodes) preserve earlier attempts, including Error 31, black screens, and driver-install freezes while testing Vega, Polaris, and Navi reset handlers. They are historical records, not alternative validated recipes, and predate the working configuration summarized above.
