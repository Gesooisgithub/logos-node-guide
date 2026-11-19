# Logos node on a rented server: 15 steps

Follow in order. Each step has a command to paste and what you should see back. If you see
something else, check [reference.md](reference.md).

**Everything from Step 2 onward runs on the server, logged in as `root`.**

Versions in this guide: `blockchain_module` **0.3.4** and `logosctl` **0.3.2**, the latest
releases on 19 November 2026.

How I got here: I installed my node on a Windows PC, then moved it, keys and chain data
included, to a server where it runs under systemd. These steps follow what runs on that
server. The `logos` shortcut and the start script are tidied versions of the commands and the
script I use there.

---

## Step 1: Rent the server

No commands. When ordering, make sure you get:

- **2+ cores whose CPU has ADX.** Any AMD Zen or Intel Broadwell or newer has it, but some
  providers show a generic CPU model (`kvm64`, `qemu64`) that hides it, and then the node
  crashes on start. Step 5 checks it before you install anything
- **4+ GB RAM**: the node uses 1 to 4 GB, more during the first sync, and Step 2 adds swap
  ([numbers](reference.md#numbers-i-measured))
- **20+ GB on an SSD or NVMe disk.** The data stays around 2 GB, but the node writes to the
  disk all day ([why](reference.md#disk-writes))
- **Ubuntu 24.04** (or newer). Not 22.04: its system library is too old for the node
- **A public IPv4 address** and **root access over SSH**

**Which provider?** Mine runs on a **Contabo** VPS, next to an Octra validator on the same
machine. Any provider that meets the list above works.

If the provider offers its own cloud firewall, **leave it off**. You set up the firewall on
the machine itself in Step 4, and running both makes problems twice as hard to find.

---

## Step 2: Log in, update and add swap

```bash
ssh root@YOUR_SERVER_IP
```

```bash
apt-get update && apt-get upgrade -y && reboot
```

Wait a minute, then log back in.

Add 4 GB of swap. Without it, the system kills the node if memory runs out
([why](reference.md#memory)):

```bash
fallocate -l 4G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
swapon --show
```

**You should see:** a line starting with `/swapfile file 4G`.

> **On Windows?** In PowerShell, always put **double quotes outside, single quotes inside**
> for remote commands. Otherwise the quotes get eaten and you get confusing errors. See
> [reference.md](reference.md#windows-powershell-quoting).

---

## Step 3: Lock down SSH

**On your own computer**, make a key and copy it up:

```bash
ssh-keygen -t ed25519 -C "logos node"
```

```bash
ssh-copy-id root@YOUR_SERVER_IP
```

Windows has no `ssh-copy-id`. In PowerShell, use this instead:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh root@YOUR_SERVER_IP "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
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
permitrootlogin without-password
passwordauthentication no
```

(`without-password` is OpenSSH's other name for `prohibit-password`.)

**If it still says `yes`, do not reload: see [reference.md](reference.md#ssh-hardening-does-nothing).**

```bash
systemctl reload ssh
```

Open a third terminal and confirm you can still log in. Then close the others.

---

## Step 4: Firewall

```bash
apt-get install -y ufw
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp comment 'ssh'
ufw allow 3000/udp comment 'logos blockchain p2p'
ufw --force enable
ufw status
```

**You should see:** `22/tcp` and `3000/udp` allowed. **`8080` must not be there.**

The official guide opens five more ports. They belong to the storage and messaging modules,
which this guide does not run.

---

## Step 5: Check the CPU and the system

```bash
grep -o adx /proc/cpuinfo | head -1
ldd --version | head -1
```

**You should see:** `adx`, then a line ending in `2.39` or a higher number.

**No `adx`? Stop here.** The node will crash with `signal 4`. Ask the provider for the
`host` CPU model, or pick another server. ([details](reference.md#adx-and-signal-4))

---

## Step 6: Install `logosctl`

`logosctl` is the program that downloads, starts and talks to the node.

```bash
apt-get install -y curl jq tar fuse3
mkdir -p /root/logosctl-download && cd /root/logosctl-download
curl -fL -o logosctl-x86_64-linux.tar.gz \
  https://github.com/logos-co/logos-logoscore-cli/releases/download/0.3.2/logosctl-x86_64-linux.tar.gz
sha256sum --check <<'EOF'
294fbcc6409a8851b7471582bdad2dbfaf681b5d1c67dfffec38ada925dc6762  logosctl-x86_64-linux.tar.gz
EOF
tar -xzf logosctl-x86_64-linux.tar.gz
sha256sum --check <<'EOF'
dc2a7e67c7326dada2b9302fb15f9df2cacbe6de3b9db50502ad7831608ffd56  logosctl-x86_64.AppImage
EOF
```

**You should see:** `logosctl-x86_64-linux.tar.gz: OK` and `logosctl-x86_64.AppImage: OK`.

**If either says `FAILED`, stop.** The file is not the one the Logos team published.

```bash
install -m755 logosctl-x86_64.AppImage /usr/local/bin/logosctl
cd /root && rm -rf /root/logosctl-download
logosctl --version | head -1
```

**You should see:** `logosctl version 0.3.2`

The checksums are the ones in the official guide; the first one also matches the digest
GitHub shows for the release file.

> **ARM server?** I have only run x86_64. Download `logosctl-aarch64-linux.tar.gz` from the
> same release instead and check it against
> `e77ccd9c1a7e673c1a5533f2bcda03f84fd04e80f415ac414c378a897e5135fe` (GitHub's digest).
> List it with `tar -tzf` to get the AppImage's name. There is no published checksum for the
> ARM AppImage itself, so skip the second check.

---

## Step 7: Create the `logos` user and the `logos` shortcut

The node runs as its own user, with everything in `/var/lib/logos-node`:

```bash
useradd --system --home /var/lib/logos-node --create-home --shell /usr/sbin/nologin logos
mkdir -p /var/lib/logos-node/.logosctl
chown -R logos:logos /var/lib/logos-node
chmod 700 /var/lib/logos-node/.logosctl
```

Every `logosctl` command has to run as that user, from that folder: the node keeps its chain
data in `./state`, relative to the folder it was started from. This one-time shortcut does
both, so the rest of the guide stays short:

```bash
cat > /usr/local/bin/logos <<'EOF'
#!/bin/sh
cd /var/lib/logos-node || exit 1
exec runuser -u logos -- env HOME=/var/lib/logos-node \
  LOGOSCTL_CONFIG_DIR=/var/lib/logos-node/.logosctl /usr/local/bin/logosctl "$@"
EOF
chmod 755 /usr/local/bin/logos
logos --version | head -1
```

**You should see:** `logosctl version 0.3.2`

From now on you type `logos status` instead of the long form.

---

## Step 8: Install the blockchain module

```bash
printf '{}\n' | logos daemon config set -
logos daemon start --detach
logos catalog refresh
logos package install blockchain_module \
  --version 0.3.4 \
  --root-hash 911a849083727457c25df4c29b2f2efd7c5b2906cd8f45eaa7f0cd89178aa08e \
  --yes
```

**The install takes 2 to 3 minutes and prints nothing while it works.** That is normal.

```bash
logos package ls --type core
```

**You should see:** a line `blockchain_module   0.3.4   core   user` among a few others.

**Leave the daemon running** for the next step.

---

## Step 9: Make your keys

The node needs a list of peers to start from. These four are the ones in the
[0.3.4 release notes](https://github.com/logos-blockchain/logos-blockchain/releases/tag/0.3.4);
if a newer release lists others, use those.

```bash
cat > /var/lib/logos-node/peers.json <<'EOF'
{
  "initial_peers": [
    "/ip4/65.109.51.37/udp/3000/quic-v1/p2p/12D3KooWFrouXfmrR4nsLMtE7wu15DoMJ6VtoUtHinREZCvbWHar",
    "/ip4/65.109.51.37/udp/3001/quic-v1/p2p/12D3KooWJRGau8M1rjT7R5e4YYsgdFhsMX35nRDtMwCDjxQkXAHz",
    "/ip4/65.109.51.37/udp/3002/quic-v1/p2p/12D3KooWQXJavMDTRscjauFSgVAB1VLB6Rzpy2uY5SU9Tk7927tb",
    "/ip4/65.109.51.37/udp/50001/quic-v1/p2p/12D3KooWSQc7CcGtvWDPF1yCbBthFnQjprfCVHmfmNDUrSmqQsU1"
  ]
}
EOF
chown logos:logos /var/lib/logos-node/peers.json
```

Now generate your keys and your configuration, lock them down, and stop the daemon:

```bash
logos module load blockchain_module
logos call blockchain_module generate_user_config @peers.json
chmod 600 /var/lib/logos-node/user_config.yaml /var/lib/logos-node/keystore.yaml
logos daemon stop
ls -l /var/lib/logos-node
```

**You should see:** `keystore.yaml` and `user_config.yaml`, both `-rw------- 1 logos logos`.

`keystore.yaml` holds your keys. `user_config.yaml` only lists the peers, plus any setting
you change later.

**Run `generate_user_config` once per node.** A second run refuses while `user_config.yaml`
exists, and it never overwrites `keystore.yaml`. You only get new keys where `keystore.yaml`
is missing, for example on a new machine: that is why Step 10 matters.

---

## Step 10: Back up your keys

```bash
cat /var/lib/logos-node/keystore.yaml
```

Copy **the whole thing** into a password manager. Then close the terminal output.

**Do this before you mine.** The file says it too: do not share your secret keys. Nobody
ever needs this file from you.

`user_config.yaml` is worth keeping next to it (it is not secret). With both files you can
rebuild the node anywhere.

---

## Step 11: Run it under systemd

Two pieces: a service that keeps the daemon running, and a small script that loads and
starts the blockchain module every time the daemon starts. The official guide describes that
script but does not provide one; this is a shortened copy of the one I run.

**1. The start script:**

```bash
cat > /usr/local/bin/logos-node-start <<'EOF'
#!/bin/bash
# Load and start blockchain_module once the logosctl daemon answers.
# Run by logos-node-bootstrap.service after every daemon (re)start.
set -u
DIR=/var/lib/logos-node
API=http://127.0.0.1:8080
lc() { timeout 120 runuser -u logos -- env HOME="$DIR" LOGOSCTL_CONFIG_DIR="$DIR/.logosctl" /usr/local/bin/logosctl "$@"; }
running() { lc daemon status 2>/dev/null | jq -e '.daemon.status == "running"' >/dev/null 2>&1; }
loaded() { lc daemon status 2>/dev/null | jq -e '.modules[] | select(.name == "blockchain_module") | .status == "loaded"' >/dev/null 2>&1; }

cd "$DIR" || exit 1
for f in user_config.yaml keystore.yaml; do
  [ -s "$f" ] || { echo "Missing $DIR/$f: not starting blockchain_module."; exit 1; }
done
# The node syncs its clock over the network; right after boot DNS can take a minute.
for i in $(seq 1 36); do getent hosts pool.ntp.org >/dev/null && break; sleep 5; done
echo "Waiting for the daemon..."
for i in $(seq 1 60); do running && break; sleep 2; done
running || { echo "The daemon is not running after 2 minutes."; exit 1; }
if loaded; then
  echo "blockchain_module already loaded: not starting it again."
else
  echo "Loading and starting blockchain_module..."
  lc module load blockchain_module >/dev/null || { echo "module load failed"; exit 1; }
  # On a synced node the start replays stored blocks first and this call can time out.
  # Not fatal: the API check below decides.
  lc call blockchain_module start "$DIR/user_config.yaml" "" >/dev/null 2>&1
fi
for i in $(seq 1 160); do
  if info=$(curl -sf -m 5 "$API/cryptarchia/info"); then
    echo "blockchain_module is up: $(jq -r '.cryptarchia_info | "state=\(.state) height=\(.height)"' <<<"$info")"
    exit 0
  fi
  sleep 15
done
echo "No answer from the node API after 40 minutes. Log: $DIR/.logosctl/logs/daemon.log"
exit 2
EOF
chmod 755 /usr/local/bin/logos-node-start
```

**2. The two services:**

```bash
cat > /etc/systemd/system/logos-node.service <<'EOF'
[Unit]
Description=Logos node (logosctl daemon)
After=network-online.target
Wants=network-online.target logos-node-bootstrap.service

[Service]
User=logos
Group=logos
# The chain data goes to ./state, relative to this folder.
WorkingDirectory=/var/lib/logos-node
Environment=HOME=/var/lib/logos-node
Environment=LOGOSCTL_CONFIG_DIR=/var/lib/logos-node/.logosctl
ExecStart=/usr/local/bin/logosctl daemon start
ExecStop=/usr/local/bin/logosctl daemon stop
Restart=on-failure
RestartSec=10
TimeoutStopSec=120
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

cat > /etc/systemd/system/logos-node-bootstrap.service <<'EOF'
[Unit]
Description=Load and start blockchain_module in the Logos node
After=logos-node.service
BindsTo=logos-node.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/logos-node-start
TimeoutStartSec=45min
StandardOutput=journal
StandardError=journal
EOF
```

**3. Start it, and have it start at every boot:**

```bash
systemctl daemon-reload
systemctl enable --now logos-node
journalctl -u logos-node-bootstrap -f
```

**You should see, within a minute or two:** `Waiting for the daemon...`, then
`Loading and starting blockchain_module...`, then
`blockchain_module is up: state=Bootstrapping height=...`

Leave the log with **Ctrl+C** (the node keeps running).

Why two services: the module must be started after the daemon is up, and the official guide
warns against doing it from the daemon's own service, where a slow start can get the daemon
killed. `Wants=` in the daemon's service starts the script with the daemon, and `BindsTo=`
stops it with the daemon, so it runs again after every restart, including automatic ones.

---

## Step 12: Wait for `Online`

```bash
curl -s 127.0.0.1:8080/cryptarchia/info | jq
curl -s 127.0.0.1:8080/network/info | jq .n_peers
```

**You should see:** `"state": "Bootstrapping"` with a `height` that climbs quickly, and a
peer count well above zero within a minute.

The first sync happens in two parts:

1. **The download.** In October, at 43,000 blocks, mine took 11 minutes. The chain is more
   than twice as long now, so expect longer.
2. **Then one hour of waiting.** The height reaches the tip and moves one block at a time,
   and the state still says `Bootstrapping`. That hour is a fixed setting, not a problem.
   ([details](reference.md#the-hour-in-bootstrapping))

**Avoid restarting the node during this first sync.** The docs say it starts over; a test by
the developers on 0.3.4 shows it replays every block it already has first, about 22 minutes
with the API down, before going on.

Run the first command again after that hour.

**You should see:** `"state": "Online"` and `"phase": "Following"`.

Prove it survives a reboot:

```bash
systemctl reboot
```

Wait two minutes, log back in, and run the first command again. On my server it was
`Online` again 89 seconds after the reboot command, with no hour of bootstrapping, because it
was off for less than 20 minutes.

---

## Step 13: Get test tokens by mining

**Only once Step 12 says `Online`.** ([why](reference.md#pow_status-hangs-and-everything-after-fails))

The testnet hands out tokens through a small proof-of-work game, not a faucet. Start it:

```bash
logos call blockchain_module pow_start_mining
```

**You should see:** `"success": true`

Two minutes later:

```bash
logos call blockchain_module pow_status | jq -r .result.value | jq
```

**You should see:** `"is_mining": true`, `"is_armed": true`, and under `auto_claim.targets` a
`balance` that is no longer `"0"`.

Each claim pays about 1.9 million units. On one thread of my desktop CPU it made about one
claim a minute, and the default target of **2,000,000,000** took about 16 hours of mining.
**Then it stops by itself.** While it runs it uses one to two cores.

> **Mining does not survive a restart.** If the node restarts before the target is reached,
> run `pow_start_mining` again once it is `Online`.

---

## Step 14: Wait for your first block

Nothing to do for the first days. Your tokens are also your stake: with them the node can be
picked to propose blocks, but only after they have aged in.

The testnet counts in **epochs of 300,000 one-second slots, about 3.5 days.** Tokens count
once they have existed since the start of the previous epoch, so your node can start
proposing **3.5 to 7 days after you mined**. The release notes say about 3.5 hours; that is
wrong for this testnet. ([details](reference.md#epochs-and-your-first-block))

Even then, blocks are rare: with 2 billion units against the testnet's starting stake of
about 25 trillion, the odds give about one block every four or five days. Mine proposed six
in two weeks, so luck swings it a lot. Check for proposals and rewards with:

```bash
journalctl -u logos-node --since -7d | grep -c 'proposed block'
logos call blockchain_module wallet_get_claimable_vouchers | jq -r .result.value | jq
```

**You should see:** a count (0 for the first days), then `"vouchers": []` until a block of
yours has made it onto the chain.

If `vouchers` is not empty, claim them. Each call claims one voucher, so run it once per
voucher, a few seconds apart:

```bash
logos call blockchain_module leader_claim
```

`No claimable voucher found` means there is nothing to claim yet; try again later.

> **On 0.3.4, proposed blocks rarely make it onto the chain.** My node proposed six between
> 5 and 19 November and none of them stayed. Two bugs that may explain some of these are
> fixed in the coming 0.3.6; at least one loss came from a problem that is not fixed yet.
> ([details](reference.md#blocks-that-never-make-it))

---

## Step 15: Keep it updated

**Check for a new release every week or so** on the
[releases page](https://github.com/logos-blockchain/logos-blockchain/releases). Pre-releases
marked for internal devnet testing are not for this testnet.

To update, take the new version number and its root hash from the install step of the
[official guide](https://docs.logos.co/run-a-node), then:

```bash
systemctl stop logos-node
logos daemon start --detach
logos catalog refresh
logos package install blockchain_module --version NEW_VERSION --root-hash NEW_ROOT_HASH --yes
logos package ls --type core
```

**You should see:** `blockchain_module` with the new version. Then:

```bash
logos daemon stop
systemctl start logos-node
```

Once it answers again, `logos call blockchain_module version` prints the new version.

**Never run `generate_user_config` again for an update.** The new version reads your
existing files; 0.3.4 even rewrote mine into its new format and kept a `.bak` copy.

Then check it as in Step 12. If the update tells you to start from an empty state (a new
genesis), see [reference.md](reference.md#updates).

---

## Everyday commands

```bash
systemctl status logos-node                          # is the daemon up
logos status                                         # daemon and modules
curl -s 127.0.0.1:8080/cryptarchia/info | jq         # state and height
curl -s 127.0.0.1:8080/network/info | jq .n_peers    # peers
logos call blockchain_module pow_status | jq -r .result.value | jq   # balance, only when Online
journalctl -u logos-node -f                          # live log (full of harmless ERROR lines)
systemctl restart logos-node                         # restart
du -sh /var/lib/logos-node                           # disk space
```

**Is it really keeping up?** `Online` is not enough: a node can say it while it is hours
behind. Compare the slot of your newest block with the network's clock (one slot per
second):

```bash
now=$(logos call blockchain_module get_time_info | jq -r .result.value | jq .current_slot)
tip=$(curl -s 127.0.0.1:8080/cryptarchia/info | jq .cryptarchia_info.slot)
echo "behind by $((now - tip)) seconds"
```

**You should see:** a number under a couple of minutes. Blocks come every 30 to 60 seconds
on average, in uneven bursts. If it is hours and keeps growing, see
[reference.md](reference.md#falling-hours-behind).

---

## Something looks wrong?

Go to **[reference.md](reference.md)**: it has a symptom table and the explanation behind
every warning in this guide.
