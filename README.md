# Octra Node Guide

Step-by-step guide to running a node on the Octra devnet — written by
doing it end to end on real machines and writing down everything that went wrong.

The node software itself lives in the official repository:
**<https://github.com/octra-labs/lite_node>**. This guide does not redistribute it. It
tells you how to run it, in the order things actually need to happen, and it flags every
place where we deliberately did something the official README does not describe.

> This is community documentation. It is not endorsed by Octra Labs. Where this guide and
> the official repository disagree, the official repository is authoritative — but read
> the "Where this guide departs from the official README" section of each guide first,
> because most disagreements are deliberate and explained.

---

## Start here: which guide do you need?

One question decides it.

> **Can a computer somewhere on the Internet open a TCP connection to port 19000 on the
> machine you want to use?**

| Your situation | Guide |
|---|---|
| A rented server with its own public IP address | **[vps.md](vps.md)** — recommended |
| A home PC running Windows | **[local-wsl.md](local-wsl.md)** — but do the reachability test in its first section *before installing anything* |

If you are not sure, you are probably in the second row. Many home internet connections
cannot accept inbound connections at all, no matter how you configure your router. The
first section of [local-wsl.md](local-wsl.md) shows you how to find out in about two
minutes, and it is the single most valuable thing in this repository: it can save you a
full day spent installing software that was never going to work.

**Running an observer only?** Then none of this matters — an observer makes only outbound
connections and needs no open ports. Either guide works, and you can skip every firewall
and port-forwarding step.

---

## Observer or validator?

You always start as an observer. There is no way to configure `--role validator`
directly; the `enroll.sh join` command is what promotes a synchronised observer into a
validator, and it refuses to do so until the node is ready.

|  | Observer | Validator |
|---|---|---|
| Follows the chain and serves data | yes | yes |
| Inbound port 19000 reachable from the Internet | not needed | **required** |
| Votes on blocks | no | yes |
| Earns rewards | **no** | yes |
| Tokens bonded | none | 1,000,000 raw units (= 1 OCT) minimum, plus a margin for fees |
| Can lose the bond | n/a | only by double signing |

An observer is a completely legitimate way to run a node. It is also the right way to
start: every validator is an observer that got promoted.

---

## What it takes

Requirements are written so they apply to any provider or machine. The right-hand column
is what we actually measured, not vendor claims.

| | Minimum that worked | Notes |
|---|---|---|
| CPU | 4 cores | 6 cores built the toolchain in about 6 minutes |
| RAM | 8 GB | the node settles at roughly 3–4 GB resident and stays there |
| Disk | 120 GB | about 44 GB after the initial sync, and it grows |
| OS | Ubuntu 22.04 or 24.04 | the installer expects `apt-get` |
| Network | a public IP address the machine actually owns | required for validating, not for observing |
| Bandwidth | enough to pull a ~38 GB snapshot once | this is the slowest step by far |

Budget **half a day** for a first install, most of it unattended waiting on the snapshot
download. The hands-on part is perhaps an hour.

On cost: a small virtual server of this size is an ordinary monthly commodity purchase,
and broadly comparable across providers. Compare on the specification above rather than
on a headline price.

---

## Non-negotiable safety rules

Read these once before you start. Each one is a way people lose money or lose a node.

1. **Never expose port 8080 to the Internet.** It is the node's RPC interface and it has
   no authentication whatsoever. It must be reachable from the machine itself and from
   nowhere else. Only port 19000 belongs on the public Internet.

2. **Never share the contents of `wallet.json`, and never paste it into a chat, a support
   ticket, a screenshot, or an AI assistant.** It is the key that signs blocks. Anyone
   who has it can impersonate your validator. To receive tokens from a faucet, your
   *public address* is the only thing anybody needs from you.

3. **Never run two nodes with the same key at the same time.** Double signing
   (`vote_conflict`) is the one and only offence that gets a bond slashed. When migrating
   a node to a new machine: stop the old one, verify it is stopped, and only then start
   the new one.

4. **Back up the identity before you put tokens on it**, and back up the whole
   `wallet.json` file rather than just the private key — the node wants all three fields
   (`priv`, `pub`, `address`) present and consistent when it restores.

5. **A validator key is by definition a hot key.** It has to sit in plaintext in the RAM
   of a machine that is always on. No configuration protects it from whoever controls
   that machine. Use it as a validator identity, never as a personal wallet.

Downtime, on the other hand, is **not** punished. There is no penalty in the code for
being offline — you simply miss the rewards of the epochs you were absent for. You can
stop and start your node whenever you like.

---

## Numbers we measured

Useful as a sanity check: if your machine is wildly off these, something is wrong.

| Step | Measured |
|---|---|
| `install.sh --source-build` | ~6 minutes on 6 vCPU |
| Building the node from source | ~10 minutes on 20 cores |
| State-sync snapshot | ~38 GB, download-bound |
| Disk used after first sync | ~44 GB |
| Node memory once settled | 3.2–3.8 GB resident, stable — this is not a leak |
| Time from start to RPC answering | 2–3 minutes; do not interrupt it |

---

## Where this guide departs from the official README

The official README is a correct and compact description of the commands. It is not a
description of how to operate a machine on the public Internet, and it does not claim to
be. We added the operational layer around it. Each guide lists its own departures in full
with reasons; in summary:

- We add a **firewall** step, and we never expose port 8080.
- We add **SSH hardening** before the machine is exposed, including a trap in Ubuntu's
  default configuration that silently keeps password logins enabled.
- We add an explicit **identity backup** step before the identity is funded.
- We **test that port 19000 is reachable from outside** before enrolling, instead of
  discovering the answer afterwards.
- We run the long unattended steps inside **tmux**, so a dropped connection does not kill
  a multi-hour download.
- We hand the pm2 process manager to **systemd** properly. The official installer
  registers a systemd unit, but nothing ever starts the daemon *through* it, so the
  unit's automatic restart never actually applies.
- We use `sudo -H -u octra` (non-login) instead of the README's `sudo -iu octra`
  (login), because on some systems closing a login session takes the node down with it.
- We treat announced releases marked **`action = required` as mandatory before their
  deadline**, because they change consensus rules at a fixed epoch and a node left behind
  will diverge from the network.

Every one of them is explained where it appears, marked like this:

> **Departure from the official README** — what we do instead, and why.

---

## Repository layout

| File | What it is |
|---|---|
| [`vps.md`](vps.md) | Full install on a rented server, from ordering to validating |
| [`local-wsl.md`](local-wsl.md) | Install on a Windows PC via WSL2, and how to tell in advance whether it can work |
| `.gitignore` | Refuses to commit key material, in case you keep notes inside a clone |
| `LICENSE` | MIT |

---

## Contributing

If a step is wrong, unclear, or has drifted out of date, open an issue. Include the
command you ran and the output you got — with addresses and key material removed.

---

## License

[MIT](LICENSE). Use it, adapt it, translate it, republish it — keep the copyright notice
with it.

The node software itself is a separate project with its own terms:
<https://github.com/octra-labs/lite_node>.
