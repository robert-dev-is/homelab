# Performance and Testing

## 2.5GbE Baseline

Before the 10GbE upgrade, the NAS regularly reached approximately 280 MB/s over 2.5GbE.

That result was close to the practical ceiling of a 2.5GbE connection and showed that the Proxmox bridge and virtualized TrueNAS network path were not major bottlenecks at that speed.

## HDD Pool Performance

Observed direct HDD-pool testing produced approximately:

| Test | Observed Result |
|---|---:|
| Sustained writes | ~185-200 MB/s |
| Uncached reads | ~275 MB/s |
| Cache-assisted reads | Higher, depending on ZFS ARC state |

ZFS ARC can make repeated reads appear significantly faster than the underlying disks. For that reason, cache state and test data must be considered when interpreting storage benchmarks.

## 10GbE Upgrade

The NAS was upgraded from 2.5GbE to 10GbE SFP+.

The goal was not necessarily to make the HDD mirror reach 10GbE speeds. The upgrade removed the network as the limiting factor for:

- SSD-backed storage
- ZFS ARC cached reads
- Concurrent clients
- Future storage expansion

## SSD Game Pool

A 2TB Samsung 860 QVO was added as an SSD-backed ZFS pool for game storage.

During a real-world transfer of an approximately 65GB game directory containing thousands of files from the NFS game share to local storage:

- Peak throughput reached roughly 350 MB/s.
- The entire transfer completed in under five minutes.

This exceeded the previous 2.5GbE ceiling and demonstrated that the 10GbE upgrade was providing useful headroom.

## Small Files vs Sequential Throughput

One of the most important observations from the project is that network throughput alone does not describe application performance.

A large sequential file transfer can pipeline I/O and use hundreds of megabytes per second. A workload that repeatedly opens, stats, or scans thousands of small files may instead become dominated by per-operation latency and metadata activity.

This is why storage architecture should be evaluated using real workloads in addition to synthetic throughput tests.

## Benchmarking Lesson: Compression Matters

An early ZFS `dd` test using zero-filled data produced unrealistically high throughput because the dataset compressed the zeros heavily.

Using incompressible/random test data provided results that were much closer to the actual physical storage performance.

This reinforced a useful benchmarking rule: understand what the filesystem, cache, compression layer, and operating system are doing before treating a single benchmark number as physical disk speed.
