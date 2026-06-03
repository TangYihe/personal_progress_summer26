# GPU procurement — local prototyping card

> Sub-topic of workstream **C (compute)**. Goal: add a discrete GPU to my workstation for
> local prototyping. PI has approved buying a card. ← back to [overview](../README.md).
>
> Last updated: 2026-06-03

---

## Decision (leaning)

**Buy an RTX 4090 (24 GB).** Rationale:

- **40-series over 50-series:** Ada (40-series) has mature, stable CUDA/driver/framework
  support. Blackwell (50-series) needs newer CUDA (12.8+) and recent framework builds →
  setup friction we want to avoid in the explore/prototyping phase.
- **24 GB VRAM:** clears most single-GPU prototyping/fine-tuning; the prior 16 GB machine hit
  CUDA OOM at times.
- **Reference setup:** a labmate (师姐) working on similar things runs a 4090 — so I can reuse
  her working environment / troubleshooting.

**Plan B:** new 4090s are largely EOL by mid-2026 → if supply/price is bad, a **used RTX 3090
(also 24 GB)** is the cheaper fallback. (50-series only if 40-series is truly unavailable.)

---

## Compatibility checklist

Software-checkable items confirmed below; physical items pending a colleague's hands-on check.

**Physical fit — highest risk (case looks compact):**
- [ ] GPU **length** vs. case clearance (4090 models ≈ **300–360 mm**) — measure PCIe-slot-to-front.
- [ ] **Slot thickness** — most 4090s are **3–3.5 slots**; need that many free/unobstructed below the x16.
- [ ] **Width** clearance from card top edge to side panel.
- [ ] **12VHPWR cable** bend clearance above the card (~35 mm) — tight bends are a fire-risk failure mode.

**Power supply — second risk (likely needs replacing):**
- [ ] **PSU wattage** — want **850 W+** quality unit (4090 TGP 450 W + system + transient spikes).
      *Not readable in software — read the physical label.*
- [ ] **Connectors** — native 12VHPWR / 12V-2x6 (ATX 3.0) **or** 3–4× PCIe 8-pin via adapter.
- [ ] If replacing: new PSU **physically fits** the (compact) case.

**Confirmed via command line (✅ not blockers):**
- [x] **PCIe x16 slot present and empty** — `00:01.0` (CPU PEG) not enumerated → slot free.
- [x] **No existing discrete GPU to remove** — only Intel UHD 770 iGPU.
- [x] **CPU/board support** — i5-14600K + B760M handle a 4090 fine (4090 is PCIe 4.0 x16).

**Minor / later:**
- [ ] Motherboard **Resizable BAR** enabled (BIOS) for best perf.
- [ ] Consider **64 GB RAM** later for large dataloaders (currently 32 GB; not a fit issue).

---

## Workstation specs (probed 2026-06-03, via CLI)

| Component | Value |
|---|---|
| Motherboard | ASUS **B760M-AYW WIFI D4** (micro-ATX, DDR4) |
| CPU | Intel **i5-14600K** (14C / 20T) |
| RAM | **32 GB** DDR4 |
| Discrete GPU | none (Intel UHD 770 iGPU only) |
| PCIe x16 slot | **present, empty/available** |
| Chassis | SMBIOS reports "Desktop" but uses OEM-default strings → **unreliable**, generic/self-built case |
| PSU | **unknown — software can't read it; needs physical label** |

---

## Open items / next actions

- [ ] **Colleague: physical check** — case internal length/width clearance, free slots below x16,
      and **PSU label** (model · wattage · PCIe/12VHPWR connectors). ← long pole.
- [ ] Once clearances known: pick concrete 4090 model (or confirm it won't fit → bigger case / Plan B).
- [ ] Confirm 4090 availability/pricing (mid-2026 EOL risk).
- [ ] Decide on PSU swap (model + wattage) if current one is insufficient.
