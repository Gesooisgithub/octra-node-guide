# Octra node on a rented server — 16 steps

Follow in order. Each step has a command to paste and what you should see back. If you see
something else, check [reference.md](reference.md).

**Everything from Step 2 onward runs on the server, logged in as `root`.**

---

## Step 1 — Rent the server

No commands. When ordering, make sure you get:

- **4+ cores, 8+ GB RAM, 120+ GB disk**
- **Ubuntu 24.04** (or 22.04) — not Debian
- **A dedicated public IPv4 address**
- **Root access over SSH**

**Which provider?** **Contabo** is the usual recommendation for this — the official Octra
README names it too, and it is what this guide was written on. Any provider that meets the
list above works just as well.

If the provider offers its own cloud firewall, **leave it off**. You will set up the
firewall on the machine itself in Step 4, and running both makes problems twice as hard to
find.

---

## Step 2 — Log in and update

```bash
ssh root@YOUR_SERVER_IP
```

```bash
apt-get update && apt-get upgrade -y && reboot
```

Wait a minute, then log back in.

> **On Windows?** In PowerShell, always put **double quotes outside, single quotes inside**
> for remote commands. Otherwise the quotes get eaten and you get confusing "file not
> found" errors. See [reference.md](reference.md#windows-powershell-quoting).

---

## Step 3 — Lock down SSH

**On your own computer**, make a key and copy it up:

```bash
ssh-keygen -t ed25519 -C "octra node"
```

```bash
ssh-copy-id root@YOUR_SERVER_IP
```

**Open a second terminal and check the key login works. Keep both open.**

Now **on the server**, turn off password logins:

```bash
cat > /etc/ssh/sshd_config.d/10-hardening.conf <<'EOF'
PasswordAuthentication no
PermitRootLogin prohibit-password
KbdInteractiveAuthentication no
EOF
```

```bash
sshd -T | grep -iE '^(passwordauthentication|permitrootlogin)'
```

**You should see:**

```
passwordauthentication no
permitrootlogin prohibit-password
```

**If it still says `yes`, do not reload — see [reference.md](reference.md#ssh-hardening-does-nothing).**

```bash
systemctl reload ssh
```

Open a third terminal and confirm you can still log in. Then close the others.

---

## Step 4 — Firewall

```bash
apt-get install -y ufw
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp comment 'ssh'
ufw allow 19000/tcp comment 'octra consensus'
ufw allow 9000/tcp comment 'octra p2p'
ufw --force enable
ufw status
```

**You should see:** `22`, `9000` and `19000` allowed. **`8080` must not be there.**

---

## Step 5 — Download the node

```bash
apt-get install -y ca-certificates git tmux
install -d -m 0755 /opt/octra
git clone --branch main --single-branch \
  https://github.com/octra-labs/lite_node.git \
  /opt/octra/libv_litecore
cd /opt/octra/libv_litecore
sha256sum -c config/network.env.sha256
```

**You should see:** `config/network.env: OK`

---

## Step 6 — Install the toolchain

```bash
cd /opt/octra/libv_litecore
env OCTRA_OPERATOR_USER=octra OCTRA_DATA_ROOT=/var/lib/octra \
  sh controls/install.sh --source-build
```

**You should see:** `status = ready source_build = 1 user = octra`

Takes about **6 minutes**.

---

## Step 7 — Add the `octra` shortcut

Every node command has to run as the `octra` user from a specific directory. This one-time
shortcut does that for you, so the rest of the guide stays short:

```bash
cat > /usr/local/bin/octra <<'EOF'
#!/bin/sh
exec sudo -H -u octra sh -c "cd /opt/octra/libv_litecore && sh controls/$*"
EOF
chmod 755 /usr/local/bin/octra
```

From now on you type `octra stat.sh` instead of the long form.

---

## Step 8 — Check the package

```bash
octra check.sh
```

**You should see, at the very end:** `status = pass gate = validator_tools`

> **Ignore everything above that line.** It is a test suite printing fake data — you will
> see scary things like `status = validator_active`, `head_epoch = 41`, and
> `event = recovery status = rolled_back`. **None of it is your node.**
> ([why](reference.md#checksh-prints-alarming-nonsense))

---

## Step 9 — Configure and sync

This downloads about **38 GB**. Run it inside `tmux` so it survives a dropped connection.

```bash
tmux new -s octra
```

Inside tmux, set your details and **check the address printed**:

```bash
NODE_NAME=my-octra-node
PUBLIC_IP=$(curl -s https://api.ipify.org)
echo "Name: $NODE_NAME   Public address: $PUBLIC_IP"
```

**That address must be your server's real public IP.** If it looks wrong, stop.

```bash
cd /opt/octra/libv_litecore
NETSHA=$(cut -d' ' -f1 config/network.env.sha256)
sudo -H -u octra sh -c "cd /opt/octra/libv_litecore && sh controls/config_val.sh \
  --role observer --name $NODE_NAME --advertise $PUBLIC_IP:19000 \
  --api-port 8080 --consensus-port 19000 --p2p-port 9000 \
  --data-dir /var/lib/octra/devnet --sync-stage /var/lib/octra/devnet.state_sync \
  --network config/network.env --network-sha $NETSHA \
  --build --sync --yes" 2>&1 | tee /var/log/octra-config.log
```

Detach with **Ctrl+b** then **d**. Come back later with `tmux attach -t octra`.

**You should see, in order:** `sync_start` → `sync_download_complete` → `sync_verified` →
`event = configured role = observer address = oct...`

**Write down that `oct...` address.** It is your node's public identity.

---

## Step 10 — Back up your key

```bash
cat /opt/octra/libv_litecore/.keys/validator/wallet.json
```

Copy **the whole thing** into a password manager. Then close the terminal output.

**Do this before you put any tokens on it.** Nobody ever needs this file from you — the
faucet only needs the `oct...` address from Step 9.

---

## Step 11 — Start the node

```bash
octra run.sh
```

If it refuses with `candidate paths are stale`:

```bash
octra run.sh --rebind-runtime
```

**Now wait 3 minutes without touching it.** The node replays history before it opens up.
`rpc = unavailable` during this time is normal.

Then:

```bash
octra stat.sh
```

**You should see:** `process = online`, `rpc = ready`, `peer_rpc = ready`, and `epoch`
within one or two of `head_epoch`.

---

## Step 12 — Check you are reachable

**From another machine, not the server:**

```bash
nc -vz YOUR_SERVER_IP 19000
```

**You should see:** the port is open.

If it times out, fix it before going further — you cannot validate without this.
([what to check](reference.md#port-19000-is-not-reachable))

---

## Step 13 — Get tokens

You need **1,000,000 raw units (1 OCT)** for the bond, plus a little extra for fees.

### 13a. Find your address

It is the `oct...` string printed at the end of Step 9. If you did not write it down:

```bash
octra stat.sh | grep '^address'
```

**You should see:** `address = oct...` followed by a long string.

That is your **public** address. It is safe to share and it is the only thing the faucet
needs. **Never send anyone the contents of `wallet.json`** — that is the key itself, and
nobody legitimate will ever ask for it.

### 13b. Claim from the faucet bot

The claim is made through the Telegram bot **[@octradevbot](https://t.me/octradevbot)**.

Open it in Telegram, start it, and give it the `oct...` address from 13a when it asks.

### 13c. Check they arrived

```bash
curl -s https://devnet.octrascan.io/rpc -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"octra_account","params":["YOUR_OCT_ADDRESS",1]}'
```

**You should see:** `balance_raw` of at least `1000000`.

---

## Step 14 — Become a validator

Only when Steps 11, 12 and 13 all passed.

```bash
octra enroll.sh join --amount 1000000
```

**This takes several minutes. Do not interrupt it.**

```bash
octra enroll.sh status
```

**You should see:** `state = ready` and your `bond = 1000000`.

> **Bonded is not yet validating.** You may sit at `ready` for hours or days waiting for a
> free seat. That is normal and nothing is broken — do not re-run `join`.
> ([why](reference.md#bonded-and-ready-but-never-active))

---

## Step 15 — Make it survive

Out of the box, if the process manager dies, nothing restarts your node. Fix it once:

```bash
octra stop.sh
sudo -H -u octra pm2 kill
systemctl start pm2-octra
sudo -H -u octra pm2 list
```

Your node will show as **`stopped`** in that list. That is expected. Start it:

```bash
octra run.sh
```

Now check the fix took:

```bash
systemctl show -p MainPID --value pm2-octra
cat /home/octra/.pm2/pm2.pid
```

**The two numbers must be identical.** If they are, you are done.

Prove it with a reboot:

```bash
systemctl reboot
```

Wait two minutes, log back in, and run `octra stat.sh`. Everything should be back on its
own. (SSH says `Connection refused` for ~20 seconds during the reboot. Normal.)

---

## Step 16 — Keep it updated

**Check weekly.** Updates are not optional — they change the rules of the network on a
deadline, and a node left behind stops working.

```bash
octra upgrade.sh
```

**Look for:** `action = required` and an `expires_at` date. If you see them, apply before
that date:

```bash
setsid nohup octra upgrade.sh --apply > /var/log/octra-upgrade.log 2>&1 < /dev/null &
```

Follow along with:

```bash
tail -f /var/log/octra-upgrade.log
```

It takes **20–30 minutes** and goes quiet for long stretches while it compiles. That is
normal.

**You should see, at the end:** `status = observer_synced` (or `validator_active`) with
`binary_match = True`, `source_match = True`, `runtime_match = True`, `lag = 0`.

---

## Everyday commands

```bash
octra stat.sh                 # how is it doing
octra enroll.sh status        # bond and validator state
octra upgrade.sh              # is there an update (checks only)
octra stop.sh                 # stop it
octra run.sh                  # start it
df -h /                       # disk space
```

---

## Something looks wrong?

Go to **[reference.md](reference.md)** — it has a symptom table and the explanation behind
every warning in this guide.
