# ESXi Migration: Or How I Learned to Stop Worrying and Check My NFS Config

> **TL;DR**: I spent days thinking my HDD was too slow for ESXi VM migration. Turns out I just forgot to reload my NFS config. Don't be like me. Benchmarks inside.

| | |
|---|---|
| 📄 **[Full styled article](https://cipeople.github.io/esx-migration/esxi-migration-guide.html)** | HTML version with syntax highlighting |
| 📊 **[Benchmark charts](https://cipeople.github.io/esx-migration/benchmark-results.html)** | Interactive 16-run matrix, sync penalty, IOPS |
| 💰 **[Advanced deep dive](https://vladster601.gumroad.com/l/selpp)** ($9) | tmpfs target, fan-out, vmkfstools ceiling |

---

> **Units:** all sizes/speeds use binary units (powers of two). In this doc: 1 MB = 2^20 bytes, 1 GB = 2^30 bytes. (No marketing units.)

---

## Why This Guide Exists (And Why You Can't Just dd Your Way Out)

**"Can't I just mount the VMFS volume on Linux and copy the files?"**

No. Here's why you're stuck with this approach:

### The Fundamental Problems

1. **No supported way to mount VMFS6 on Linux**

   - VMware doesn't ship a Linux VMFS6 driver, and there's no in-kernel support
   - Third-party projects exist (e.g., vmfs6-tools / vmfs-fuse), but they're typically **read-only and fragile** -- fine for last-ditch recovery, not a predictable migration runbook
2. **ESXi won't use "normal Linux filesystems" as datastores**

   - For datastores: **VMFS** (block) or **NFS** (file)
   - Removable media is a different story (VFAT/FAT32; sometimes ext3 via CLI), but it's not a great place to park large VMs
3. **Cloning the disk doesn't help**

   - `dd if=/dev/esxi-disk of=/dev/new-disk` copies the VMFS filesystem
   - Linux still can't use it
   - You're back to square one
4. **Passthrough doesn't magically solve it**

   - RDM / passthrough gives a Linux VM the block device
   - You're still stuck at "how do I read VMFS6?"

### The Practical Approach (If You Don't Have vCenter)

**Mount an NFS datastore on ESXi -> Use vmkfstools to copy VMDKs off VMFS**

This works because:

- ESXi can read its own VMFS (obviously)
- ESXi can write to NFS shares
- vmkfstools runs inside ESXi and understands VMDK formats
- Linux can mount NFS and access the copied files
- You can then convert VMDK -> qcow2/raw on Linux with qemu-img

**Alternative scenarios:**

- *Fresh ESXi install from USB*: If you only have the datastore disk, boot ESXi from USB, add NFS storage, copy VMs out (no license needed for this)
- *USB disk as scratch target*: Removable-media support on ESXi is limited (VFAT/FAT32; sometimes ext3 via CLI). It can work in a pinch, but it's awkward for large VM exports and not consistently supported -- NFS is usually simpler and faster.
- *OVF/OVA export tools*: exist, but for bulk migration of hundreds of VMDKs they are not a sane automation surface. Stick to vmkfstools over an NFS datastore.
- *vCenter with shared storage*: Use Storage vMotion (requires licenses you probably don't have)
- *VMware Converter*: Requires Windows, vCenter, licenses, and still uses network transfer
- *Starwind V2V*: Works but still requires mounting the VMFS somehow (catch-22\*)

\* *Catch-22: A paradoxical situation where you need X to get Y, but need Y to get X. You need to mount VMFS to use V2V tools, but you need V2V tools to access VMFS without ESXi.*

### Why This Guide Matters

In practice, it's `vmkfstools` over NFS. The question is: **do you want it to take 3 minutes or 3 hours per VM?**

That's what this guide is about.

---

## The Problem That Wasn't

I needed to migrate VMs off an ESXi host. Standard approach: mount an NFS datastore as the target, then use `vmkfstools` to copy VMDKs out of VMFS and onto the NFS share.

**Expected speed**: Maybe 100 MB/s?  
**Actual speed**: 3-5 MB/s  
**My conclusion**: "The HDD is dying from all these random reads"

So I did what any reasonable person would do: ordered a 4TB SSD for $350, cloned the entire VMFS datastore to it, and prepared to write a guide about how SSDs are essential for ESXi migration.

Spoiler: I'm an idiot.

---

## The "Aha" Moment (That Took Too Long)

After getting the SSD and seeing great speeds (98 MB/s), I decided to benchmark the HDD one more time for comparison.

**Result**: 82 MB/s.

Wait, what?

Same HDD. Same VM. Same everything. Except now it was only 20% slower than the SSD, not 95% slower.

**What changed?**

I'd edited `/etc/exports` to use `async` mode for better performance... but I never ran `exportfs -ra` to reload the config. The NFS server was still running with `sync` mode from the previous config.

---

## The Real Culprit: NFS Sync Mode

Here's what `sync` mode does:

- Every write must hit physical disk before acknowledging to the client
- No write caching, no buffering
- Safe for data integrity
- **Absolutely murders performance**

Here's what `async` mode does:

- Writes are buffered in RAM and acknowledged immediately
- NFS can batch writes efficiently
- Less safe if you lose power mid-transfer
- **13-24x faster for bulk data transfer** -- measured, not estimated

The performance numbers, benchmarked across real hardware:

| Configuration | Throughput | Time for 32 GB provisioned (~19 GB allocated, ~41% sparse) | Penalty |
| --- | --- | --- | --- |
| HDD->HDD, 1G, NFS sync | 6.5 MB/s | 2h 49m 58s | baseline doom |
| HDD->HDD, 1G, NFS async | 82.4 MB/s | 4 minutes | **13x faster** |
| HDD->HDD, 25G, NFS sync | 6.5 MB/s | 2h 49m 49s | network irrelevant |
| HDD->HDD, 25G, NFS async | 110.2 MB/s | 3 minutes | **17x faster** |
| SSD->NVMe, 25G, NFS sync | 12.3 MB/s | 26m 21s | the cruelest joke |
| SSD->NVMe, 25G, NFS async | 289.7 MB/s | 67 seconds | **24x faster** |

The cruelest entry: SSD->NVMe over 25G with sync enabled. You bought all the right hardware, built the perfect stack -- and a single config option turns 290 MB/s into 12 MB/s. The sync penalty actually *grows* as your hardware gets faster, because there's more potential being wasted.

**The lesson**: Forgetting to run `exportfs -ra` cost me 13-24x in performance, not the HDD.

---

## What I Actually Learned (With Benchmarks)

### Test Setup

The full matrix covers all combinations of source x destination x NIC speed x MTU -- 16 runs total, same VM each time -- 32 GB provisioned, ~19 GB allocated (~41% sparse). Folder naming tells you everything: `esxi_{src}_nfs_{dst}_{nic}_mtu{mtu}`.

- **Source**: WD Gold WD4003FRYZ (HDD) · Samsung 870 QVO 4TB (SSD)
- **Destination**: WD Gold RAID array (HDD) · KIOXIA Exceria Pro (NVMe)
- **NIC**: onboard 1G · Intel XXV710-DA2 25G, direct DAC cable
- **MTU**: 1500 (standard) · 9000 (jumbo), both tested at both speeds
- **VM**: k8s\_master.vmdk -- 32 GB provisioned, ~19 GB on-disk (`du`), ~41% sparse (≈1.68x thinness)
  - Note: throughput and "MB/s" figures are computed from the **allocated** bytes (~19 GB), because `vmkfstools -d thin` copies allocated blocks. If your disks are thick-provisioned, expect times to scale roughly with the **provisioned** size.
  - How that was measured on Linux:  

    ```bash
    du -h k8s_master-flat.vmdk  # ~19G on disk  

    du -h --apparent-size k8s_master-flat.vmdk  # ~32G provisioned
    ```

### Full Results Matrix

| Source | Dest | NIC | MTU | Time | MB/s | vs base |
| --- | --- | --- | --- | --- | --- | --- |
| HDD | HDD | 1G | 1500 | 3m 56s | 82.4 | baseline |
| HDD | HDD | 1G | 9000 | 3m 47s | 85.7 | 1.0x |
| HDD | NVMe | 1G | 1500 | 3m 54s | 82.9 | 1.0x |
| HDD | NVMe | 1G | 9000 | 3m 46s | 85.8 | 1.0x |
| HDD | HDD | 25G | 1500 | 2m 56s | 110.2 | 1.3x |
| HDD | HDD | 25G | 9000 | 2m 49s | 114.5 | 1.4x |
| HDD | NVMe | 25G | 1500 | 2m 38s | 122.9 | 1.5x |
| HDD | NVMe | 25G | 9000 | 2m 32s | 127.4 | 1.5x |
| SSD | HDD | 1G | 1500 | 3m 18s | 98.1 | 1.2x |
| SSD | HDD | 1G | 9000 | 3m 09s | 102.6 | 1.2x |
| SSD | NVMe | 1G | 1500 | 3m 17s | 98.6 | 1.2x |
| SSD | NVMe | 1G | 9000 | 3m 08s | 103.0 | 1.2x |
| SSD | HDD | 25G | 1500 | 2m 05s | 155.1 | 1.9x |
| SSD | HDD | 25G | 9000 | 2m 03s | 157.4 | 1.9x |
| SSD | NVMe | 25G | 1500 | 1m 08s | 285.6 | 3.5x |
| **SSD** | **NVMe** | **25G** | **9000** | **1m 07s** | **289.7** | **3.5x** |

### What The Data Proves

**MTU 9000 vs 1500 -- the free lunch:**

Consistent +3-4 MB/s absolute gain across every single combination. The percentage looks better on slower setups (~4% on 1G vs ~1.5% on 25G+NVMe) -- not because the lemon got juicier, just because the glass was smaller. Set it. It is free.

**NVMe destination -- context is everything:**

On 1G: NVMe vs HDD = +0.4 MB/s. That is not a rounding error, that is a wall called 1G. The ceiling does not care what is behind it. On 25G + SSD source: +130 MB/s, +84%. Completely different story. Do not upgrade the destination in isolation -- upgrade the full stack.

**25G network -- the bigger the pipe, the more storage matters:**

With HDD source, 25G buys ~30%. With SSD+NVMe, 25G is a 3x multiplier. The network upgrade does not help much with slow storage -- there is nothing to carry. But once the storage can actually feed it, the pipe pays off.

**SSD source -- latency matters more than you think:**

On 1G: SSD is 1.2x faster than HDD. Network is the wall, both drives can feed it easily. On 25G+NVMe: SSD is 2.3x faster. The SSD's 1.5ms read latency vs HDD's 9ms finally has room to matter -- vmkfstools pipelines better, the network stops being the lid, and everything accelerates together.

**Upgrades multiply, not add:**

| Layer | Effect in isolation | Effect when full stack ready |
| --- | --- | --- |
| async (vs sync) | 13-24x -- always | 13-24x -- always |
| MTU 9000 | +4% everywhere | +4% everywhere |
| 1G -> 25G | 1.3x with HDD | 2.9x with SSD+NVMe |
| HDD -> SSD source | 1.2x on 1G | 2.3x on 25G+NVMe |
| HDD -> NVMe dest | ~0 on 1G | +84% with SSD+25G |

Stack all four: 82 -> 290 MB/s. 3.5x. These are 16 data points, not theory.

### Source IOPS (ESXi esxtop SCSI counters)

You'll notice source read MB/s is consistently higher than the wire throughput MB/s in the table above. This is expected: `vmkfstools -d thin` scans the full provisioned range of the VMDK to find allocated blocks, but only transmits the allocated ones over the wire. For a ~41% sparse disk, ESXi reads ~32 GiB worth of blocks from the source device while only ~19 GiB travels over the network. Source IOPS reflect the scan; wire throughput reflects what was actually sent.

| Source | Setup | Read IOPS | Read MB/s |
| --- | --- | --- | --- |
| HDD | 1G MTU1500 | 2,206 | 138 MB/s |
| HDD | 1G MTU9000 | 2,311 | 145 MB/s |
| HDD | 25G (HDD dst) | 2,955-3,044 | 185-190 MB/s |
| HDD | 25G (NVMe dst) | 3,238-3,349 | 202-209 MB/s |
| SSD | 1G MTU1500 | 2,604 | 163 MB/s |
| SSD | 1G MTU9000 | 2,751 | 172 MB/s |
| SSD | 25G (HDD dst) | 4,175-4,184 | 261-262 MB/s |
| **SSD** | **25G (NVMe dst)** | **7,164-7,693** | **448-481 MB/s** |

### Destination IOPS (Linux iostat)

| Dest | Source | Setup | Write IOPS | Write MB/s | %util |
| --- | --- | --- | --- | --- | --- |
| HDD | HDD | 1G | 1,211 | 89 MB/s | 2.0% |
| HDD | SSD | 1G | 1,375 | 101 MB/s | 2.2% |
| HDD | HDD | 25G | 414-443 | 120-128 MB/s | 5-6% |
| HDD | SSD | 25G | 247-249 | 160 MB/s | ~7% |
| NVMe | HDD | 1G | 1,434-1,468 | 90-92 MB/s | 0.0% |
| NVMe | SSD | 1G | 1,617-1,707 | 101-107 MB/s | 0.1% |
| NVMe | HDD | 25G | 2,157-2,237 | 138-143 MB/s | 0.1% |
| **NVMe** | **SSD** | **25G** | **4,712-4,724** | **296 MB/s** | **0.2%** |

NVMe at 0.2% utilisation doing 4,700+ IOPS. It could run this workload 500 times over. It is not the bottleneck. It will never be the bottleneck.

## Why The HDD Didn't Suck

My WD Gold is an enterprise drive designed for this kind of workload:

- 7200 RPM (not slow consumer 5400 RPM)
- Large cache (256 MB)
- Firmware optimized for random I/O
- Rated for 24/7 datacenter use

But more importantly, vmkfstools' read pattern isn't actually tiny random 4K reads:

- Average read size: ~64 KB chunks
- Sequential reads within each VMDK extent
- Read-ahead caching helps
- 2,000+ IOPS is achievable on enterprise HDDs

**Consumer desktop drives (WD Blue, Seagate Barracuda) would perform worse** - probably 40-60 MB/s instead of 82 MB/s. But still infinitely better than 3-5 MB/s with sync mode.

---

## The SSD Staging Approach (Still Valid, But Context Matters)

Even though the HDD performed OK, the SSD staging approach still has merit - but the reason depends on your network:

### Why SSD staging makes sense on 1G:

- 19% faster (98 vs 82 MB/s) - modest but real
- Eliminates variables (consistent performance regardless of HDD quality)
- Consumer HDDs would show a bigger gap

### Why SSD staging REALLY makes sense on 25G:

- **129% faster** when paired with NVMe destination (286 vs 125 MB/s)
- Source latency (1.5ms vs 9ms) becomes the critical factor
- HDD's 9ms latency limits pipelining the 25G link can handle
- The advantage scales dramatically with network speed

### When you DON'T need SSD staging:

- You have 1G network and enterprise HDDs (82 MB/s is fine for most migrations)
- Your migration window isn't tight
- You remembered to enable NFS async (check it twice!)

### The upgrade decision matrix

| Setup | Cost | MB/s | Notes |
| --- | --- | --- | --- |
| HDD + 1G async | $0 | 82 | Good enough for most |
| SSD + 1G async | $350 | 98 | +19%, network still limits |
| HDD + 25G async | ~$200 NICs | 114-125 | Better, HDD latency limits |
| SSD + 25G + NVMe dest | ~$550 | **286** | 3.5x baseline |
| Any + NFS **sync** | $0 | 6-12 | **Don't do this** |

---

## The Step-by-Step Guide (With All The Checks)

### Step 0: Don't Be Like Me

Before you do ANYTHING else, let's make sure NFS is configured correctly.

#### On your Linux NFS server:

```bash
# Edit /etc/exports - USE ASYNC NOT SYNC
cat > /etc/exports  mount denied
# - Firewall blocking port 2049 -> timeout
# - Wrong subnet in /etc/exports -> permission denied
```

Test from another Linux machine first if you're paranoid (you should be):

```bash
showmount -e 192.168.0.105
mount -t nfs 192.168.0.105:/mnt/raid/esxi-import /mnt/test
```

### Step 4: Migrate VMs

**Consistency rule:** don't copy live disks unless you're deliberately taking that risk. Power the VM off, or copy from a snapshot chain (quiesced if you can). `vmkfstools -i` is a copier, not a consistency layer.

```bash
# For each VM disk
time vmkfstools -i /vmfs/volumes/source/vm/disk.vmdk \
  -d thin \
  /vmfs/volumes/migration_target/vm/disk.vmdk
```

**Why not `cp` / `scp` / "download from the datastore browser"?**

Those paths often read the *logical* size of a thin VMDK and write it back out as a fully-allocated file -- i.e., "32 GB provisioned" turns into "32 GB transmitted and stored", even if only ~19 GB was actually allocated. If you care about preserving sparseness (and keeping the migration fast), stick to `vmkfstools -i` for the copy step.

**Expected performance (assuming NFS async is enabled):**

| Source Drive Type | Effective Speed | Time for 32 GB provisioned (~19 GB allocated, ~41% sparse) |
| --- | --- | --- |
| Enterprise HDD | 80-90 MB/s | ~4 minutes |
| SSD | 95-105 MB/s | ~3 minutes |
| Consumer HDD | 40-60 MB/s | ~6-8 minutes |
| Any drive with NFS sync | 6-12 MB/s | up to 2h 50m |

If you're seeing 3-5 MB/s, **STOP**. You forgot to enable async or forgot to reload the config. Go back to Step 0.

### Step 5: Validate Copies

Hash while both datastores are still mounted on ESXi, saving the checksum file to the NFS target alongside the copy. Then you can power down the source and verify cold on the Linux side — no cache ambiguity, no "same mount" shortcut.

```bash
# On ESXi — hash the source flat file, save checksum to NFS target
md5sum /vmfs/volumes/source-datastore/vm/disk-flat.vmdk > /vmfs/volumes/migration_target/vm/disk-flat.vmdk.md5

# Power down / unmount source when done with all VMs

# On Linux — hash the destination flat file and compare against the saved checksum
md5sum /mnt/raid/esxi-import/vm/disk-flat.vmdk
cat /mnt/raid/esxi-import/vm/disk-flat.vmdk.md5
# Both lines should show the same hash
```

The .md5 file lives on the NFS share next to the copy, so it travels with the data and doubles as a permanent audit trail.

For large disks, spot-check before committing to a full hash:

```bash
# First 100MB (on ESXi, against source)
head -c 100M /vmfs/volumes/source-datastore/vm/disk-flat.vmdk | md5sum

# Same on Linux, against destination
head -c 100M /mnt/raid/esxi-import/vm/disk-flat.vmdk | md5sum
```

---

## Troubleshooting: The Checklist

**If migration is slow ( 3 min per VM)
- Actual benefit: Eliminated risk of consumer HDD being slow
- Bonus: I now own a 4TB SSD

**When it's worth it:**

- Large migration (100+ VMs)
- Tight migration window
- Unknown/consumer source drives
- You want guaranteed performance
- You can repurpose the SSD afterward

**When it's not worth it:**

- Small migration ( NVMe dest, 25G MTU9000)
- Full data: [benchmark results](https://cipeople.github.io/esx-migration/benchmark-results.html)

For 1G networks, the SSD vs HDD difference is muted by network limits.

---

## Lessons Learned (The Hard Way)

### 1. **Configuration mistakes cost more than hardware**

- Forgetting `exportfs -ra`: 13-24x penalty in measured runs (and up to ~27x if you're in the 3-5 MB/s pit)
- Buying an SSD to "fix" it: $350
- Feeling stupid when you realize: Priceless

### 2. **Check your assumptions with benchmarks**

- "The HDD is too slow" -> Actually fine with proper config
- "I need an SSD" -> Only 20% faster in practice
- "This will take 2 weeks" -> Takes 4 hours with correct setup

### 3. **Enterprise hardware has a reason to exist**

- WD Gold handled 2,174 IOPS just fine
- Consumer drives would struggle more
- But config matters more than hardware

### 4. **Document your changes**

When you edit a config file at 2 AM:

- ☐ Make the change
- ☐ Reload the service
- ☐ Verify it applied
- ☐ Test it works
- ☐ Write down what you did

Skip any step and you'll be buying unnecessary SSDs.

### 5. **The checklist is your friend**

Before declaring "the hardware is slow":

1. Is the config actually loaded?
2. Is the service actually restarted?
3. Did I typo the path?
4. Is the firewall blocking it?
5. Did I test from another machine?

If answer to any is "uh... maybe?", go check.

---

## The Updated Guide Philosophy

**Original plan**: "Buy SSD, use SSD, go fast"

**Reality**: "Enable NFS async, double-check you enabled it, triple-check it actually loaded, THEN decide if you need an SSD"

The SSD staging approach is still valid - it eliminates variables and guarantees performance. But it's optimization, not a requirement.

The **requirement** is proper NFS configuration.

---

## Conclusion

If you take away one thing from this guide:

**NFS sync mode will ruin your day, your week, and your migration project.**

Enable async. Reload the config. Verify it loaded. Test it works.

*Then* worry about whether you need an SSD.

And if you do buy an SSD for staging, at least you'll actually get the performance you paid for, unlike me who got the same speed from the HDD after fixing the config.

After a week of experiments, the answer that emerges isn't about hardware — it's about pipeline design. The bottleneck hierarchy is proven and strict. But there's a higher-level observation buried in the tmpfs results: **the destination storage is almost never the real constraint**. You're paying for NVMe write performance that sits at 0.2% utilisation while vmkfstools and your source drive do all the work.

The data points toward a better approach. One where the destination disk drops out of the critical path entirely during the copy phase, and where copy and commit become separate, independently-optimised operations. The throughput numbers in this article — the tmpfs ceiling, the parallelism plateau, the source scan rates — are the inputs to that design.

The full pipeline — copy, detect, convert, verify, commit — can be automated. When it is, migrations that currently require babysitting become fire-and-forget. The benchmarks here tell you what the ceiling looks like. The automation is what gets you there consistently.

> If you'd rather not build that yourself — get in touch: vlad [at] cipeople [dot] com — this is a solved problem.

---

## Appendix: The Actual Commands (Copy-Paste Ready)

### NFS Server Setup

```bash
# /etc/exports
/mnt/raid/esxi-import 192.168.0.0/24(rw,async,no_root_squash,no_subtree_check)

# Apply changes (DON'T FORGET THIS)
exportfs -ra

# Verify async is enabled (DO THIS TOO)
exportfs -v | grep async

# Start NFS server
systemctl enable --now nfs-server
```

### ESXi NFS Mount

```bash
# Mount NFS
esxcli storage nfs add -H 192.168.0.105 -s /mnt/raid/esxi-import -v migration_target

# Verify
esxcli storage nfs list
```

### Migrate VM Disk

```bash
# Thin provisioning (copies only used blocks)
vmkfstools -i /vmfs/volumes/source/vm/disk.vmdk \
  -d thin \
  /vmfs/volumes/migration_target/vm/disk.vmdk

# Thick eager zeroed (faster but uses full space)
vmkfstools -i /vmfs/volumes/source/vm/disk.vmdk \
  -d eagerzeroedthick \
  /vmfs/volumes/migration_target/vm/disk.vmdk
```

### Validate

```bash
# Hash the flat files
md5sum /vmfs/volumes/source/vm/disk-flat.vmdk
md5sum /vmfs/volumes/migration_target/vm/disk-flat.vmdk
```

---

**License**: CC0 / Public Domain - do whatever you want with this

**Last updated**: February 2026 (and won't be updated again)

**Author**: A sysadmin who spent $350 on an SSD to fix a configuration problem, so you don't have to

**Disclaimer**: This worked for me. Your mileage may vary. Back up your data. Don't blame me if something breaks. Seriously, back up your data first.

---

## P.S. For the Homelab Folks

Yes, people run desktop HDDs in ESXi servers. I've seen:

- "Temporary" repurposed gaming drives (permanent)
- Random WD Blues in whitebox builds
- Seagate Barracudas "because I had them"
- "It works in my PC" logic

If you're running consumer drives:

- NFS async is **mandatory** (not optional)
- SSD staging will help more (maybe 2-3x faster)
- Consider upgrading to WD Red/Red Pro for ~$50 more
- Or just buy a used enterprise drive on eBay for $30

But most importantly: **Check your NFS config three times**.

You're welcome.