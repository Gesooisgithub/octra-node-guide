# Octra node on a Windows PC (WSL2)

**Do Step 0 first.** Most home connections cannot host a validator, and Step 0 tells you in
two minutes. Everything after it assumes you passed.

---

## Step 0 — Can your connection host a validator? (2 minutes)

A validator has to be **reachable from outside**. Many home connections share one public
address between many customers, so nothing from the Internet can reach you — and no router
setting fixes it.

**The test.** Open your router's admin page and find the WAN / Internet address it shows.
Then, in PowerShell:

```powershell
curl https://api.ipify.org
```

**Compare the two numbers:**

| Result | Meaning |
|---|---|
| **They match** | Good. Continue to Step 1. |
| **They differ** | You cannot host a validator on this connection. See below. |
| Router shows `100.64.x.x` – `100.127.x.x` | Same thing. You cannot host a validator. |

### If they differ

Three options:

1. **Rent a small server** → [vps.md](vps.md). This is what we did, and it is much less
   work than fighting your connection.
2. **Ask your provider for a public IP address.** Many sell one as an add-on. Ask
   specifically for a *public, dedicated* address — a "static" one that is still shared
   does not help.
3. **Run without validating.** The node still works and still follows the chain, it just
   never votes and never earns. Everything in this guide works except Step 6.

Do not bother with port forwarding, UPnP, or dynamic DNS. They cannot deliver a packet
that never arrives.

---

## Step 1 — Install WSL2

```powershell
wsl --install -d Ubuntu-24.04 --no-launch
```

---

## Step 2 — Move it to a disk with room (skip if C: has 50+ GB free)

```powershell
wsl --shutdown
wsl --manage Ubuntu-24.04 --move D:\WSL\Ubuntu-24.04
```

**Note:** `wsl --shutdown` also stops Docker Desktop if you use it.

---

## Step 3 — Turn on systemd

Open Ubuntu and edit `/etc/wsl.conf`:

```bash
sudo tee /etc/wsl.conf > /dev/null <<'EOF'
[boot]
systemd=true

[interop]
appendWindowsPath=false
EOF
```

Back in PowerShell:

```powershell
wsl --terminate Ubuntu-24.04
```

Reopen Ubuntu and check:

```bash
ps -p 1 -o comm=
```

**You should see:** `systemd`

If you see anything else, stop — nothing below will work.

---

## Step 4 — Install the node

From here it is **the same as the server guide**. Do
**[vps.md Steps 5 to 11](vps.md#step-5--download-the-node)**, then come back here.

Three differences for a home machine:

- **Skip Step 4 (firewall)** in that guide — on Windows the firewall is handled in Step 6
  below.
- In **Step 9**, use your public address for `--advertise`, or a dynamic DNS name if your
  address changes. Not your `192.168.x.x`.
- Run every command with `sudo` in front, since you are not root.

---

## Step 5 — Keep the distro awake

Windows shuts a WSL distro down when no Windows program is holding it open. Your node then
looks like it is crash-looping when it is not.

```powershell
Start-Process -WindowStyle Hidden wsl.exe -ArgumentList '-d','Ubuntu-24.04','-u','root','--','sleep','infinity'
```

Also run this once inside Ubuntu:

```bash
sudo loginctl enable-linger octra
```

**This does not survive a reboot.** To make it permanent, add the PowerShell line above as
a Task Scheduler task that runs at logon. ([details](reference.md#wsl-keeps-shutting-the-distro-down))

---

## Step 6 — Open the ports (validators only)

Four things must all be right. Skip this entirely if you are not validating.

**1. Windows must see WSL's port.** Add a portproxy, or put `networkingMode=mirrored` in
your `.wslconfig`.

**2. Windows Firewall must allow it.** Create inbound TCP rules for **19000** and **9000**,
and set the profile to **Any**:

```powershell
New-NetFirewallRule -DisplayName "Octra 19000" -Direction Inbound -Protocol TCP -LocalPort 19000 -Action Allow -Profile Any
New-NetFirewallRule -DisplayName "Octra 9000"  -Direction Inbound -Protocol TCP -LocalPort 9000  -Action Allow -Profile Any
```

**3. Your router must forward port 19000** to this PC. Look for **port forwarding** or
**virtual servers**.

> **Do not use "port triggering".** It is a different feature that will not work here, and
> it usually sits right next to the one you want.

**4. Reserve this PC's address in your router's DHCP settings**, or the rule will point at
the wrong machine after a reboot.

**Never forward port 8080.**

---

## Step 7 — Finish

Now do **[vps.md Steps 12 to 16](vps.md#step-12--check-you-are-reachable)**: check you are
reachable, get tokens, enroll, make it survive, keep it updated.

Step 15 (making it survive) applies to your PC too.

---

## Everyday commands

```bash
octra stat.sh            # how is it doing
octra enroll.sh status   # bond and validator state
octra upgrade.sh         # is there an update
```

---

## Something looks wrong?

**[reference.md](reference.md)** has a symptom table, including the two WSL-specific traps
that make a healthy node look broken.
