# Logos Node Guide

Step-by-step guide to running a Logos blockchain node on the public testnet, written from a
real install: on a Windows PC first, then moved to a rented server where it runs under
systemd, writing down everything that went wrong.

Copy, paste, check the expected output, move to the next step.

Node software: <https://github.com/logos-blockchain/logos-blockchain>, run through `logosctl`
(<https://github.com/logos-co/logos-logoscore-cli>). Official docs:
<https://docs.logos.co/run-a-node>. This is unofficial community documentation, written from
a real install, and it covers the blockchain node only.

**Testnet tokens have no value, and there is no reward programme for running a node.** You
do it to learn the stack and to help test it.

---

## Which guide?

**Will this machine stay on, and can its SSD take a lot of writing?**

- **A rented server** → **[vps.md](vps.md)** ← start here, this is the setup I keep running
- **A home PC on Windows** → **[local-wsl.md](local-wsl.md)** ← fine for trying it for a few
  days; read its Step 0 first

You do not need an open port at home: behind my router, with nothing forwarded, the node
found 140 peers within a minute.

---

## Requirements

- A 64-bit CPU with **2+ cores**. On Intel or AMD it must have **ADX** (any AMD Zen, Intel
  Broadwell or newer), and some cheap servers hide it
- **4+ GB RAM** plus swap. The official minimum of 1 GB is not enough: mine used up to about
  4 GB. A Windows PC needs 12+ GB, since WSL gets only half of it
- **20+ GB on an SSD**. The data is small (about 2 GB), but the node writes about
  **1.4 TB a day** to the disk ([why](reference.md#disk-writes))
- **Ubuntu 24.04** or newer. Ubuntu 22.04 is too old
- About two hours for the install and the first sync, then most of a day of mining for test
  tokens

---

## Five rules

1. **Never open port 8080.** It is the node's API and has no password. It listens on
   `127.0.0.1` only; keep it that way.
2. **Never share `keystore.yaml`.** It holds your secret keys. Nobody needs it from you.
3. **Never run two nodes with the same keys.** The docs ask for a fresh set per node. When
   you move a node, stop the old one before you start the new one.
4. **Back up `keystore.yaml` before you mine.** Your test tokens live on those keys.
5. **Never ask for mining status before the node is `Online`.** `pow_status` hangs while the
   node bootstraps, and every call after it fails until a restart.
   ([why](reference.md#pow_status-hangs-and-everything-after-fails))

---

## Files

| | |
|---|---|
| **[vps.md](vps.md)** | 15 steps, rented server, under systemd |
| **[local-wsl.md](local-wsl.md)** | Windows PC via WSL2, for a trial run |
| **[reference.md](reference.md)** | Troubleshooting, explanations, numbers I measured, and how this differs from the official guide |

---

## License

[MIT](LICENSE). Use it, adapt it, translate it, and keep the copyright notice with it.
