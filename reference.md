# Reference

Everything that would have cluttered the step-by-step guides: what to do when something
looks wrong, why each warning exists, and where this guide deliberately differs from the
official README.

You do not need to read this. Come here when something surprises you.

---

## Symptom table

| What you see | What it means |
|---|---|
| `rpc = unavailable` just after starting | Normal for 2–3 minutes. It is replaying history. Leave it alone — restarting in a panic means it never finishes. |
| `run.sh` says `candidate paths are stale` | Normal on first run. Use `octra run.sh --rebind-runtime`. |
| Alarming events during `check.sh` or an upgrade | Test fixtures, not your node. [Details below](#checksh-prints-alarming-nonsense). |
| `p2p_connected = 0`, `round_peers = 0` | Usually fine. [Details below](#peer-counters-read-zero). |
| `source_match = False`, `runtime_match = False` | Either a restart that has not settled, or an update is available. Run `octra upgrade.sh`. |
| Port 19000 times out from outside | [Details below](#port-19000-is-not-reachable). |
| Bonded and `ready` but never `active` | [Details below](#bonded-and-ready-but-never-active). |
| "File not found" for a file that exists (Windows) | [PowerShell quoting](#windows-powershell-quoting). |
| Node vanishes and nothing restarts it | The process manager was never handed to systemd. Do [vps.md Step 15](vps.md#step-15--make-it-survive). |
| SSH `Connection refused` right after a reboot | The machine is still booting. Wait 20 seconds. |
| Log looks frozen during an upgrade | The compiler prints nothing for long stretches. Check it is alive with `ps -eo pid,pcpu,comm --sort=-pcpu \| head`. |
| `upgrade.sh` says `status = refused reason = release marker is expired` | The release notice ran out and the team has not re-issued it yet. Your node is fine. [Details below](#updates-and-why-they-are-not-optional). |
| New `sequence`, `action = required`, but `upgrade_available = False` | Nothing to apply — same code, re-issued notice. [Details below](#updates-and-why-they-are-not-optional). |
| `validator_admission_pending` / `leave_running` after an update | The update worked; the node rejoins the active set by itself. Leave it alone. |
| `enroll.sh status` shows `bond status = missing` | Harmless if it also shows `state = ready` and your bond. Your node only keeps recent history, and the bond transaction is older than that. |
| Memory use roughly doubles in the first days | Normal. It settles around 6.5 GB. [Numbers](#numbers-i-measured). |
| Disk filling up | Run `octra storage.sh`. [Details below](#disk-keeps-growing). |

---

## `check.sh` prints alarming nonsense

`check.sh` runs the project's test suite, and the tests print their **fake data** to the
screen. On a first run this is genuinely frightening. You will see things like:

```
event = recovery status = rolled_back ...
event = restore_restart status = held reason = identity_changed action = remain_stopped
status = validator_active ... head_epoch = 41 voting = True
event = state_sync_source status = ready source = https://seed-a.example
event = bond status = resumed tx = aaaaaaaaaaaaaaaa... epoch = 101
```

**None of that is your node.** How to tell fake from real:

- **Epoch numbers in the tens or hundreds.** The real chain is in the millions.
- **Transaction IDs made of one repeated character** (`aaaa...`).
- **Hostnames ending in `.example`.**
- **Resource lines describing a machine that is not yours.**

Only the lines *after* `Ran NNN tests ... OK` describe your node. The line that matters is
the last one: `status = pass gate = validator_tools`.

This trips up automation too: if you write a script that waits for the upgrade to finish by
watching for `status = validator_active`, it will match the fake line and finish early.
Match on `^status = observer_synced` instead.

---

## Peer counters read zero

`p2p_connected`, `consensus_peers` and `round_peers` are **instant snapshots**, and for a
node that is not validating they often read `0`.

This is not a broken network. Other nodes drop non-validators shortly after connecting, so
your node reconnects continuously. In one 300-line slice of my log I counted **78**
`event = connected` lines while `stat.sh` was reporting `p2p_connected = 0` in the same
minute — and the chain head kept advancing perfectly.

**Judge connectivity by whether `head_epoch` is going up and `epoch` is keeping pace.** Not
by those counters.

To see the truth:

```bash
sudo -iu octra tail -n 300 /opt/octra/libv_litecore/data/operator_logs/node.log \
  | grep -c 'event = connected'
```

**One exception:** once you are actually validating, `round_peers = 0` is no longer
harmless — a validator needs peers to take part in a voting round. Watch it at that
moment specifically.

---

## Bonded and ready, but never active

Being bonded is not the same as validating. Your node moves through stages:

| `stat.sh` field | Meaning |
|---|---|
| `validator_bond` | your stake is locked |
| `validator_enrollment = ready` | you have declared yourself available |
| `validator_scheduled` | the network has slotted you into an upcoming set |
| `validator_active` | you are in and voting |

**You can sit at `ready` for a long time.** The number of validator seats is a network
parameter — not "everyone who bonded". When I enrolled, every seat was taken and existing
members had priority, so a perfectly healthy node simply waited.

**Do not re-run `enroll.sh join`** — it will not help. `enroll.sh activate` will refuse
until the network schedules you.

The seat limit is itself changed by protocol updates on a schedule. If you are stuck at
`ready`, read the project's release notes before changing anything. Meanwhile the node
keeps doing useful work — leave it running.

---

## Port 19000 is not reachable

Check in this order:

1. **The provider's own cloud firewall.** If you left it enabled, it blocks the port
   regardless of `ufw`. This is the most common cause.
2. **Your advertised address.** Run
   `grep advertise /opt/octra/libv_litecore/.keys/validator/node.env`. If it holds a
   private address (`10.x`, `192.168.x`, `172.16–31.x`), other nodes are being told to
   reach you at an address that means nothing to them.
3. **`ufw status`** — is 19000 allowed?
4. **Is the node actually listening?** `ss -ltn | grep 19000`

At home, add: is this CGNAT? See [local-wsl.md Step 0](local-wsl.md#step-0--can-your-connection-host-a-validator-2-minutes).

---

## SSH hardening does nothing

In SSH configuration, **the first setting wins**, not the last. Files in
`/etc/ssh/sshd_config.d/` are read in alphabetical order, so `50-something.conf` beats
`60-something.conf`.

Stock Ubuntu cloud images usually ship a `50-cloud-init.conf` containing
`PasswordAuthentication yes`. Put your hardening in a file numbered *above* 50 and it is
silently ignored — the config looks right and does nothing.

That is why [vps.md Step 3](vps.md#step-3--lock-down-ssh) writes `10-hardening.conf`, and
why it checks with `sshd -T` (which shows the **effective** result) instead of trusting the
file.

Still saying `yes`? Find what is winning:

```bash
grep -rn PasswordAuthentication /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

---

## Windows PowerShell quoting

PowerShell strips double quotes when passing arguments to a program like `ssh`. So this
arrives on the server mangled:

```powershell
ssh root@SERVER "sudo -iu octra sh -c "cd /opt/octra/libv_litecore && sh controls/stat.sh""
```

The server ends up running `cd` on its own and then looking for the script in the wrong
directory, giving you a "file not found" error for a file that plainly exists.

**Always: double quotes outside, single quotes inside.**

```powershell
ssh root@SERVER "sudo -iu octra sh -c 'cd /opt/octra/libv_litecore && sh controls/stat.sh'"
```

The `octra` shortcut from [vps.md Step 7](vps.md#step-7--add-the-octra-shortcut) avoids the
problem entirely:

```powershell
ssh root@SERVER "octra stat.sh"
```

---

## WSL keeps shutting the distro down

Windows terminates a WSL distribution when no Windows process is holding it open. The
virtual machine may stay alive, but your Ubuntu distro is torn down and rebuilt on the next
command: systemd restarts, the process manager recreates the node, and it gets a new
process ID.

**It looks exactly like a crash loop and is not one.** The tell: `journalctl` shows
repeated `Startup finished in ...ms` while `/proc/uptime` runs continuously without
resetting.

The fix is in [local-wsl.md Step 5](local-wsl.md#step-5--keep-the-distro-awake).

There is a second WSL trap: **`sudo -iu octra` can kill the node — but only on WSL.**
The login form of `sudo` opens a session, and when that session ends systemd may kill
everything it owned, including the process manager.

On a normal Ubuntu server it is harmless, and it is the form the Octra team recommends, so
it is the one this guide uses. Two things protect you there: stock Ubuntu ships
`KillUserProcesses=no`, and once `install.sh` has registered pm2 as a systemd service the
daemon lives in `system.slice/pm2-octra.service`, outside any login session's scope. I
verified this on my VPS on 2026-08-30 — after two `-iu` sessions and 75 seconds, the pm2
God daemon and the node still had the same PIDs and uptimes.

On WSL it killed my node reliably, so the local guide keeps `sudo -H -u octra`. The same
caution applies anywhere pm2 was started by hand outside systemd: check with
`systemctl show -p MainPID --value pm2-octra` and `cat /home/octra/.pm2/pm2.pid` — if the
two numbers differ, or the unit is inactive, your daemon is not under systemd.

---

## Disk keeps growing

It should not, any more. The node now prunes its own history roughly every two days, so the
live directory stays flat at around 44 GB. Check what is using the space:

```bash
octra storage.sh
```

**You should see:** `pack_gc enabled = true`, and `prior_count` / `prior_bytes`.

Those "prior" states are snapshots kept as rollback points after recoveries and major
updates, tens of gigabytes each. If `prior_bytes` is large, remove them with the official
command:

```bash
octra storage.sh --prune-prior --yes
```

**Never delete anything under `/var/lib/octra` by hand.** Older versions of this guide did,
with `rm -rf`, before the official tool existed. Use `storage.sh`.

The one thing nothing cleans is the log file,
`/opt/octra/libv_litecore/data/operator_logs/node.log`. It grows about 90 MB a day — not a
problem on a 120 GB disk, but it only goes up.

---

## Updates and why they are not optional

`octra upgrade.sh` prints a line like:

```
event = release_marker sequence = 8 action = required \
  public_commit = ... expires_at = 2026-08-29T20:49:00Z
```

A release marked **`action = required` changes the rules of consensus at a fixed point in
time**. A node still running the old version when that point arrives computes different
results from everyone else and falls off the network. I watched this happen: a node stuck
on an old release died repeatedly at the same point until the fix shipped.

**Treat `expires_at` as a hard deadline** and apply well before it.

**A new `sequence` is not always new code.** Each release notice is valid for a few days,
and when it runs out the team re-issues it with a new number — sometimes pointing at
exactly the same code you already run. Every notice says `action = required`, so that word
alone tells you nothing. The line to read is further down: `upgrade.sh` compares the notice
with what you are running and prints **`upgrade_available = True`** only when there is
really something to apply.

**`status = refused reason = release marker is expired`** means the notice has run out and
the new one is not out yet. Nothing is wrong with your node — it keeps running and never
reads the notice. Try again later.

About `runtime_match = False`, which looks alarming:

- **Right after a restart it is normal.** The node only publishes its runtime profile once
  consensus has started. Wait for `rpc = ready` and look again.
- **Alongside `source_match = False`** it usually just means an update exists.

**If you are already validating:** never re-run `--sync`, never edit
`config/network.env`, never re-enroll. The upgrade handles the restart itself.

---

## How the rewards work

From the source, since the documentation is quiet on it:

- **There is no downtime penalty.** The only slashable offence is `vote_conflict` —
  signing two conflicting blocks. There is no evidence type for being absent. Being
  offline costs you the rewards of the epochs you missed, nothing else.
- **The only realistic way to lose your stake** is running two nodes with the same key.
  When moving machines: stop the old one, verify it is stopped, then start the new one.
- **Rewards:** 70% to whoever proposed the epoch, the other 30% split among everyone who
  signed the commit.
- **A node that is not in the active set earns nothing.**
- **Minimum bond: 1,000,000 raw units (1 OCT).** Keep extra for fees.
- To leave: `octra enroll.sh exit`, then `octra enroll.sh withdraw`. To come back after a
  long absence: `octra rejoin.sh`.

---

## Numbers I measured

If your machine is wildly different from these, something is off.

| | |
|---|---|
| `install.sh --source-build` | ~6 minutes on 6 cores |
| Update, start to finish | 10–30 minutes on 6 cores |
| State-sync snapshot | ~38 GB |
| Disk after first sync | ~44 GB, then flat — the node prunes its history every ~2 days |
| Memory right after a restart | ~3.5 GB |
| Memory once settled | ~6.5 GB, reached in the first 2–3 days and then flat (6.47 GB at 2½ days, 6.57 GB at 5 days) |
| Log file growth | ~90 MB/day, never rotated |
| Start to `rpc = ready` | 2–3 minutes |

**About memory:** most rented servers come with **no swap**. With no swap, running out of
memory does not slow the node down — the system kills it. That is why the guide asks for
12 GB: 8 GB leaves too little room once the node has settled, especially while an update
is compiling next to it.

---

## Where this guide differs from the official README

The official README (<https://github.com/octra-labs/lite_node>) is a correct, compact list
of commands. It is not a guide to operating a machine on the public Internet, and does not
claim to be. Everything below is something I added or changed, with the reason.

| # | What I do differently | Why |
|---|---|---|
| 1 | Add a firewall step; never expose 8080 | The control interface has no password. The README does not discuss ports. |
| 2 | Harden SSH, and verify with `sshd -T` | Ubuntu's default config silently overrides hardening put in the wrong file. |
| 3 | Cover provider choice, public IP, cloud firewalls | A validator without inbound reachability cannot work at all. |
| 4 | Print and check the advertised address before using it | A wrong address gives you a node nobody can reach, with no obvious symptom. |
| 5 | Run the long steps under `tmux` | A dropped connection otherwise kills a 38 GB download. |
| 6 | Add the `octra` shortcut command | Removes the most error-prone piece of syntax from every single command. |
| 7 | Explicit key backup step before funding | The README never mentions that a key was generated and is now your responsibility. |
| 8 | Test port 19000 from outside before enrolling | Better than enrolling and then debugging. |
| 9 | Hand the process manager to systemd, verify with matching PIDs | The installer enables the systemd unit but nothing ever starts the daemon through it, so its automatic restart never applies. |
| 10 | Say where `sudo -iu` is safe, instead of banning it | The team's own instructions use the login form. It is safe on stock Ubuntu with pm2 under systemd, and I verified that; it is fatal on WSL, and wherever pm2 was started outside systemd. A blanket ban teaches the wrong rule. |
| 11 | Check for updates every couple of days, treat `expires_at` as a deadline, and read `upgrade_available` rather than `action` | Required releases change consensus rules at a fixed point; a node left behind diverges. Every notice says `required`, even a re-issue of the same code. |
| 12 | Warn that `check.sh` prints test fixtures | They look exactly like real node state. |
| 13 | Explain that peer counters read zero harmlessly | Otherwise a healthy node looks disconnected. |
| 14 | Explain that bonded + ready ≠ active | Otherwise a correct node looks broken while it is merely waiting. |
| 15 | Explain what `storage.sh` reports and when to prune | The README lists the commands; this says when you need them. Old state directories are tens of gigabytes each. |
| 16 | Note PowerShell quoting for Windows users | The failure mode is a misleading "file not found". |
| 17 | Test for CGNAT before installing anything (home guide) | The README assumes a reachable host. Finding out at the end costs a day. |
| 18 | Document the WSL distro-shutdown behaviour | It looks exactly like a crash loop. |

---

*Written from a real deployment. If something here is out of date, please open an issue.*
