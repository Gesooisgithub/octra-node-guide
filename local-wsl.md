# Running an Octra node at home on Windows (WSL2)

This guide covers running the node on a Windows PC using WSL2. It works, and we ran a
fully synchronised node this way.

**But read section 0 first.** Most home internet connections cannot host a validator, for
a reason no amount of configuration can fix, and section 0 tells you which kind you have
in about two minutes. Everything after it assumes you passed that test.

**Contents**

0. [Read this first: can your connection host a validator?](#0-read-this-first-can-your-connection-host-a-validator)
1. [If the answer was no](#1-if-the-answer-was-no)
2. [Prepare WSL2](#2-prepare-wsl2)
3. [Install the node](#3-install-the-node)
4. [The two WSL traps](#4-the-two-wsl-traps)
5. [The four network layers](#5-the-four-network-layers)
6. [Numbers we measured](#6-numbers-we-measured)
7. [Departures from the official README](#7-departures-from-the-official-readme)

---

## 0. Read this first: can your connection host a validator?

A validator must be **contactable**. Other nodes need to open a connection *to* you,
at any time, without you having contacted them first. An observer only makes outbound
connections and works from anywhere — a validator does not.

Many home connections put you behind **carrier-grade NAT (CGNAT)**: your router does not
hold a public address of its own, and you share one with many other subscribers. Inbound
connections have nowhere to land. Port forwarding in your router does nothing, because the
packets never reach your router in the first place. This is a property of the connection
you are sold, not a setting, and you cannot configure your way out of it.

Three tests, cheapest first. **Do them before installing anything.**

### Test 1 — Compare the two addresses (authoritative)

Log into your router's admin page and find the WAN / Internet address it reports. Then,
from the PC:

```powershell
curl https://api.ipify.org
```

- **The two match** → your router holds a real public address. Good sign, continue to
  test 3.
- **They differ** → **you are behind CGNAT.** Port forwarding cannot work. Stop here and
  read section 1.

This is the test that gives a definitive answer, and it takes a minute.

### Test 2 — Is the WAN address in the CGNAT range?

If the address your router reports falls within **`100.64.0.0/10`** (that is, `100.64.x.x`
through `100.127.x.x`), it is the range reserved for exactly this purpose by RFC 6598 and
you are certainly behind CGNAT.

You can also look at the second hop of a traceroute:

```powershell
tracert -d -h 6 8.8.8.8
```

> **Important caveat, learned the hard way:** the *absence* of a `100.64.x.x` hop does
> **not** clear you. On the connection we tested, the second hop was an ordinary public
> address and CGNAT was in place anyway. Test 2 can prove the problem exists; it cannot
> prove it does not. **Test 1 is the one to trust.**

### Test 3 — The real thing

Once the node is actually running and listening (section 3), have something outside your
network try to reach port 19000. Any "open port check" web service will do, or from a
machine on a different network:

```bash
nc -vz <YOUR_PUBLIC_IP> 19000
```

- **Open** → you can run a validator.
- **Connection timed out** → the packet is not even reaching your router. CGNAT, or a
  firewall in front of you.

---

## 1. If the answer was no

You have three options, in order of how much sense they make:

1. **Rent a small server** and follow [vps.md](vps.md). This is what we ended up doing.
   A machine that meets the requirements is an ordinary monthly commodity cost, and the
   whole class of problem disappears: a rented server holds its own public address.

2. **Ask your provider for a public address.** Many sell one as an add-on, sometimes
   called a static or dedicated IP. If yours does, it is usually a small recurring
   charge and it makes your existing connection usable. Ask specifically whether the
   address is *public and dedicated*, not merely *static* — a static address inside CGNAT
   solves nothing.

3. **Run an observer instead.** An observer needs no inbound port and works perfectly
   behind CGNAT. It follows the chain, verifies everything, and stays useful. It votes on
   nothing and earns nothing — but if you want to learn the software, this is a real and
   valid way to run it, and everything in this guide except section 5 applies.

What does **not** work, no matter how long you spend on it: port forwarding rules, UPnP,
port triggering, firewall exceptions, or a dynamic DNS name. None of them can deliver a
packet that never arrives.

---

## 2. Prepare WSL2

### 2.1 Install a distribution

```powershell
wsl --install -d Ubuntu-24.04 --no-launch
```

### 2.2 Put it on a disk with room

You need roughly **50 GB free**, and the default location is your system drive. Moving the
distribution requires shutting WSL down, **which also stops Docker Desktop** if you run it:

```powershell
wsl --shutdown
wsl --manage Ubuntu-24.04 --move D:\WSL\Ubuntu-24.04
```

### 2.3 Enable systemd

`install.sh` registers pm2 as a systemd service, so systemd must be running. Inside the
distribution, edit `/etc/wsl.conf`:

```ini
[boot]
systemd=true

[interop]
appendWindowsPath=false
```

Then, from Windows:

```powershell
wsl --terminate Ubuntu-24.04
```

Reopen it and confirm:

```bash
systemctl is-system-running     # should print: running
ps -p 1 -o comm=                # should print: systemd
```

If PID 1 is not `systemd`, stop and fix this before continuing — nothing downstream will
work properly.

---

## 3. Install the node

From here the steps are **identical to the server guide**, and rather than duplicate them
(and let the two copies drift apart), follow [vps.md](vps.md) sections 5 through 10:

| Step | Section in [vps.md](vps.md) |
|---|---|
| Clone and verify the package | [5](vps.md#5-get-the-node-software) |
| Install the toolchain | [6](vps.md#6-install-the-toolchain) |
| Verify the package — **and read the warning about test fixtures** | [7](vps.md#7-verify-the-package) |
| Configure, build, sync | [8](vps.md#8-configure-build-and-sync) |
| Back up your identity | [9](vps.md#9-back-up-your-identity) |
| Start the node | [10](vps.md#10-start-the-node) |

Three differences that matter at home:

- **`--advertise`**: use your public address, or better a DNS name that follows it if the
  address changes. It accepts either. It must be the address the *outside world* sees, not
  your `192.168.x.x`.
- **No `ufw`**: on a home machine the relevant firewall is Windows' own — see section 5.
- **`tmux` is still worth it.** The sync is long, and a closed terminal window ends it.

Then come back here for section 4, which is where home installs actually go wrong.

---

## 4. The two WSL traps

These two cost us hours, and both make a perfectly healthy node look like it is crashing
in a loop.

### Trap 1 — WSL shuts the distribution down behind your back

WSL terminates a distribution when no Windows process is holding it open. The virtual
machine's kernel may stay alive (Docker Desktop keeps it up, since it shares the same VM),
but your Ubuntu distribution is torn down and rebuilt on the next command: systemd starts
again, `pm2 resurrect` recreates the node, and it gets a new process ID.

**The symptom that identifies it:** `journalctl` shows repeated
`Startup finished in ...ms` lines, while `/proc/uptime` runs continuously without
resetting. Nothing crashed — the distribution was stopped and started.

**The fix** is to keep one Windows process attached to the distribution at all times:

```powershell
Start-Process -WindowStyle Hidden wsl.exe -ArgumentList '-d','Ubuntu-24.04','-u','root','--','sleep','infinity'
```

That does not survive a reboot. To make it durable, register it as a scheduled task that
runs at logon.

This trap does not exist on a real server, which is one more argument for
[vps.md](vps.md).

### Trap 2 — `sudo -iu octra` kills the node

The **login** form of `sudo` opens a login session, and when that session ends, systemd may
kill every process it owns — including the pm2 daemon supervising your node. Always use
the **non-login** form:

```bash
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/stat.sh'
```

> **Departure from the official README** — the README uses `sudo -iu octra` throughout.
> On WSL that reliably killed our node. On a stock Ubuntu server it turns out to be
> harmless, because `/etc/systemd/logind.conf` ships with `KillUserProcesses=no` — we
> verified this by running an entire upgrade through `sudo -iu` and confirming the node's
> process ID was unchanged. But systemd's *upstream* default is `KillUserProcesses=yes`,
> so the trap is real wherever that default survives. The non-login form is correct
> everywhere; use it and never think about this again.

Also run this once, so the `octra` user's processes are allowed to persist without an
active session:

```bash
sudo loginctl enable-linger octra
```

### The corollary: give it 105 seconds

The node needs roughly **105 uninterrupted seconds** from start to replay its epochs, and
only then does it open port 8080 (RPC) and port 19000 (consensus). Before that,
`rpc = unavailable` is correct and expected.

Both traps above interrupt exactly this window, which is why the node "never comes up":
each restart puts it back at the beginning. Start it, leave it completely alone for two
minutes, and only then ask it how it is doing.

---

## 5. The four network layers

**Only needed for a validator.** An observer makes outbound connections only and needs
none of this.

A packet from the Internet has to cross four boundaries to reach your node. All four must
be right, and a failure at any one looks identical from outside.

| # | Layer | How to satisfy it | How to check |
|---|---|---|---|
| 1 | The node listens inside WSL | automatic once running | `ss -tln \| grep 19000` |
| 2 | Windows can see WSL's port | `netsh interface portproxy`, or `networkingMode=mirrored` in `.wslconfig` | connect to the port from Windows itself |
| 3 | Windows Firewall allows it | inbound TCP rules for 19000 and 9000 | see the profile warning below |
| 4 | Your router forwards it | port forwarding to the PC's local address | test from outside the network |

### Layer 3: the firewall profile trap

Windows classifies each network as *Public*, *Private* or *Domain*, and a firewall rule
only applies to the profiles it was created for. A rule created for *Private* does nothing
while Windows considers the network *Public* — and it will show as enabled the whole time.

Check which profile is active:

```powershell
Get-NetConnectionProfile
```

Create inbound rules for **profile `Any`** to sidestep the problem entirely.

### Layer 4: port forwarding is not port triggering

On consumer routers the feature you want is usually called **port forwarding** or
**virtual servers** (often under a NAT or Advanced menu; the wording differs by brand).

**Do not use "port triggering".** It is a different feature that opens a port temporarily,
and only after *you* make an outbound connection first. A validator must be contactable at
any moment by a peer it has never spoken to. Triggering cannot do that. The two options
often sit next to each other in the same menu, which is how the mistake gets made.

Two more things worth doing at this layer:

- **Reserve the PC's local address in DHCP.** Otherwise the forwarding rule points at an
  address your PC no longer has after a reboot, and everything silently stops.
- **If your public address changes, use dynamic DNS**, and configure it *in the router*
  rather than on the PC, so it keeps updating even when the PC is off. `--advertise`
  accepts a DNS name.

### And once more

**Never forward port 8080.** It is the node's RPC and has no authentication. Only 19000
belongs on the public Internet.

---

## 6. Numbers we measured

Machine: 20 cores, 32 GB RAM, with 15.5 GB assigned to WSL.

| | |
|---|---|
| Build from source | ~10 minutes (20 cores) |
| State-sync snapshot | 37.6 GB |
| Disk after sync | 44 GB |
| Node memory once settled | 3.55 GB resident, stable — **not a leak**, it settles and stays |
| Startup before ports open | 105 seconds |

Disk does **not** double during snapshot installation: the staged snapshot is moved into
place with a rename, not a copy. The peak is during staging.

---

## 7. Departures from the official README

| # | What we do differently | Why |
|---|---|---|
| 1 | Test for CGNAT **before** installing anything | The official README assumes a reachable host. Finding out at the end costs a full day. |
| 2 | Use `sudo -H -u` instead of `sudo -iu` | The login form killed our node on WSL. Harmless on stock Ubuntu, fatal where systemd's upstream default applies. |
| 3 | Document the WSL distribution-shutdown behaviour | It looks exactly like a crash loop and is not one. |
| 4 | Spell out the four network layers, with Windows' firewall profiles | The README covers the node, not the four boundaries around it. |
| 5 | Warn that port triggering is not port forwarding | The two sit side by side in most router menus. |
| 6 | Explicit identity backup before funding | The README does not mention that a key was generated. |
| 7 | Warn that `check.sh` prints test fixtures | They look like real node state and cause real panic. |
| 8 | State plainly that an observer earns nothing | Worth knowing before investing a day, if rewards were the point. |

---

## Where to go next

If you got a validator running at home: excellent, and do read
[vps.md section 14](vps.md#14-make-the-process-manager-survive) anyway — the process
manager issue it describes applies to your machine too, and the fix is the same.

If section 0 ruled you out: [vps.md](vps.md) is the way, and honestly the calmer one.
