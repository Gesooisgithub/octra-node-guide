# Octra Node Guide

Step-by-step guide to running a node on the Octra devnet — written by
doing it end to end on real machines and writing down everything that went wrong.

Copy, paste, check the expected output, move to the next step.

Node software: <https://github.com/octra-labs/lite_node>. This is unofficial community
documentation, written from a real install.

---

## Which guide?

**Can the Internet reach port 19000 on your machine?**

- **A rented server** → **[vps.md](vps.md)** ← start here, this is the easy path
- **A home PC on Windows** → **[local-wsl.md](local-wsl.md)** ← do its Step 0 first, it
  takes two minutes and often says "no"

Just want to follow the chain without validating? Either guide works and you can skip
every network step. You will not earn anything.

---

## Requirements

- A machine with **4+ cores, 12+ GB RAM, 120+ GB disk**, running **Ubuntu 22.04 or 24.04**
- A **public IP address** the machine actually owns (only if you want to validate)
- **1 OCT** (1,000,000 raw units) for the bond, plus a little for fees
- Half a day, most of it waiting on a download

---

## Five rules

1. **Never open port 8080.** It is the node's control interface and has no password.
2. **Never share `wallet.json`.** It signs your blocks. The faucet only needs your public
   `oct...` address.
3. **Never run two nodes with the same key.** It is the only way to lose your stake.
4. **Back up your key before putting tokens on it.**
5. **Being offline is not punished.** You just miss rewards while you are down.

---

## Files

| | |
|---|---|
| **[vps.md](vps.md)** | 16 steps, rented server |
| **[local-wsl.md](local-wsl.md)** | Windows PC via WSL2 |
| **[reference.md](reference.md)** | Troubleshooting, explanations, and how this differs from the official README |

---

## License

[MIT](LICENSE). Use it, adapt it, translate it — keep the copyright notice with it.
