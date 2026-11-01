# Nexus-OS
Thin bare-metal partitioning layer for running multiple unmodified operating systems with direct hardware access and fast no-reboot switching. Experimental research project.

# NEXUS

**A thin partitioning layer for running multiple unmodified operating systems on one PC, with direct hardware access and fast switching without a full restart.**

> **Status: early research and design. Nothing here is usable yet.**
> NEXUS touches PCIe devices, IOMMU settings, and disks. Experiments can crash your machine or corrupt data. Use a dedicated test machine, keep backups, and never run this on a computer you depend on.

## What is NEXUS?

NEXUS aims to let several real operating system installs (Windows, Linux, and others) share one physical computer without acting like a virtual machine manager.

- **Real installs.** Each OS lives on its own drives and can still boot on its own without NEXUS.
- **Real drivers, real hardware.** Devices are assigned directly to an OS. No emulated GPU, disk, or USB.
- **Near-native performance.** CPU cores, RAM, SSDs, and USB controllers are partitioned per OS.
- **Fast switching.** Move the display, GPU, and input from one OS to another in seconds instead of rebooting.

"No emulation" does not mean "no virtualization." Isolating unmodified operating systems requires CPU virtualization extensions (VT-x/AMD-V) and an IOMMU (VT-d/AMD-Vi). NEXUS aims for **zero device emulation, not zero virtualization**: a thin layer that partitions and assigns hardware and emulates almost nothing.

## Execution modes

| Mode | Description |
|---|---|
| **Concurrent** | Multiple OSes run at once, each with its own CPU cores, RAM, and assigned devices. |
| **Exclusive** | One OS is active. Others are asleep (ACPI S3/S4) and keep their state. |
| **Hybrid** | Some OSes run while others sleep. |

## How switching works

NEXUS does **not** try to capture and restore internal hardware state. That is not feasible for complex devices like GPUs. Instead, it relies on each OS's own sleep and resume support:

1. The active OS suspends itself (S3), and its drivers quiesce the devices.
2. NEXUS detaches the device from that OS (IOMMU mappings removed).
3. The host driver resets the device through the vendor's own reset path.
4. NEXUS attaches the device to the next OS.
5. The next OS wakes up, and its own driver re-initializes the device.
6. If a reset fails, only the affected OS is rebooted.

## GPU policy (planned)

| Policy | Behavior |
|---|---|
| **Locked** | The dGPU stays with one chosen OS. The other uses the iGPU. No resets needed. |
| **Auto** | The dGPU follows the visible OS. The other OS gets the iGPU. |
| **Manual** | The user chooses on each switch whether the dGPU moves. |

## Architecture

```
 Guest OSes (real installs, real drivers)
 ───────────────────────────────────────
 NEXUS layer
   Thin hypervisor (VT-x, EPT, VT-d)
   Device reset manager + per-device quirk database
   Resource allocator (cores, RAM, PCIe)
   Management OS + daemon + switch UI
 ───────────────────────────────────────
 Physical hardware (assigned per OS)
```

## Known hard problems

- **GPU reset and re-initialization**, especially AMD cards and single-GPU setups. This decides how good the final result can be.
- **Kernel-level anti-cheat** often detects hypervisors. Games that use it are considered unsupported until a vendor says otherwise.
- **Windows specifics:** VBS/HVCI, TPM, BitLocker, and activation behave differently under a hypervisor.
- **Shared platform hardware:** SMM, power management, the embedded controller, and chipset resources.
- **Consumer board limits:** IOMMU grouping, ACS, and a limited number of assignable USB controllers.
- **Hybrid CPUs** (P/E cores) need careful topology handling.

## Roadmap

- [ ] **Milestone 0:** Hardware feasibility study (IOMMU groups, device reset behavior, GPU reset cycle testing)
- [ ] **Milestone 1:** Architecture and technical specification
- [ ] **Milestone 2:** Concurrent proof of concept (Linux host, VFIO, CPU pinning)
- [ ] **Milestone 3:** Exclusive switching with NVMe and USB controllers
- [ ] **Milestone 4:** GPU handoff with the host-driver reset path
- [ ] **Milestone 5:** Windows guest support
- [ ] **Milestone 6:** Management daemon and GUI
- [ ] **Later:** Replace general-purpose components with purpose-built ones where measurements justify it

## Not in the first version

Hybrid mode, macOS, nested virtualization, TPM/BitLocker support, hot reallocation of cores or RAM between running guests, and AMD CPUs or laptops.

## Prior art and research

KVM/QEMU and VFIO, vendor-reset, Looking Glass, Jailhouse, Xen, Qubes OS, ACRN, Hedron/NOVA, Genode, rust-vmm, Cloud Hypervisor, crosvm, Bareflank, BitVisor.

## Contributing

Contributions are welcome, especially from people with experience in:

- Hypervisors and kernel development
- PCIe, IOMMU, and device reset behavior
- GPU drivers
- UEFI/ACPI
- Windows internals

Hardware test reports are very valuable. Please open an issue with your motherboard, CPU, GPU, and results.

## Transparency

NEXUS is built to be open and auditable. It is not designed to hide itself from anti-cheat or other security software, and it will not include features meant to do so.

## License

Apache-2.0
