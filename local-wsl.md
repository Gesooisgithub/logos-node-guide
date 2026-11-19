# Logos node on a Windows PC (WSL2)

This is how I first ran my node, over about a week (some three days of actual running),
before moving it to a server. It is a good way
to try Logos. **Read Step 0 before deciding to leave it running for months.**

---

## Step 0: Think about your SSD (1 minute)

The node writes to the disk all day: on my PC, **8.6 MB/s, about 0.75 TB a day**, while its
data stayed under 2 GB. A typical 1 TB consumer SSD is rated for about 600 TB written, so a
year of this uses almost half of its rated life.
([details](reference.md#disk-writes))

| Plan | Verdict |
|---|---|
| **Try it for a few days** | Fine. This guide is for you. |
| **Leave it on for months** | Use a rented server → [vps.md](vps.md). That is what I did. |

Two more things to know:

- **No open ports needed.** Behind my router, with nothing forwarded, the node found 140
  peers within a minute.
- **Switching the PC off costs nothing** except the blocks you miss. Off for more than 20
  minutes, the node spends an hour in `Bootstrapping` when it comes back.

---

## Step 1: Install Ubuntu 24.04 in WSL2

In PowerShell:

```powershell
wsl --install -d Ubuntu-24.04 --location D:\WSL\Ubuntu-24.04 --no-launch
```

`--location` puts the distro on another disk; change the path or drop it if C: has room. It
took 11 seconds here.

Ubuntu 24.04 is the oldest version that works: older ones have a system library too old for
the node.

Then open it:

```powershell
wsl -d Ubuntu-24.04
```

The first launch asks for a Unix user name and password. Any will do.

---

## Step 2: Check it

Inside Ubuntu:

```bash
ps -p 1 -o comm=
grep -o adx /proc/cpuinfo | head -1
ldd --version | head -1
free -h
```

**You should see:** `systemd`, then `adx`, then a line ending in `2.39`, then a `Mem` total
of at least 6 GB.

The 24.04 image already has systemd turned on, so there is nothing to edit. WSL gets half of
your PC's memory by default; the node used up to about 4 GB on mine, so a 16 GB PC is fine
without changing anything. **No `adx`? Stop:** the node will crash. ([why](reference.md#adx-and-signal-4))

---

## Step 3: Install the node

Open a root shell, so you can paste the server guide's commands as they are:

```bash
sudo -i
apt-get update && apt-get upgrade -y
```

Now do **[vps.md Steps 6 to 10](vps.md#step-6-install-logosctl)**, then come back here.

Skip everything before Step 6 (server, SSH, firewall), and skip Step 11: on a PC I ran the
node in a window instead, as below.

---

## Step 4: Start it in a window

The daemon runs in the foreground, in a window of its own. That window also keeps WSL from
shutting the distro down. Open a **new PowerShell window** and run:

```powershell
wsl.exe -d Ubuntu-24.04 -u root --cd /var/lib/logos-node --exec /usr/sbin/runuser -u logos -- env HOME=/var/lib/logos-node /usr/local/bin/logosctl daemon start
```

**Leave that window open.** It shows the daemon's messages.

Then, in your **Ubuntu root shell**, first check the network is up (right after a reboot it
can take a minute; [why](reference.md#cold-start-crash-on-wsl)):

```bash
getent hosts pool.ntp.org
```

**You should see:** a line with an IP address. Then load and start the blockchain module:

```bash
logos module load blockchain_module
logos call blockchain_module start /var/lib/logos-node/user_config.yaml ""
```

**You should see:** `"success": true` after up to 20 seconds.

On a later start, once the node has synced before, this call can end in a timeout instead,
while the node replays its last blocks. That is normal: do not run it again. Wait a minute
and check with `curl -s 127.0.0.1:8080/cryptarchia/info`.

Now **[vps.md Steps 12 and 13](vps.md#step-12-wait-for-online)**: wait for `Online`, then
mine. Skip the reboot test in Step 12.

---

## Step 5: Stop it, and start it again

To stop it, in the Ubuntu root shell:

```bash
logos daemon stop
```

In the other window the daemon shuts down within a few seconds and you get the PowerShell
prompt back.

To start it again, after a reboot or whenever you like, repeat **Step 4**: the window first,
then the network check, then the two `logos` commands. Then:

- **Off for less than 20 minutes:** `Online` again within one or two minutes.
- **Off for longer:** an hour in `Bootstrapping` first.
  ([details](reference.md#the-hour-in-bootstrapping))
- **Mining is always off after a start.** If it had not reached its target yet, run
  `logos call blockchain_module pow_start_mining` again once `Online`.

**Never start it while the same keys run anywhere else**, for example on a server you moved
it to.

---

## Step 6: Moving it to a server later

You can keep your keys and your tokens: stop it here first, then copy `keystore.yaml`,
`user_config.yaml` and the chain data to the server. I did exactly that.
([steps](reference.md#moving-a-node-to-another-machine))

When you are done with the PC copy for good, remove it with `wsl --unregister Ubuntu-24.04`.
**That deletes the keys inside it too**, so make sure your backup from vps.md Step 10 is
safe. If it says the distro is in use, run `wsl --shutdown` first (that also stops Docker
Desktop, if you use it).

---

## Everyday commands

In the Ubuntu root shell (`wsl -d Ubuntu-24.04`, then `sudo -i`):

```bash
logos status                                         # daemon and modules
curl -s 127.0.0.1:8080/cryptarchia/info | jq         # state and height
curl -s 127.0.0.1:8080/network/info | jq .n_peers    # peers
logos call blockchain_module pow_status | jq -r .result.value | jq   # balance, only when Online
logos daemon stop                                    # stop it
```

---

## Something looks wrong?

**[reference.md](reference.md)** has a symptom table, including the cold start crash I hit on
WSL.
