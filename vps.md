# Running an Octra node on a rented server

This is the recommended path. A rented server has a public IP address it genuinely owns,
which is the one thing a validator cannot do without.

Every command below is run over SSH as `root` unless stated otherwise. Work through the
steps in order and check the expected output before moving on — several steps take a long
time, and starting them on a broken foundation wastes hours.

> **Placeholders.** Replace `<SERVER_IP>` with your server's public IP address and
> `<NODE_NAME>` with a short name of your choosing (letters, digits and dashes). Never
> paste your real address or key material into a chat, an issue, or a screenshot.

**Contents**

1. [Before you rent the machine](#1-before-you-rent-the-machine)
2. [First login](#2-first-login)
3. [Lock down SSH](#3-lock-down-ssh)
4. [Firewall](#4-firewall)
5. [Get the node software](#5-get-the-node-software)
6. [Install the toolchain](#6-install-the-toolchain)
7. [Verify the package](#7-verify-the-package)
8. [Configure, build and sync](#8-configure-build-and-sync)
9. [Back up your identity](#9-back-up-your-identity)
10. [Start the node](#10-start-the-node)
11. [Prove port 19000 is reachable](#11-prove-port-19000-is-reachable)
12. [Get tokens](#12-get-tokens)
13. [Enroll as a validator](#13-enroll-as-a-validator)
14. [Make the process manager survive](#14-make-the-process-manager-survive)
15. [Reading stat.sh](#15-reading-statsh)
16. [Keeping the node current](#16-keeping-the-node-current)
17. [Disk housekeeping](#17-disk-housekeeping)
18. [Stopping, leaving, withdrawing](#18-stopping-leaving-withdrawing)
19. [Validator economics](#19-validator-economics)
20. [Troubleshooting](#20-troubleshooting)
21. [Departures from the official README](#21-departures-from-the-official-readme)

---

## 1. Before you rent the machine

Specify the machine, not the brand. Any provider that gives you the following will work:

| Requirement | Why it matters |
|---|---|
| 4+ CPU cores | the toolchain is compiled from source on first install |
| 8+ GB RAM | the node settles at 3–4 GB resident |
| 120+ GB disk | ~44 GB after the first sync, and it grows with the chain |
| **A dedicated public IPv4 address** | a validator must be reachable from outside; an address shared behind provider NAT will not do |
| Ubuntu 22.04 or 24.04 | `install.sh` expects `apt-get` and is tested on Ubuntu |
| Root access over SSH | the installer creates a system user and a systemd unit |

The official README names Contabo among suitable providers, and that is what we used —
mentioned here as a concrete example, not a recommendation. Any provider meeting the
table above is equivalent for this purpose.

**Two things to get right at order time:**

- **Pick an Ubuntu image, not Debian.** The installer is written against `apt-get` but
  tested on Ubuntu; Debian is untested territory you do not want to be in on day one.
- **If the provider offers its own cloud firewall, leave it switched off.** You will
  configure `ufw` on the machine itself in step 4. Running both means every connectivity
  problem has two possible causes and you will chase the wrong one. Pick one layer — the
  on-machine firewall is the one this guide covers, and it is the one that travels with
  you if you change provider.

> **Departure from the official README** — the official README does not discuss provider
> selection, NAT, or firewalls at all. Everything in this section is ours.

---

## 2. First login

```bash
ssh root@<SERVER_IP>
```

```bash
apt-get update && apt-get upgrade -y
```

If the upgrade offers to restart services, let it. If it installs a new kernel, reboot
now — far better now than halfway through a sync.

```bash
reboot
```

Wait a minute, then log back in.

> **Windows users:** if you drive SSH from PowerShell, be careful with quoting. PowerShell
> strips double quotes when passing arguments to a native program, so a command like
> `ssh host "sh -c "cd /some/dir && ..."" ` arrives mangled on the server and fails in a
> confusing way. **Put double quotes on the outside and single quotes on the inside:**
> ```
> ssh root@<SERVER_IP> "sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/stat.sh'"
> ```
> This bit us repeatedly. The symptom is an error about a file "not found" that plainly
> exists, because the `cd` never happened.

---

## 3. Lock down SSH

Do this **before** the machine has anything worth stealing on it.

### 3.1 Put your public key on the server

From **your own computer**, not the server:

```bash
ssh-keygen -t ed25519 -C "octra node"
```

Accept the default path, and set a passphrase. Then copy the **public** half up:

```bash
ssh-copy-id root@<SERVER_IP>
```

If `ssh-copy-id` is unavailable (it often is on Windows), append the contents of your
`.pub` file to `/root/.ssh/authorized_keys` on the server by hand.

**Open a second terminal and confirm key login works before continuing.** Keep your
current session open the whole time. If you lock yourself out with the next step and have
no working session, you are reinstalling the machine.

### 3.2 Turn off password logins — and beware the override trap

Ubuntu's SSH configuration ends with an include directive, and **within SSH configuration
the first occurrence of a setting wins**, not the last. Files in
`/etc/ssh/sshd_config.d/` are read in lexical order, so a file named `50-…` beats a file
named `60-…`.

On a stock Ubuntu cloud image there is usually a `50-cloud-init.conf` containing
`PasswordAuthentication yes`. If you drop your hardening into a file numbered above 50,
**it is silently ignored** and password logins stay enabled. We hit exactly this, and the
configuration looked perfectly correct while doing nothing.

Write a file that sorts *before* it:

```bash
cat > /etc/ssh/sshd_config.d/10-hardening.conf <<'EOF'
PasswordAuthentication no
PermitRootLogin prohibit-password
KbdInteractiveAuthentication no
EOF
```

Now verify the **effective** configuration — not the files, the result:

```bash
sshd -T | grep -iE '^(passwordauthentication|permitrootlogin|kbdinteractiveauthentication)'
```

Expected:

```
passwordauthentication no
permitrootlogin prohibit-password
kbdinteractiveauthentication no
```

If `passwordauthentication` still says `yes`, something earlier in the order is winning.
Find it:

```bash
grep -rn PasswordAuthentication /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

Only once `sshd -T` reports what you want, apply it:

```bash
systemctl reload ssh
```

Then open a **third** terminal and confirm you can still log in. Only then close the
others.

> **Departure from the official README** — SSH hardening is not mentioned there. The
> override trap is the kind of thing that leaves a machine exposed while looking secured,
> which is why we verify with `sshd -T` rather than trusting the file we just wrote.

---

## 4. Firewall

Open only what is needed. **Port 8080 must never be exposed** — it is the node's RPC and
it has no authentication.

```bash
apt-get install -y ufw
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp comment 'ssh'
ufw allow 19000/tcp comment 'octra consensus'
ufw allow 9000/tcp comment 'octra p2p'
ufw --force enable
ufw status verbose
```

Expected: `22`, `9000` and `19000` allowed, everything else denied. Port 8080 must **not**
appear.

> **An observation about port 9000.** The node is configured with `--p2p-port 9000` and we
> opened it to match. In practice, on the build we ran, only **19000** and **8080** were
> ever listening, and every peer connection in the logs was to or from port 19000. Opening
> 9000 is harmless and matches the configuration flag, so we left it. Do not be alarmed if
> nothing ever binds to it.

Check what is actually listening once the node is up (step 10):

```bash
ss -ltnp | grep -E '8080|19000|9000'
```

`8080` must be bound, but it must be unreachable from outside — that is the firewall's
job, and it is why 8080 is not in the `ufw` list above.

---

## 5. Get the node software

```bash
apt-get install -y ca-certificates git
install -d -m 0755 /opt/octra
git clone --branch main --single-branch \
  https://github.com/octra-labs/lite_node.git \
  /opt/octra/libv_litecore
cd /opt/octra/libv_litecore
cat SOURCE_COMMIT
sha256sum -c config/network.env.sha256
```

Expected: `config/network.env: OK`.

That checksum line matters. `network.env` defines which chain you join. If it does not
verify, stop and find out why before going further.

---

## 6. Install the toolchain

This creates the `octra` system user, downloads a pinned OCaml and Rust toolchain into the
repository, installs system packages, and registers a `pm2` systemd unit.

```bash
cd /opt/octra/libv_litecore
env OCTRA_OPERATOR_USER=octra OCTRA_DATA_ROOT=/var/lib/octra \
  sh controls/install.sh --source-build
```

Expected at the end: `status = ready source_build = 1 user = octra`.

**Measured: about 6 minutes on 6 vCPU.** If you have seen an estimate of 30–60 minutes
elsewhere, including in our own earlier notes, it was pessimistic.

### About `sudo -H -u octra`

From here on, every command that touches the node runs as the `octra` user. This guide
always uses the **non-login** form:

```bash
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/stat.sh'
```

> **Departure from the official README** — the official README uses `sudo -iu octra`, the
> *login* form. On some systems, ending a login session kills every process that session
> owns, and that includes the pm2 daemon supervising your node. We first hit this on WSL,
> where it reliably killed the node.
>
> We later verified that on a stock Ubuntu server it is harmless, because
> `/etc/systemd/logind.conf` ships with `KillUserProcesses=no`: we ran an entire upgrade
> through `sudo -iu` and the node's process ID was unchanged afterwards. **So the official
> form is fine on a normal Ubuntu server.** We still teach the non-login form, because
> systemd's own upstream default is `KillUserProcesses=yes` and on a differently
> configured distribution the trap is real. The non-login form is correct everywhere; the
> login form is correct in most places.

---

## 7. Verify the package

```bash
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/check.sh'
```

Expected at the end: `status = pass gate = validator_tools`.

> **Read this before you read that output.** `check.sh` runs the project's test suite, and
> the test suite prints its **fixtures** — fake data — to the same stream. On a first run
> this is genuinely alarming: you will see lines like
>
> ```
> event = recovery status = rolled_back ...
> event = restore_restart status = held reason = identity_changed action = remain_stopped
> status = validator_active ... head_epoch = 41 voting = True
> event = state_sync_source status = ready source = https://seed-a.example
> event = bond status = resumed tx = aaaaaaaaaaaaaaaa... epoch = 101
> ```
>
> **None of that is your node.** The tells are unmistakable once you know them: epoch
> numbers in the tens or low hundreds (the real chain is in the millions), transaction IDs
> made entirely of repeated characters, hostnames ending in `.example`, and resource lines
> describing a machine that is not yours. The only lines that describe your node are the
> ones after `Ran NNN tests ... OK`.
>
> This is not a defect — it is a test suite doing its job on stdout. But nothing warns you,
> and it recurs every time you run `check.sh` or an upgrade.

---

## 8. Configure, build and sync

This is the long one: it builds the node and downloads a state snapshot of roughly 38 GB.

We wrap the configuration in a small script so that it is reproducible and so the
advertised address can be checked before it is used.

```bash
cat > /usr/local/bin/octra-configure.sh <<'EOF'
#!/bin/sh
set -eu
cd /opt/octra/libv_litecore
sha256sum -c config/network.env.sha256
NETSHA=$(cut -d" " -f1 config/network.env.sha256)
exec sh controls/config_val.sh \
  --role observer \
  --name NODE_NAME_PLACEHOLDER \
  --advertise IP_PLACEHOLDER:19000 \
  --api-port 8080 \
  --consensus-port 19000 \
  --p2p-port 9000 \
  --data-dir /var/lib/octra/devnet \
  --sync-stage /var/lib/octra/devnet.state_sync \
  --network config/network.env \
  --network-sha "$NETSHA" \
  --build \
  --sync \
  --yes
EOF
chmod 755 /usr/local/bin/octra-configure.sh
```

Fill in the two placeholders. The address is detected from outside, which is the value
that matters:

```bash
sed -i "s/NODE_NAME_PLACEHOLDER/<NODE_NAME>/" /usr/local/bin/octra-configure.sh
sed -i "s/IP_PLACEHOLDER/$(curl -s https://api.ipify.org)/" /usr/local/bin/octra-configure.sh
grep -E 'advertise|--name' /usr/local/bin/octra-configure.sh
```

**Check that line.** `--advertise` is how the rest of the network learns to reach you. If
it holds a private address (`10.x`, `192.168.x`, `172.16–31.x`) or the wrong host, no peer
will ever connect inbound and you will not understand why.

`--advertise` accepts a DNS name as well as an IP address. If your address can change, a
name that follows it is the better choice.

Now run it inside `tmux`, so that losing your SSH connection does not kill a multi-hour
download:

```bash
apt-get install -y tmux
tmux new -s octra
```

Inside the tmux session:

```bash
sudo -H -u octra /usr/local/bin/octra-configure.sh 2>&1 | tee /var/log/octra-config.log
```

Detach with `Ctrl+b` then `d`. Reattach later with `tmux attach -t octra`.

Expected, in order: `sync_start`, `sync_download_complete`, `sync_verified`, and finally:

```
event = configured role = observer address = oct...
```

**Write down that `oct...` address.** It is your node's identity, it is public, and you
will need it for the faucet.

> **Departure from the official README** — the README runs this interactively, prompting
> for the host and node name. On a remote server a dropped connection then kills a 38 GB
> download with no way to resume where you were. Running it under `tmux` and logging to a
> file costs one extra package and removes the whole failure mode.

---

## 9. Back up your identity

**Do this before you put a single token on the address.**

The identity lives at `/opt/octra/libv_litecore/.keys/validator/wallet.json`, mode `600`.

Open it on the server, copy the **entire file contents** into a password manager, and
close it:

```bash
cat /opt/octra/libv_litecore/.keys/validator/wallet.json
```

Rules, restated because this is where people lose everything:

- Save the **whole file**, not just the private key. On restore the node wants `priv`,
  `pub` and `address` all present and consistent.
- Put it in a password manager or another encrypted store. Not a text file, not a cloud
  note, not a chat message to yourself.
- **The faucet needs only the public address.** Nobody legitimate will ever ask for the
  file. Anyone who does is stealing your validator.
- Treat this key as a hot key that is only ever a validator identity — never a wallet you
  keep value in.

> **Departure from the official README** — the README does not mention backing up the
> identity. The node generates it silently during step 8, and it is easy not to realise
> you are now responsible for a key.

---

## 10. Start the node

```bash
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/run.sh'
```

If it refuses with `candidate paths are stale`, that is normal on a first run:

```bash
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/run.sh --rebind-runtime'
```

Expected: `status = started name = <NODE_NAME> role = observer`.

**Now leave it alone for two to three minutes.** The node replays epochs on startup and
opens its RPC and consensus ports only when that is finished. Until then `stat.sh` reports
`rpc = unavailable`, and that is correct behaviour, not an error. We measured about 105
seconds on one machine and just over two minutes on another.

Then:

```bash
ss -ltn | grep -E '8080|19000'
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/stat.sh'
```

Expected: `process = online`, `rpc = ready`, `peer_rpc = ready`, `head_epoch` within an
epoch or two of the network, and `promotion_readiness = ready`.

---

## 11. Prove port 19000 is reachable

**Do not skip this, and do not do it from the server itself** — the server can always
reach its own port, which tells you nothing.

From another machine:

```bash
nc -vz <SERVER_IP> 19000
```

Or use any "open port check" web service with port 19000 while the node is running.

Expected: the port is open. `Connection timed out` means packets are not arriving, and
enrolling as a validator will not work until that is fixed. On a properly provisioned
server this works first time; if it does not, check the provider's own cloud firewall
(section 1) before suspecting anything else.

> **Departure from the official README** — not mentioned there. Enrolling first and
> debugging afterwards is a much worse order to do this in.

---

## 12. Get tokens

There is no faucet in the repository. Devnet tokens are requested in Octra's community
channels.

You need at least **1,000,000 raw units (= 1 OCT)** for the bond, plus a margin for fees.

**Give out your public `oct...` address and nothing else.**

Check that they arrived:

```bash
curl -s https://devnet.octrascan.io/rpc -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"octra_account","params":["<YOUR_ADDRESS>",1]}'
```

Look for a `balance_raw` field of at least `1000000`.

---

## 13. Enroll as a validator

Only when **all** of these are true: port 19000 is reachable from outside, the node is
synchronised, `promotion_readiness = ready`, and the tokens have arrived.

```bash
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/enroll.sh join --amount 1000000'
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/enroll.sh status'
```

`join` bonds the stake, marks the node ready, waits for selection, and finally restarts
the node in consensus mode. **It can take several minutes. Do not interrupt it.**

### What "enrolled" actually means

Being bonded is not the same as validating. The node moves through states, and `stat.sh`
reports each one:

| Field | Meaning |
|---|---|
| `validator_bond` | stake is locked, with `validator_bonded_epoch` recording when |
| `validator_enrollment = ready` | the node has declared itself available |
| `validator_scheduled` | the network has scheduled it into an upcoming set |
| `validator_active` | it is in the active set and voting |

**You can sit at `ready` for a long time.** The size of the active set is a consensus
parameter, not simply "everyone who bonded". When we enrolled, the set was capped at the
number of already-active members and incumbents were preferred, so a correctly configured,
fully synced, bonded node with a healthy pulse simply waited — nothing was broken and
nothing needed restarting.

That particular cap was lifted by a scheduled protocol change (as a concrete example: in
late August 2026, an "open validator scheduling" release raised the cap and dropped
incumbent preference at a fixed epoch). The lesson generalises: **if you are `ready` but
not `active`, read the release notes before you touch anything.** Re-running
`enroll.sh join` will not help, and `enroll.sh activate` will refuse until the network
schedules you.

Meanwhile the node still does useful work and still accrues balance. Leave it running.

---

## 14. Make the process manager survive

> **This entire section is a departure from the official README, and it is the most
> consequential one.**

`install.sh` runs `pm2 startup systemd`, which **creates and enables** a
`pm2-octra.service` unit. But `run.sh` starts the node with the `pm2` command-line tool,
and that spawns its own pm2 daemon directly — not through systemd. The result:

```
systemctl is-enabled pm2-octra   ->  enabled
systemctl is-active  pm2-octra   ->  inactive
```

The unit is enabled, so **a reboot works**. But the daemon actually supervising your node
is an orphan that systemd knows nothing about, so the unit's `Restart=on-failure` never
applies. If that daemon dies on its own, nothing brings it back and your node is simply
gone until you notice.

**Do this while your node is not yet in the active set** — a restart then costs nothing at
all. The whole procedure takes about five minutes.

### 14.1 Stop the node cleanly

```bash
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/stop.sh'
```

Expected: `status = stopped name = <NODE_NAME>`.

### 14.2 Kill the orphan daemon and start it under systemd

```bash
sudo -H -u octra pm2 kill
systemctl start pm2-octra
systemctl is-active pm2-octra
sudo -H -u octra pm2 list
```

Expected: `active`, then a pm2 table.

**Your node will almost certainly show as `stopped` in that table, and that is expected.**
`stop.sh` saved the process list while the node was stopped, and `pm2 resurrect` — which
is what the systemd unit runs — faithfully restores it in the state it was saved in.

### 14.3 Start the node again

If the table shows `stopped`, start it properly. `run.sh` also re-saves the process list,
so the next reboot restores it running:

```bash
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/run.sh'
```

If the table already showed `online`, do **not** run `run.sh` — you would end up with two.
Just persist the state:

```bash
sudo -H -u octra pm2 save --force
```

### 14.4 Verify — this is the step that proves it worked

```bash
systemctl show -p MainPID --value pm2-octra
cat /home/octra/.pm2/pm2.pid
```

**The two numbers must be identical.** If they are, systemd owns the pm2 daemon and
`Restart=on-failure` is finally real. If they differ, the daemon is still an orphan and
the procedure did not take.

### 14.5 Prove it end to end with a reboot

The only real test:

```bash
systemctl reboot
```

Wait a couple of minutes and check:

```bash
sudo -H -u octra pm2 list
systemctl is-active pm2-octra
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/stat.sh'
```

Everything should be back with no intervention. A useful tell: the process IDs after a
reboot are very low numbers (three or four digits), which is itself evidence that the
node was started at boot rather than by hand.

**During the reboot, SSH will answer `Connection refused` for perhaps twenty seconds.**
That is the machine coming up, not a failure. Wait and retry.

**If something goes wrong at 14.2**, you are never stranded: `run.sh` from 14.3 brings the
node straight back up exactly as it was before you started, orphan daemon and all.

---

## 15. Reading `stat.sh`

```bash
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/stat.sh'
```

The fields worth knowing:

| Field | What good looks like |
|---|---|
| `process` / `pid` | `online` with a stable pid |
| `restarts` | `0`. A climbing number means something is killing the node |
| `rpc`, `peer_rpc` | `ready` (`unavailable` for the first 2–3 minutes after a start) |
| `epoch` vs `head_epoch` | within one of each other |
| `state_sync` | `verified` |
| `promotion_readiness` | `ready` before you can enroll |
| `voting` | `disabled` for an observer, enabled once in the active set |
| `validator_bond` | your bonded amount, once enrolled |
| `disk_free` | watch it; see [Disk housekeeping](#17-disk-housekeeping) |

### The peer counters, which will scare you

`p2p_connected`, `consensus_peers` and `round_peers` are **instantaneous samples**, and for
an observer they routinely read `0`.

This is not a broken network. Peers drop observers shortly after the handshake, so the
node reconnects continuously. In one 300-line window of our log we counted **78**
`event = connected` lines while `stat.sh` was reporting `p2p_connected = 0` in the same
minute — and the head kept advancing with zero lag the whole time.

**Judge connectivity by whether `head_epoch` is advancing and `epoch` is keeping up with
it, not by those counters.** To see the real picture:

```bash
sudo -H -u octra tail -n 300 /opt/octra/libv_litecore/data/operator_logs/node.log \
  | grep -c 'event = connected'
```

The log path comes from `pm2 describe <NODE_NAME>`, in the `out log path` row — a way to
find it that never touches the `.keys/` directory.

**One caveat:** once your node is in the active set, `round_peers = 0` stops being
innocuous, because a validator needs peers to take part in a round. Watch it at that
transition specifically.

---

## 16. Keeping the node current

```bash
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/upgrade.sh'
```

That is the dry run — it changes nothing. To apply:

```bash
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/upgrade.sh --apply' 2>&1 \
  | tee /var/log/octra-upgrade.log
```

Expected at the end: `status = observer_synced` (or `status = validator_active` if you are
in the set) with `binary_match = True`, `source_match = True`, `runtime_match = True` and
`lag = 0`.

The rebuild takes a while and dies if your SSH connection drops. Either run it in `tmux`,
or start it detached:

```bash
setsid nohup sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/upgrade.sh --apply' \
  > /var/log/octra-upgrade.log 2>&1 < /dev/null &
```

and follow it with `tail -f /var/log/octra-upgrade.log`.

### Upgrades are not optional

The dry run prints a line like:

```
event = release_marker source = origin sequence = 7 action = required \
  public_commit = <sha> source_commit = <sha> expires_at = <timestamp>
```

> **Departure from the official README** — the README presents `upgrade.sh` as a
> convenience. In practice a release marked `action = required` **changes consensus rules
> at a fixed activation epoch**. A node still running the previous release when that epoch
> arrives computes a different result from the rest of the network and diverges. We watched
> this happen: a node stuck on an older release died repeatedly at the same point until the
> fixed release landed.
>
> Treat `action = required` as a deadline, and apply it with room to spare. Check for
> updates at least weekly, and always after seeing an announcement.

Two things about `runtime_match = False`, which looks frightening and usually is not:

- Immediately after a restart it is **normal** — the node publishes its runtime profile
  only once consensus has started. Wait for `rpc = ready` and check again.
- If it persists alongside `source_match = False`, it usually just means a newer release
  exists and you are behind. Run the dry run and see.

**If you are already a validator:** do not re-run `--sync`, do not edit
`config/network.env`, and do not re-enroll. The upgrade script handles the restart.

---

## 17. Disk housekeeping

Each recovery or major upgrade may preserve the previous state directory:

```bash
ls -la /var/lib/octra/
du -sh /var/lib/octra/*
```

You will see `devnet` (live) alongside directories like `devnet.prior-<epoch>`. Each is
tens of gigabytes.

Keep the most recent one — it is your rollback if an upgrade goes wrong. Older ones, once
you have been running happily on a newer state for a while, are candidates for deletion
when you need space. **Never delete `devnet` itself**, and never delete anything while the
node is running.

---

## 18. Stopping, leaving, withdrawing

Stop the node (it is safe; downtime is not penalised):

```bash
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/stop.sh'
```

Leave the active set and get the bond back:

```bash
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/enroll.sh exit'
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/enroll.sh status'
sudo -H -u octra sh -c 'cd /opt/octra/libv_litecore && sh controls/enroll.sh withdraw'
```

To rejoin after a long absence, there is `controls/rejoin.sh`.

**When migrating to another machine: stop the old node, verify it is stopped, and only
then start the new one.** Two nodes signing with one key is the only way to lose a bond.

---

## 19. Validator economics

Taken from the source, because the documentation is quiet on it:

- **No downtime penalty exists.** There is exactly one slashable proof type,
  `vote_conflict` — double signing. There is no evidence type for absence or inactivity
  and no removal-for-downtime logic. Being offline costs you the rewards of the epochs you
  missed, nothing more.
- **Rewards:** 70% to the proposer of the epoch, the remaining 30% split among the
  participants in the commit — that is, those who actually signed.
- **An observer votes on nothing and earns nothing.** Only members of the active set are
  paid.
- **The minimum bond is 1,000,000 raw units (1 OCT).** Keep a margin above it for fees.

---

## 20. Troubleshooting

| Symptom | What is going on |
|---|---|
| `rpc = unavailable` just after starting | Normal for 2–3 minutes. Querying too early and restarting in a panic means it never finishes. Wait. |
| `run.sh` refuses: `candidate paths are stale` | Normal on first run. Use `run.sh --rebind-runtime`. |
| Alarming events during `check.sh` or an upgrade | Test fixtures, not your node. See [section 7](#7-verify-the-package). Look for epochs in the tens, transaction IDs of repeated characters, `.example` hostnames. |
| `p2p_connected = 0` | Usually fine for an observer. See [section 15](#15-reading-statsh). Judge by whether the head is advancing. |
| `source_match = False`, `runtime_match = False` | Either a restart that has not settled, or a newer release exists. Run the upgrade dry run. |
| Port 19000 times out from outside | The provider's own cloud firewall, a wrong `--advertise` value, or `ufw`. Check in that order. |
| "File not found" for a file that exists (Windows) | PowerShell quoting. Double quotes outside, single quotes inside. See [section 2](#2-first-login). |
| Node disappears and nothing restarts it | The pm2 daemon was an orphan. See [section 14](#14-make-the-process-manager-survive). |
| SSH says `Connection refused` right after a reboot | The machine is still booting. Wait twenty seconds. |
| Bonded and `ready` but never `active` | Probably a set-size cap, not a fault. See [section 13](#13-enroll-as-a-validator). Read release notes before changing anything. |

---

## 21. Departures from the official README

Collected in one place, with reasons.

| # | What we do differently | Why |
|---|---|---|
| 1 | Add a `ufw` firewall step; never expose 8080 | The RPC has no authentication. The README does not discuss ports. |
| 2 | Harden SSH before exposing the machine, and verify with `sshd -T` | Ubuntu's `50-cloud-init.conf` overrides higher-numbered files, so hardening can silently do nothing. |
| 3 | Discuss provider choice, dedicated public IP, and cloud firewalls | A validator without inbound reachability cannot work, and two firewalls make diagnosis twice as hard. |
| 4 | Wrap configuration in a script and check `--advertise` before use | A wrong advertised address produces a node nobody can reach, with no obvious symptom. |
| 5 | Run the long steps under `tmux`, log to a file | A dropped SSH connection otherwise kills a ~38 GB download. |
| 6 | Explicit identity backup step before funding | The README never mentions that a key was generated and is now your responsibility. |
| 7 | Test port 19000 from outside before enrolling | Better order of operations than enrolling and debugging afterwards. |
| 8 | Hand the pm2 daemon to systemd, verify with matching PIDs | `install.sh` enables the unit but nothing starts the daemon through it, so `Restart=on-failure` never applies. |
| 9 | Use `sudo -H -u` instead of `sudo -iu` | Harmless on stock Ubuntu (`KillUserProcesses=no`), fatal where systemd's upstream default applies. Non-login is correct everywhere. |
| 10 | Treat `action = required` releases as deadlines | They change consensus rules at a fixed epoch; a node left behind diverges. |
| 11 | Warn that `check.sh` prints test fixtures | They look exactly like real node state and cause real panic. |
| 12 | Explain that peer counters read zero harmlessly | Otherwise a healthy observer looks disconnected. |
| 13 | Explain that bonded + ready ≠ active | Otherwise a correctly configured node looks broken while it is merely waiting. |
| 14 | Document disk growth from preserved state directories | They are tens of gigabytes each and accumulate silently. |
| 15 | Note PowerShell quoting for Windows operators | The failure mode is a misleading "file not found". |

---

*Written from a real deployment. If a step has drifted out of date, please open an issue.*
