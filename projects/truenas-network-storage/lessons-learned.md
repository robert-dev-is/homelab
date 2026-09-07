# Lessons Learned

## 1. Small Systems Can Support Serious Storage Architectures

A Mini PC does not have to be limited to USB storage. Native PCIe expansion through M.2 and OCuLink made it possible to attach a real SAS HBA and 10GbE NIC while keeping the compute system compact.

The result behaves much more like a conventional server-storage architecture than the physical form factor suggests.

## 2. Direct HBA Passthrough Creates a Clean Responsibility Boundary

Proxmox manages the VM. TrueNAS manages the disks.

Passing the complete HBA through to TrueNAS keeps ZFS close to the physical storage and avoids unnecessary virtual-disk abstraction.

## 3. Cooling PCIe Storage Hardware Matters

HBAs designed for server airflow can run very hot in unconventional enclosures. General chassis airflow was not enough to keep the HBA heatsink comfortable, while a small fan directly over the heatsink solved the problem simply and quietly.

## 4. Network Speed and Storage Speed Are Different Bottlenecks

The HDD mirror cannot sustain 10GbE by itself, but that does not make 10GbE unnecessary.

10GbE provides headroom for SSD storage, ZFS ARC, concurrent clients, and future expansion. It also prevents the network from artificially limiting storage that is capable of exceeding 2.5GbE.

## 5. SSDs Change More Than Sequential Transfer Rates

The SSD game pool improved random-access and small-file behavior in addition to raw transfer speed. This matters for workloads such as game installations that contain many files and perform frequent metadata operations.

## 6. Use the Native Protocol for the Client When Possible

Using NFS for Linux and SMB for Windows has proven simpler than forcing one protocol onto every platform.

The NAS can provide both while clients use the interface that fits their operating system naturally.

## 7. Not Every Storage Tier Needs the Same Redundancy

The primary HDD pool contains important bulk data and uses a ZFS mirror.

The game SSD pool currently has no redundancy because most of its contents are replaceable. The architecture reflects the value and recoverability of the data instead of applying one storage policy to every workload.

## 8. VM Memory Sizing Still Has to Respect the Hypervisor

Giving ZFS more memory is useful, but the TrueNAS VM cannot consume nearly all physical host RAM without consequences.

A larger allocation caused QEMU startup failures when the Proxmox host ran out of memory. A fixed 24GB allocation on the 32GB host provided a workable balance between TrueNAS ARC capacity and host overhead.

## 9. Real Workloads Are Better Than One Benchmark

The most useful validation came from normal use:

- media storage
- Proxmox backups
- Linux NFS mounts
- Windows SMB access
- large game transfers
- thousands-of-files workloads
- repeated host and enclosure power cycles

A design that survives those workloads reliably is more meaningful than a single peak benchmark result.

## 10. The Project Did Not Need to Be This Complicated — and That Was Part of the Point

A simpler solution could have stored games on local SSDs and used a conventional NAS for bulk files.

The value of the project was also the engineering process itself: experimenting with PCIe topology, OCuLink, HBA passthrough, ZFS, cooling, automatic enclosure power, NFS, SMB, and 10GbE in one working system.

That experimentation turned a storage upgrade into a reusable infrastructure project and a practical learning platform.
