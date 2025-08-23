# drivers

**Out-of-tree Linux drivers for CopperlineOS hardware.**  
This repo contains the kernel modules, UAPI headers, DKMS packaging, and user‑space helpers needed to talk to Copperline hardware in Phase‑1+ (e.g., the `fpga-copper` timeline engine and `fpga-blitter` 2D DMA). It also includes udev rules and example programs.

> TL;DR: build once, `dkms install`, and you get `/dev/copperline-*` devices with a tiny ring/IOCTL API the daemons (`copperd`, `blitterd`) can use.

---

## Scope

- **Kernel modules** (out‑of‑tree):  
  - `copperline_copr` — PCIe driver for the **Copper** timeline engine.  
  - `copperline_bltr` — PCIe driver for the **Blitter** 2D DMA engine.
- **User‑space**: minimal libs for ring setup and examples.
- **Headers**: UAPI headers installed to `/usr/include/copperline/`.  
- **DKMS**: packaging for stable installs across kernel updates.  
- **udev/systemd**: group/permissions and optional autostart helpers.

Phase‑0 (Linux‑hosted) does **not** need custom kernel drivers; these are for Phase‑1 hardware bring‑up and beyond.

---

## Supported devices (initial / draft)

| Device | PCI IDs (example) | Node | Notes |
|---|---|---|---|
| Copper timeline (fpga‑copper) | `1D1D:00C0` | `/dev/copperline-copr0` | BAR0/1/2, MSI/MSI‑X |
| 2D Blitter (fpga‑blitter) | `1D1D:00B1` | `/dev/copperline-bltr0` | BAR0/1/2, MSI/MSI‑X |

*(IDs are placeholders until hardware settles.)*

---

## Repository layout

```
drivers/
├─ kernel/
│  ├─ copperline_copr/            # Copper PCIe driver
│  │  ├─ copr_main.c
│  │  ├─ copr_pci.c
│  │  ├─ copr_uapi.h              # shared with userspace
│  │  └─ Kbuild / Kconfig
│  ├─ copperline_bltr/            # Blitter PCIe driver
│  │  ├─ bltr_main.c
│  │  ├─ bltr_pci.c
│  │  ├─ bltr_uapi.h
│  │  └─ Kbuild / Kconfig
│  └─ common/                     # ring helpers, logging, pci utils
├─ uapi/                          # installed headers (copy of *_uapi.h)
├─ userspace/
│  ├─ libclring/                  # tiny C library to open/map rings
│  └─ examples/                   # ring setup demos
├─ dkms/                          # dkms.conf templates
├─ udev/                          # 70-copperline.rules
├─ systemd/                       # optional service units
└─ README.md
```

---

## Driver model (ring + IOCTL API)

The device exposes a **character device** and a small set of IOCTLs to set up DMA rings and enable interrupts. Large buffers are shared using the Linux DMA‑API and/or DMABUF fds.

### Common IOCTLs (draft)

```c
// ioctl numbers use 'C' (Copperline) as the magic
#define COPR_IOC_MAP_BAR0   _IOR('C', 0x01, struct cl_map)    // optional
#define COPR_IOC_RING_SETUP _IOW('C', 0x10, struct cl_ring)   // PR/WB heads/tails
#define COPR_IOC_IRQ_ARM    _IOW('C', 0x11, __u32 mask)       // MSI/MSI-X mask
#define COPR_IOC_WAIT       _IOWR('C', 0x12, struct cl_wait)  // poll for events
#define COPR_IOC_DMABUF_MAP _IOWR('C', 0x20, struct cl_dmabuf)// import DMABUF
```

Structures (simplified):

```c
struct cl_map   { __u64 addr; __u32 len; __u32 bar; };
struct cl_ring  { __u64 pr_head, pr_tail, pr_addr, pr_entries;
                  __u64 wb_head, wb_tail, wb_addr, wb_entries; };
struct cl_wait  { __u32 timeout_ms; __u32 events; };
struct cl_dmabuf{ __s32 fd; __u64 iova; __u32 len; __u32 flags; };
```

Each device defines its **own** ring layout:

- **copperline_copr**: program ring (host→device), write‑back ring (device→host) for IRQ/state/vsync.  
- **copperline_bltr**: job ring (host→device), write‑back ring for completions.

User‑space daemons (`copperd`, `blitterd`) write entries into the host→device ring and read device→host events by either `poll(2)` on the fd or `COPR_IOC_WAIT`.

---

## Build & install

### Prereqs

- Linux kernel headers matching your running kernel.  
- `make`, `gcc` (or `clang`), `dkms`, `pkg-config`.  
- For userspace lib/examples: `libudev`, `libdrm` (optional for DMABUF helpers).

### Option A: Build in‑tree (quick test)

```bash
make -C kernel/copperline_copr
sudo insmod kernel/copperline_copr/copperline_copr.ko

make -C kernel/copperline_bltr
sudo insmod kernel/copperline_bltr/copperline_bltr.ko
```

Unload:

```bash
sudo rmmod copperline_bltr copperline_copr
```

### Option B: DKMS install (persists across kernel updates)

```bash
sudo make dkms-install   # adds both modules as 'copperline/{copr,bltr}'
sudo modprobe copperline_copr
sudo modprobe copperline_bltr
```

Remove:

```bash
sudo make dkms-remove
```

---

## Device nodes & permissions

udev rule (`udev/70-copperline.rules`) creates nodes readable by group **copperline**:

```
KERNEL=="copperline-copr*", GROUP="copperline", MODE="0660"
KERNEL=="copperline-bltr*", GROUP="copperline", MODE="0660"
KERNEL=="renderD*", SUBSYSTEM=="drm", GROUP="copperline", MODE="0660"
```

Add your user to the group:

```bash
sudo groupadd -f copperline
sudo usermod -aG copperline $USER
# log out/in
```

---

## Example: open device and map rings

```c
#include <fcntl.h>
#include <sys/ioctl.h>
#include "uapi/copr_uapi.h"

int fd = open("/dev/copperline-copr0", O_RDWR | O_CLOEXEC);
struct cl_ring ring = {
  .pr_addr = /* mmap'd coherent buffer addr (iova) */,
  .pr_entries = 1024,
  .wb_addr = /* ... */,
  .wb_entries = 1024,
};
ioctl(fd, COPR_IOC_RING_SETUP, &ring);

/* then write program entries into PR and arm interrupts */
```

Full examples live in `userspace/examples/` (ping/pong, timing tests).

---

## Debugging

- `dmesg -w` while loading modules to see PCIe probe messages.  
- `lspci -nn` to verify device/vendor IDs.  
- Set `echo 1 | sudo tee /sys/module/copperline_copr/parameters/debug` to enable verbose logs (if implemented).  
- Use `tools/ringdump` (planned) to decode WB entries live.

---

## Compatibility

- Kernel baseline: **5.15+** (tested on Ubuntu 22.04/24.04, Debian 12, Fedora 40).  
- Architectures: **x86_64** first; aarch64 later (subject to IOMMU/PCIe availability).  
- IOMMU: drivers use DMA‑API; enable IOMMU (intel_iommu=on/amd_iommu=on) for safe mapping.

---

## Security & safety

- Drivers validate all ring indices and descriptor bounds.  
- DMABUF imports are restricted to **render nodes** and checked for size/format when possible.  
- Devices should run as unprivileged (group‑gated) with no `CAP_SYS_ADMIN` required.  
- MSI/MSI‑X interrupts are preferred over legacy INTx.

---

## Roadmap

- **v0.1**: PCIe probe, char devices, ring IOCTLs, MSI, basic examples.  
- **v0.2**: DMABUF import/export helpers, perf counters via sysfs, better tracing.  
- **v0.3**: AArch64 support, hot‑plug robustness, selftest module.  
- **v0.4**: Upstream‑ready patches where applicable.

Designs/specs live in [`CopperlineOS/docs`](https://github.com/CopperlineOS/docs) and RTL in [`fpga-copper`](https://github.com/CopperlineOS/fpga-copper) / [`fpga-blitter`](https://github.com/CopperlineOS/fpga-blitter).

---

## Contributing

- Keep patches focused; follow kernel coding style (`scripts/checkpatch.pl`).  
- Add KUnit or user‑space tests for new UAPI and ring semantics.  
- Include a short design note if you change IOCTLs or structures.

See `CONTRIBUTING.md` and `CODE_OF_CONDUCT.md`.

---

## License

Kernel modules are licensed **GPL‑2.0** (required by Linux); shared headers and userspace code are **Apache‑2.0 OR MIT**.

---

## See also

- RTL: [`fpga-copper`](https://github.com/CopperlineOS/fpga-copper) · [`fpga-blitter`](https://github.com/CopperlineOS/fpga-blitter)  
- Daemons: [`copperd`](https://github.com/CopperlineOS/copperd) · [`blitterd`](https://github.com/CopperlineOS/blitterd)  
- Protocol: [`ports`](https://github.com/CopperlineOS/ports) · Packaging: [`packaging`](https://github.com/CopperlineOS/packaging)
