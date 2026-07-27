# Onsite Demo — Workstation Provisioning Runbook (offline SSD)

> **Milestone (separate track):** provision TIANJI's onsite workstation for **TELEOP-ONLY**
> from a USB SSD, with **no reliance on their internet**. They have 千兆网 but **no VPN** →
> `cache.nixos.org` / GitHub / Hugging Face are unreliable/slow from mainland China regardless
> of local link speed, so we bring everything offline.
>
> Robot + Orin (the driver) travel with us; the **workstation is the only thing we can't bring**.
> Created 2026-07-27. Plan: **build the SSD 07-28 → REHEARSE on the spare workstation 07-28 →
> deploy onsite 07-29** (shoot 07-30).

---

## Scope
- **TELEOP ONLY** — no policy deploy/eval onsite. Only the `operator` profile + its venv.
- Driver env stays on the **Orin** (rides with the robot) — nothing to provision there beyond
  the usual `sync_luna.sh` before travel.
- ⚠️ Teleop still opens the **MuJoCo viewer** → the onsite box needs working GL on a display,
  and `robo.nix` hardcodes `hostGraphics = "nixgl-nvidia"`. **If their box isn't NVIDIA, the
  viewer needs a robo.nix graphics tweak** (same class as the black-window `__NV_PRIME` issue).
  Keep "is it NVIDIA?" on the equipment-questions list.

## Key facts (measured on the source workstation, 2026-07-27)
- Nix: **Determinate Nix 3.21.1, multi-user (daemon)**.
- Flake exposes only a `default` devShell; `robo` selects profiles at runtime. `robo` itself is
  installed via `nix profile` from store path `/nix/store/s3p6dkd684xpq2c3v76whg6l6j6fr6gl-robo-0.1.1`.
- Nix store total: **8.1 GB**. `operator` venv: **1.2 GB** (torch/lerobot live in the *training*
  venvs, which we drop).
- Venvs live **in-repo** at `.robo-nix/venvs/<profile>`. uv-cache is in-repo at `.robo-nix/uv-cache`
  (13 GB — the `uv sync --offline` fallback only).
- **Venv relocation:** `pyvenv.cfg` points at a `/nix/store/…` python (identical on every machine),
  but script shebangs bake the **absolute repo path** (`/home/yihetang/project26/we-teleop/...`).
  → **Put the repo at the IDENTICAL path onsite and the venv works with zero rebuild.**
- **Teleop-only payload ≈ 13 GB** (without the uv-cache) → a 64–128 GB SSD is plenty.

---

## 0. Drive
- [ ] 64–128 GB **fast USB3 SSD**, formatted **ext4** (must be a Linux fs — preserves the venv
      symlinks + exec bits; exFAT/FAT will corrupt the venv). Mount at `/mnt/ssd`.
      *(If stuck with exFAT: `tar` the repo instead of rsync.)*

## 1. Part A — build the SSD on the SOURCE workstation (has internet)
- [ ] Seed the whole nix store to a binary cache on the SSD (~8 GB; includes robo + flake inputs):
  ```bash
  nix copy --all --to file:///mnt/ssd/nix-bincache
  ```
- [ ] Bring the nix eval cache (belt-and-suspenders for offline flake eval; small):
  ```bash
  rsync -aHAX ~/.cache/nix/ /mnt/ssd/nix-eval-cache/
  ```
- [ ] Copy the repo, **teleop-trimmed** (drop training venvs, uv-cache, training-only submodules):
  ```bash
  rsync -aHAX --info=progress2 \
    --exclude 'artifacts/' \
    --exclude '.robo-nix/venvs/training' \
    --exclude '.robo-nix/venvs/train-lerobot' \
    --exclude '.robo-nix/venvs/train-lerobot-runtime' \
    --exclude '.robo-nix/venvs/ci' \
    --exclude '.robo-nix/uv-cache' \
    --exclude 'third_party/GMR' \
    --exclude 'third_party/Isaac-GR00T' \
    /home/yihetang/project26/we-teleop/  /mnt/ssd/we-teleop/
  ```
  *(Keeps the teleop-essential submodules: `wuji_retargeting_v2_pkg`, `XRoboToolkit-PC-Service`
  + `-Pybind`, `tianji`, `manus_sdk`.)*
- [ ] Add the task poses back (they live under the excluded `artifacts/`):
  ```bash
  mkdir -p /mnt/ssd/we-teleop/artifacts/task_poses
  rsync -aHAX artifacts/task_poses/ /mnt/ssd/we-teleop/artifacts/task_poses/
  ```
- [ ] Save the Determinate Nix installer for offline use:
  ```bash
  curl -L -o /mnt/ssd/nix-installer https://install.determinate.systems/nix/nix-installer-x86_64-linux
  chmod +x /mnt/ssd/nix-installer
  ```
- [ ] (Optional fallback) if you want the `uv sync --offline` safety net, also copy the uv-cache
      (13 GB): `rsync -aHAX .robo-nix/uv-cache/ /mnt/ssd/we-teleop/.robo-nix/uv-cache/`.
      **Decide from the rehearsal** — if the offline provision passes without it, leave it off.

## 2. Part B — provision the TARGET (onsite box; no internet, need sudo)
- [ ] Install Determinate Nix — ⚠️ **rehearse the offline path**:
  ```bash
  /mnt/ssd/nix-installer install     # or the online one-liner if install.determinate.systems is reachable
  ```
- [ ] Restore the eval cache: `rsync -aHAX /mnt/ssd/nix-eval-cache/ ~/.cache/nix/`
- [ ] Recreate the **IDENTICAL repo path** (this is what makes the venv work):
  ```bash
  sudo mkdir -p /home/yihetang/project26 && sudo chown $(id -u):$(id -g) /home/yihetang/project26
  rsync -aHAX /mnt/ssd/we-teleop/ /home/yihetang/project26/we-teleop/
  ```
- [ ] Import the nix store from the SSD:
  ```bash
  nix copy --all --no-check-sigs --from file:///mnt/ssd/nix-bincache
  ```
- [ ] Install the robo CLI offline, straight from the store path:
  ```bash
  nix profile install /nix/store/s3p6dkd684xpq2c3v76whg6l6j6fr6gl-robo-0.1.1
  ```
- [ ] Smoke test (operator only):
  ```bash
  cd /home/yihetang/project26/we-teleop
  robo run -p operator -- we teleop --profile keyboard --headless --max-seconds 0.5   # provisioning OK?
  unset __NV_PRIME_RENDER_OFFLOAD        # if NVIDIA-primary display
  robo run -p operator -- we teleop --profile keyboard                                # viewer renders?
  ```
  - Fallback if venv/shebang errors: `robo run -p operator -- uv sync --offline` (needs the
    uv-cache from the optional step above), then re-smoke.

## 3. The 3 spots that MUST be rehearsed (unverified from the source machine)
- [ ] **Offline Determinate Nix install** — installer normally pulls a tarball; confirm offline
      behavior (or that their line can reach `install.determinate.systems`).
- [ ] **`robo` / flake eval offline** — `robo shell` may resolve the `robo-nix` flake input;
      the store copy + eval-cache *should* cover it. Most robo-nix-specific unknown.
- [ ] **Venv relocation** — identical-path should make this a non-issue; `uv sync --offline` is
      the safety net.

## 4. Rehearsal — DO THIS 07-28 on the spare workstation (network UNPLUGGED)
- [ ] Build the SSD (Part A).
- [ ] On the spare box, **pull the network cable**, provision entirely from the SSD (Part B).
- [ ] Confirm the operator smoke test + viewer render pass with **no internet**.
- [ ] Note which of the 3 unknowns bit, fix the runbook, and decide the uv-cache question.
- [ ] Goal: onsite 07-29 becomes "plug in and verify", not "install and debug".

## Payload summary (teleop-only)
| Item | Size | Notes |
|---|---|---|
| nix store (`--all`) | ~8 GB | includes robo + flake inputs |
| repo source + `.git` | ~3 GB | |
| `operator` venv | 1.2 GB | in `.robo-nix/venvs/operator` |
| teleop submodules | ~1 GB | wuji + XRoboToolkit PC-Service + tianji |
| task_poses | tiny | |
| eval cache | small | |
| **total** | **~13 GB** | +13 GB if uv-cache fallback included |
