# Onsite Demo — Workstation Provisioning Runbook (offline SSD)

> **Milestone (separate track):** provision the onsite workstation for **TELEOP-ONLY** from a USB
> drive, with **no reliance on their internet**. They have 千兆网 but **no VPN** →
> `cache.nixos.org` / GitHub / Hugging Face are unreliable/slow from mainland China regardless of
> local link speed, so we bring everything offline.
>
> Robot + Orin (the driver) travel with us; the **workstation is the only thing we can't bring**.
>
> ⚠️ The `2026-07-29` in this filename is the ORIGINAL planned deploy date (kept so existing links
> don't break). **The shoot moved to 08/11–08/13** (postponed 07-28, partner-side delay), so there
> is real runway now — this is no longer deadline-critical.
>
> **Status: Part A DONE + verified (07-31). Part C (the rehearsal) NOT YET RUN.**

---

## Status

| Part | What | State |
|---|---|---|
| **A** | Build the seed on the source workstation (needs internet) | ✅ **DONE 07-31** — `/home/yihetang/project26/onsite-seed`, 5.8 GB, also copied + verified on the USB stick |
| **B** | Provision the target offline | 📜 **scripted** — `provision.sh` ships on the seed, self-checking |
| **C** | Rehearse on the spare workstation, network unplugged | ⬜ **NOT RUN** — deferred, do this before travel |

## Scope
- **TELEOP ONLY** — no policy deploy/eval onsite. Only the `operator` profile + its venv.
- Driver env stays on the **Orin** (rides with the robot) — nothing to provision there beyond the
  usual `sync_luna.sh` before travel.
- ⚠️ Teleop opens the **MuJoCo viewer** → the onsite box needs working GL on a display, and
  `robo.nix` still hardcodes `hostGraphics = "nixgl-nvidia"` for `operator` (verified 07-31).
  **If their box isn't NVIDIA the viewer needs a robo.nix graphics tweak.** Keep "is it NVIDIA?"
  on the equipment-questions list.

---

## Key facts (re-measured on the source workstation 2026-07-31, post-refactor-0722)

- Nix: **Determinate Nix 3.21.1 (nix 2.34.7)**, multi-user (daemon). Unchanged.
- `robo` installed via `nix profile` from **`/nix/store/s3p6dkd684xpq2c3v76whg6l6j6fr6gl-robo-0.1.1`**
  — unchanged by the merge, so the offline install step still works verbatim.
- ⚠️ **TWO robo-nix revs are in play and BOTH must resolve offline:**
  | | rev |
  |---|---|
  | the installed `robo` CLI was built from | `3c55fa1065108399303a675fbb4eef3967a4ee87` |
  | the repo's `flake.lock` pins (refactor-0722 bumped it) | `fb50985641e8e346aa345bdc2c4834fcc034212c` |
  This bump is what broke the Orin on 07-29 (`Could not resolve host` on `robo shell`). Both store
  paths are captured in the seed's bincache — `fb50985`'s source is
  `/nix/store/24r8d1p0rc4w5qnrldprdm0pws0plkn6-source`.
- Nix store: **8.1 GB / 8368 paths**. The `operator` devShell closure is only **4.0 GB** of that —
  `nix copy --all` carries ~4 GB of training/CUDA we don't need. Kept `--all` anyway (headroom is
  free; scoping narrow is how you find out onsite that you scoped wrong).
- `operator` venv: **439 MB** (the runbook's old "1.2 GB" was actually the `ci` venv). Confirmed
  current for refactor-0722 — has `mujoco`, `cv2`, `msgpack` (the extras the merge moved into
  `operator` + the new `render` extra).
- Venvs live **in-repo** at `.robo-nix/venvs/<profile>`; uv-cache in-repo at `.robo-nix/uv-cache`
  (13 GB, the `uv sync --offline` fallback only — **NOT shipped**, see below).
- **Venv relocation:** `pyvenv.cfg` points at a `/nix/store/…` python (identical on every machine),
  but script shebangs bake the **absolute repo path**
  (`#!/home/yihetang/project26/we-teleop/.robo-nix/venvs/operator/bin/python3`).
  → **Put the repo at the IDENTICAL path on the target and the venv works with zero rebuild.**
  The target's *username* does not need to be `yihetang` — only the path must match.
- **Actual teleop-only payload = 5.8 GB** (not the estimated ~13 GB). A 30 GB stick is ample.

## ⚠️ Three corrections the 07-27 draft got wrong

1. **The nix tarball was never captured.** `nix-installer` downloads a nix package by default, so
   an offline `nix-installer install` would have **failed at step one**. Fix = ship the tarball and
   pass `--nix-package-url file://…`. **This can only be fetched while online — do it in Part A.**
   → Ship **nix 2.34.7** (`https://releases.nixos.org/nix/nix-2.34.7/nix-2.34.7-x86_64-linux.tar.xz`),
   deliberately **version-matched** to our Determinate Nix 3.21.1. A different nix version may use
   different cache schema names (`eval-cache-vN` / `fetcher-cache-vN`) and then **silently ignore**
   the copied cache — which puts the network dependency straight back. 2.35.1 (the installer's
   built-in default) shipped as a fallback only.
   → Do **not** pass `--determinate`; that fetches `determinate-nixd` separately. Plain Nix is
   enough for `nix copy` / `nix profile` / `nix develop`. (`determinate-nixd` shipped anyway, in
   case Determinate parity is ever wanted.)
2. **The eval/fetcher cache is LOAD-BEARING, not belt-and-suspenders.** The bincache holds the
   robo-nix store paths, but nix still needs `~/.cache/nix` to map
   `github:ausbxuse/robo-nix/<rev>` → store path without network. 104 MB, non-negotiable.
3. **Two exclusions were missing** from the rsync, worth 2.2 GB:
   - `.cache/` — in-repo, gitignored, `.cache/we/training` alone is **2.2 GB** and regenerable.
   - `third_party/policy` — untracked leftover, unreferenced by refactor-0722 (HANDOFF flagged it
     as removable).

## Other things learned 07-31
- **Use `?compression=zstd`** on the bincache URL, not the default xz: 8.1 GB store → **2.7 GB** in
  a couple of minutes instead of xz's much slower pass.
- The seed **deliberately carries the dirty working tree** — `wuji_finetune.json` (hand recal +
  index/middle pinch off), `pico_world_wuji_hands.yaml` (head/torso `hold`), `driver_supervisor.py`
  (hands-open-on-shutdown). Those are LOCAL and on no branch; onsite teleop needs them.
- **Offline `nix --offline flake metadata` and `--offline` devShell dry-run both PASS on the source
  box.** Weak evidence (the source's fetcher cache is already warm) but it means unknown #2 is
  structurally sound, not just hoped for.
- **Smoke command corrected.** `CLAUDE.md`'s canonical verification line on this tree is
  `we teleop --profile keyboard --driver-address local --headless --max-seconds 0.5` — the 07-27
  draft's version predates the merge. ⚠️ Note the standing gotcha that `--driver-address local` as a
  **flag** doesn't normalize to sim (fails ~3 s in with `paced command publisher failed`); the
  0.5 s bound exits before that, so the smoke test passes regardless. Don't read it as proof that
  sim teleop works.
- **Operator `shellHook` still exports `__NV_PRIME_RENDER_OFFLOAD=1` on every shell entry** →
  `unset` it in the SAME interactive `robo shell`; `robo run` won't do.

## 🔌 USB gotcha (cost ~25 min on 07-31)
The 双接口 (A + Type-C) stick enumerated at **480 Mb/s, `version 2.10`** — USB 2.0 — while plugged
in by its **USB-A** end, even in a blue USB3 port. On A plugs the SuperSpeed pins sit recessed
further back, so a slightly-unseated plug works fine but at USB2. **Plugging the Type-C end in gave
5000 Mb/s, `version 3.20`.** Check `cat /sys/bus/usb/devices/<n>/speed` before trusting a transfer.
⚠️ Not through the marked A→C converter on the hub (SuperSpeed in one orientation only, marked for
the D455).

**Measured stick throughput — wildly asymmetric, and it matters:**
| direction | rate | 5.8 GB takes |
|---|---|---|
| sustained write | **~2–6 MB/s** | ~30 min |
| sequential read | **~150 MB/s** | ~40 s |

`--info=progress2`'s `11.30MB/s` was fiction — the kernel was accepting writes into page cache and
the real flash write happened during the trailing `sync` (a `sync` in `D` state at
`wb_wait_for_completion`, ~1 GB of dirty pages draining). **Do not unplug until `sync` returns.**
Because the *read* side is 150 MB/s, the drive is fine for provisioning and **no purchase is
needed** for this purpose. (A larger SSD is still worth considering purely to haul recorded
episodes home — separate decision.)

---

## 0. Drive
- [x] Any ≥16 GB USB3 drive, formatted **ext4** (must be a Linux fs — preserves the venv symlinks +
      exec bits; exFAT/FAT32 corrupts the venv and caps files at 4 GB). Used the 30 GB aigo U353.
  ```bash
  udisksctl unmount -b /dev/sda1
  sudo mkfs.ext4 -L TELEOP_SEED /dev/sda1
  sudo mkdir -p /mnt/seed && sudo mount /dev/sda1 /mnt/seed
  sudo chown $(id -u):$(id -g) /mnt/seed
  ```

## 1. Part A — build the seed on the SOURCE workstation (has internet) ✅ DONE 07-31
Scripted; the seed lives at `/home/yihetang/project26/onsite-seed` and is reproducible.
```bash
SEED=/home/yihetang/project26/onsite-seed

# nix store -> zstd binary cache (8.1 GB store -> 2.7 GB)
nix copy --all --to "file://$SEED/nix-bincache?compression=zstd"

# eval + fetcher cache — LOAD-BEARING (104 MB)
rsync -aHAX ~/.cache/nix/ "$SEED/nix-eval-cache/"

# repo, teleop-trimmed (note .cache/ and third_party/policy — new)
rsync -aHAX --info=progress2 \
  --exclude 'artifacts/' --exclude '.cache/' \
  --exclude '.robo-nix/uv-cache' \
  --exclude '.robo-nix/venvs/training' --exclude '.robo-nix/venvs/train-lerobot' \
  --exclude '.robo-nix/venvs/train-lerobot-runtime' --exclude '.robo-nix/venvs/ci' \
  --exclude 'third_party/GMR' --exclude 'third_party/Isaac-GR00T' \
  --exclude 'third_party/policy' \
  /home/yihetang/project26/we-teleop/ "$SEED/we-teleop/"

# task poses live under the excluded artifacts/
mkdir -p "$SEED/we-teleop/artifacts/task_poses"
rsync -aHAX /home/yihetang/project26/we-teleop/artifacts/task_poses/ \
            "$SEED/we-teleop/artifacts/task_poses/"

# ONLINE-ONLY artifacts — cannot be obtained later
curl -L -o "$SEED/nix-installer" https://install.determinate.systems/nix/nix-installer-x86_64-linux
chmod +x "$SEED/nix-installer"
mkdir -p "$SEED/nix-tarballs"
curl -L -o "$SEED/nix-tarballs/nix-2.34.7-x86_64-linux.tar.xz" \
  https://releases.nixos.org/nix/nix-2.34.7/nix-2.34.7-x86_64-linux.tar.xz
cp /usr/local/bin/determinate-nixd "$SEED/nix-tarballs/"
```
Then copy to the drive and **wait for the flush**:
```bash
rsync -aHAX --info=progress2 "$SEED/" /mnt/seed/ && sync
```

**Seed contents / sizes (verified on the stick 07-31):**
| Item | Size |
|---|---|
| `nix-bincache/` (all 8368 paths, zstd) | 2.7 GB |
| `we-teleop/` (teleop-trimmed, operator venv incl.) | 2.9 GB |
| `nix-eval-cache/` | 104 MB |
| `nix-installer` | 70 MB |
| `nix-tarballs/` (2.34.7 + 2.35.1 + determinate-nixd) | 80 MB |
| `provision.sh`, `README.txt`, `MANIFEST.txt`, `build.log` | small |
| **total** | **5.8 GB** |

Verified on the stick: dry-run diff empty, 8368/8368 `narinfo`, md5 spot-checks match, venv
`lib64 -> lib` symlink + exec bits intact.

## 2. Part B — provision the TARGET (offline, needs sudo) — SCRIPTED
`provision.sh` on the seed does all of it and **self-checks**. On the target, network **unplugged**:
```bash
sudo -v
/mnt/seed/provision.sh          # logs to the DRIVE so the result comes home
```
It: proves the box is offline (and **fails the run if `cache.nixos.org`/`github.com` answer**, so
the target's own internet can't fake a pass) → records OS/GPU/systemd → installs nix offline via
`--nix-package-url file://` → writes an offline-safe `nix.conf` (`substituters =`,
`accept-flake-config = false`, short timeouts, so stray network use fails fast instead of hanging)
→ restores `~/.cache/nix` → recreates the identical repo path → `nix copy --all --no-check-sigs
--from file://…` → `nix profile install --offline <robo store path>` → runs the three smoke checks.

**Expect ~10–15 min**, dominated by importing 8.1 GB / 8368 paths into the target's store (an
fsync per path) — *not* by the drive.

The one thing it can't script (needs a physical display):
```bash
cd /home/yihetang/project26/we-teleop
robo shell -p operator
unset __NV_PRIME_RENDER_OFFLOAD
we teleop --profile keyboard --driver-address local
# a MuJoCo window must actually RENDER, not be black
```

## 3. The 3 spots that MUST be rehearsed
- [ ] **Offline Determinate Nix install** — plan now exists (`--nix-package-url file://` + shipped
      tarball, no `--determinate`) but **never executed**. Highest remaining unknown.
- [ ] **`robo` / flake eval offline** — structurally covered: both robo-nix revs' store paths are in
      the bincache and the fetcher cache ships. Passes on the *source* box; unproven on a cold one.
      ⚠️ Watch for cache-schema mismatch if the installed nix isn't 2.34.7.
- [ ] **Venv relocation** — identical-path should make it a non-issue. ⚠️ **The `uv sync --offline`
      safety net is NOT available** — we did not ship the 13 GB uv-cache. There's room on the drive
      if the rehearsal shows we need it.

## 4. Rehearsal — NOT YET DONE, do before travel
- [x] Build the seed (Part A).
- [ ] On the spare workstation, **pull the network cable**, run `provision.sh`.
- [ ] Confirm the operator smoke test + viewer render with **no internet**.
- [ ] Bring the drive back; read `provision-<host>-<date>.log` off it.
- [ ] Note which of the 3 unknowns bit, fix this runbook, decide the uv-cache question.
- [ ] ⚠️ Two things about the spare box change how to read the result:
      **does it already have nix?** (then unknown #1 goes untested) and
      **is it NVIDIA?** (if not, a viewer failure is a REAL preview of the TIANJI risk, not an
      artifact of the rehearsal).
- [ ] Goal: onsite becomes "plug in and verify", not "install and debug".
