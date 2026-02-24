# CLAUDE.md — ESXi Migration Repo

This file helps AI assistants understand the structure, intent, and conventions of this repository.

## What This Is

A technical article and benchmark dataset about ESXi VM migration using vmkfstools over NFS. The central finding: NFS sync mode causes a 13–24× throughput penalty that dwarfs any hardware upgrade. The article documents a 16-run benchmark matrix (source × destination × NIC × MTU) and the strict bottleneck hierarchy that emerges from it.

The repo is the **source of truth** for the free/public content. A paid advanced article exists separately (see below).

## Files

| File | Purpose |
|------|---------|
| `README.md` | Full article in markdown — renders on GitHub, links to Pages |
| `index.html` | Landing page for GitHub Pages (`cipeople.github.io/esx-migration/`) |
| `esxi-migration-guide.html` | Full styled article with syntax highlighting |
| `benchmark-results.html` | Interactive Chart.js benchmark charts (matrix + sync penalty) |
| `esxi-migration.css` | Shared stylesheet for all HTML files |
| `chart-4.4.1.umd.min.js` | Chart.js bundled locally (no CDN dependency) |
| `LICENCE` | CC BY 4.0 |

**Planned but not yet added:** raw benchmark data (CSV or similar).

## URLs

- GitHub repo: `https://github.com/cipeople/esx-migration`
- GitHub Pages: `https://cipeople.github.io/esx-migration/`
- Article: `https://cipeople.github.io/esx-migration/esxi-migration-guide.html`
- Charts: `https://cipeople.github.io/esx-migration/benchmark-results.html`

## Published Articles

| Platform | URL | Type |
|----------|-----|------|
| Dev.to | `https://dev.to/vlad73/i-spent-350-on-an-ssd-to-fix-a-configuration-problem-esxi-vm-migration-deep-dive-40nc` | Full free article |
| Hashnode | `https://hashnode.com/edit/cmm0ljr3n00cf26iu2c5pa38h` | Teaser, canonical → Dev.to |
| Gumroad | `https://vladster601.gumroad.com/l/selpp` | Paid advanced article ($9) |

## Paid Advanced Article

Exists as a separate Gumroad product — **not in this repo**. Contains:
- Full 16-run matrix + extended hardware (dual 25G, iSCSI over NVMe RAID0)
- tmpfs as migration target — destination storage out of the critical path
- Fan-out parallelism (8 simultaneous copies, plateau analysis)
- vmkfstools pipeline ceiling experiments
- Raw NVMe baseline (concurrent dd readers, queue depth)
- Interactive charts for all of the above
- A service CTA — fully automated migration pipeline is available as a service

**Do not add paid content to this repo.**

## Hardware (Test Bench)

- **Source:** spinning HDD inside ESXi (VMFS datastore)
- **Destination:** NVMe on Linux NFS server
- **Network:** 1G and 25G tested; jumbo frames (MTU 9000) tested
- **Test VM:** 32 GB provisioned, ~19 GB allocated (thin/sparse)
- All benchmark runs: same VM, NFS async unless explicitly noted sync

## Tone and Voice

- Blunt, first-person, no fluff
- Self-deprecating where appropriate ("I'm an idiot", "so you don't have to")
- Data-forward — claims backed by numbers, not assertions
- No marketing language
- Units: binary (1 MB = 2^20 bytes) — stated explicitly in the article

## Conventions

- No version suffixes in filenames — always latest, plain names
- CSS is shared across all HTML files via `esxi-migration.css`
- Chart.js is bundled locally (`chart-4.4.1.umd.min.js`) — no CDN dependency
- GitHub Pages is the canonical home for rendered HTML
- README.md is a full markdown conversion of the article — keep in sync if article is updated

## Contact

vlad [at] cipeople [dot] com
