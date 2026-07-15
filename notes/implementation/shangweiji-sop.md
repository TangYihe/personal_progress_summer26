# 上位机 (GentoPlatform) SOP

> Standard procedure for using the 上位机 to reset the robot to a safe/normal pose.
> ← back to [overview](../../README.md)

---

## Launch

```bash
cd ~/Tianji/人形控制器版本及SDK_00040403_260604/SDK/00040400/FX_PLATFORM
python3 UI.py
```

Connects to the x86 board at `6.6.7.190`. Workstation must be on the `6.6.7.x` network.

---

## Reset robot to normal pose (main use case)

1. Click **Connect Robot** (top left) — button turns green when connected.

2. For each component you want to move, set speed to **5%** (default is 10–20%) and click **Confirm Speed**. Do this before switching modes.

3. Component mapping:
   - **ARM0** = left arm
   - **ARM1** = right arm
   - **BODY** = torso/lower body
   - **HEAD** = head

4. In the **Position Cmd** field (right side), paste the target joint angles from the reference pose (comma-separated). Click **Add**.

5. Click **Position** (left side, Status switching) to enter position mode. Then click **Run** (right side). The component will move to the target pose.

6. Repeat steps 2–5 for each component.

---

## Reference poses

> Paste into Position Cmd field, comma-separated. Always set speed to 5% first.

### 遥操姿态 — robot standing upright; use this to reset after teleop

| Component | Joint angles |
|-----------|-------------|
| ARM0 (left)  | `90, -90, -90, -90, 0, 0, 0` |
| ARM1 (right) | `-90, -90, 90, -90, 0, 0, 0` |
| BODY         | `0, 0, 0, 0, 0, 0` |
| HEAD         | `0, 0, 0` |

### 打包姿态 — packed/storage pose

| Component | Joint angles |
|-----------|-------------|
| ARM0 (left)  | `88.320, -91.217, -92.725, -61.295, 4.291, -17.317, -0.184` |
| ARM1 (right) | `-89.477, -90.068, 90.970, -62.583, 3.173, -9.719, 3.115` |
| BODY         | `-0.000, 90.003, -130.006, 40.001, 0.001, -0.001` |
| HEAD         | `0, 0, 0` |

### 原点姿态 — all-zero pose

| Component | Joint angles |
|-----------|-------------|
| ARM0 (left)  | `0, 0, 0, 0, 0, 0, 0` |
| ARM1 (right) | `0, 0, 0, 0, 0, 0, 0` |
| BODY         | `0, 0, 0, 0, 0, 0` |
| HEAD         | `0, 0, 0` |

---

## Troubleshooting

- **Error during motion:** click **Reset** (Error Handling section), then retry.
- **Error persists but arm looks physically fine:** restart `python3 UI.py`, or restart the robot. Try again.
- **Can't connect:** check workstation ethernet IP is set to `6.6.7.166/24` and robot is powered on.
- For anything else: contact TIANJI support.
