# TIANJI Luna — Troubleshooting

> Known issues and fixes encountered during bringup and teleop.
> ← back to [overview](../../README.md)

---

## NoMachine to Orin stopped responding

**Symptom:** NoMachine shows "remote host '6.6.7.100' refused connection on port 4000"

**Cause:** Usually happens after robot reboot — the NoMachine service on Orin doesn't come back up automatically.

**Fix:**
```bash
ping 6.6.7.100           # confirm Orin is reachable first
ssh nvidia@6.6.7.100     # password: nvidia
sudo systemctl restart nxserver
```
Then retry NoMachine normally.
