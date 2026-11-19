# Reference

Everything that would have cluttered the step-by-step guides: what to do when something
looks wrong, why each warning exists, and where this guide deliberately differs from the
official docs.

You do not need to read this. Come here when something surprises you.

Versions: `blockchain_module` 0.3.4, `logosctl` 0.3.2, checked on 19 November 2026.

---

## Symptom table

| What you see | What it means |
|---|---|
| `Bootstrapping` for an hour with the height already at the tip | Normal. [Details below](#the-hour-in-bootstrapping). |
| Back in `Bootstrapping` for an hour after a restart | It was off for more than 20 minutes. Shorter stops skip it. |
| `RPC call failed` just after a start | Normal for a short while: the node replays its last 1,000 blocks first. The docs say it can take 10 minutes or more; on my server it took under 30 seconds. |
| `pow_status` never returns, then every call fails | It was called before `Online`. [Details below](#pow_status-hangs-and-everything-after-fails). |
| The node dies at once with `signal 4` | The CPU has no ADX, or the server hides it. [Details below](#adx-and-signal-4). |
| `No suitable fusermount binary found` | FUSE is missing: `apt-get install -y fuse3`. |
| The node stops itself and the log says `Initial Block Download failed: AllPeersFailed` | None of the four starting peers answered. Check that outgoing UDP is allowed. If the latest release notes list other peers, replace the list under `network: backend: initial_peers:` in `user_config.yaml` (and under `cryptarchia: network: bootstrap: ibd: peers:` if that exists too), then `systemctl restart logos-node`. Editing `peers.json` changes nothing after Step 9. |
| Thousands of `ERROR` lines in the log | Mostly harmless. [Details below](#the-log-is-full-of-errors). |
| The tip falls hours behind, and the log shows `BatchDeferredProofVerification(InvalidZkSignatures)` | A problem on 0.3.4 since mid November. It recovers by itself. [Details below](#falling-hours-behind). |
| `proposed block` in the log, but no reward | The block lost a fork. [Details below](#blocks-that-never-make-it). |
| Mining stopped by itself | It reached its target: `Every PoW auto-claim target reached its threshold; stopping mining`. |
| `"is_mining": false` after a restart | Mining never survives a restart. Start it again once `Online`. |
| `wallet_get_claimable_vouchers` stays empty for days | Normal: your stake needs up to two epochs, about a week, to count. [Details below](#epochs-and-your-first-block). |
| `Process did not terminate gracefully, killing: blockchain_module` when stopping | Happened on both stops of my server node (9 and 14 November). It came back fine both times. |
| `NTP resync moved backwards ... pausing until NTP catches up` | Harmless: the node's slot clock got one second ahead and waits. |
| The node restarted by itself one morning, with no error | Ubuntu's automatic updates restarted it after a system library update (mine on 14 November). It comes back by itself. |
| The disk is busy all the time | It writes about 16 MB/s around the clock. [Details below](#disk-writes). |
| Memory far above the official 1 GB | Normal. [Details below](#memory). |
| On WSL, the daemon crashed during a cold start | [Details below](#cold-start-crash-on-wsl). |
| "File not found" for a file that exists (Windows) | [PowerShell quoting](#windows-powershell-quoting). |

---

## The hour in `Bootstrapping`

A start goes through named phases, and the log prints each one:

1. **Replay.** The node reads back its last 1,000 stored blocks
   (`found 1000 stored blocks to replay`). Seconds on a new node; 24 to 29 seconds on mine.
2. **Initial Block Download** (`entering InitialBlockDownload phase`). It downloads what it
   missed from peers. On a first sync this is the long part.
3. **Prolonged bootstrap period** (`entering ProlongedBootstrapPeriod phase`). The height is
   at the tip, blocks arrive one at a time, and the state still says `Bootstrapping`. This
   lasts **exactly one hour**: it is a setting
   (`cryptarchia.service.bootstrap.prolonged_bootstrap_period: 1h`), not a problem.
4. **Following** (`entering Following phase`, then `Chain is online. Starting block
   proposals.`). The state is now `Online`.

**Phase 3 is skipped after a short stop.** A node that was off for less than 20 minutes
(`offline_grace_period.grace_period: 20m`) logs `chain is already not in bootstrapping
state` and goes straight to `Online`: one to one and a half minutes on my server, three times
out of three. After 45
minutes off, mine went back into it; after two hours off, and after five days, it sat
through the full hour again.

My first sync, in October at 43,000 blocks: download in 11 minutes, `Online` 71 minutes after
the start. After five days off: 16,700 blocks in 12 minutes, `Online` after 72 minutes.

**Avoid restarting during the very first sync.** The docs say it starts over; a test in the
release work for 0.3.6 shows a 0.3.4 node with no final block yet replaying every block it
has with full checks, about 22 minutes at today's height, with the API down. Either way,
leave it alone until it is `Online` once. (The start script waits up to 40 minutes for the
API for this reason.)

---

## `pow_status` hangs and everything after fails

While the node bootstraps, `pow_status` and `pow_claimable_rewards` never return. Worse, every
`logosctl call` after that fails too, until the daemon is stopped. The docs warn about it,
and I hit a similar lockup from repeated calls during a slow start
([below](#cold-start-crash-on-wsl)).

**Fix:** `systemctl restart logos-node` (on WSL: `logos daemon stop`, then start it again).
Then wait for `Online` before asking about mining. The start script in
[vps.md Step 11](vps.md#step-11-run-it-under-systemd) never calls them.

The node's HTTP API is safe at any time, which is why this guide checks the state with
`curl 127.0.0.1:8080/cryptarchia/info` instead of a `logosctl call`. The two give the same
numbers; the call names the field `mode`, the API names it `state`.

---

## ADX and `signal 4`

On Intel and AMD, the node needs the CPU's ADX instructions. Every AMD Zen and every Intel
chip since Broadwell has them. A virtual server can still hide them: some providers present a
generic CPU model (`kvm64`, `qemu64`) to the machine, and then the node crashes at once with
`signal 4`.

```bash
grep -o adx /proc/cpuinfo | head -1
```

No `adx` means no node on that machine as it is. On Proxmox or QEMU you set the CPU type to
`host`; on a rented server you ask the provider, or move.

---

## The log is full of errors

The node logs at `ERROR` level for things that are routine on a busy peer-to-peer network.
On my server, the last 24 hours before writing this held about 16,000 `ERROR` lines and zero
panics. These kinds are harmless:

| Line | What it is |
|---|---|
| `Failed to create a block stream: requested target block is unavailable` | A peer asked for a block this node does not keep. |
| `failed to send a block download request ... failed to open stream` | A peer went away. |
| `Sending response failed ... sending stopped by peer` | A peer closed the stream first. |
| `Failed to send blocks to peer` | Same. |
| `Failed to reconstruct block from proposal: UnresolvedReference` | A proposal refers to a transaction this node has not seen yet. The block arrives anyway. |
| `Could not reinsert a reorged tx into mempool` | After a small reorganisation. |
| `Failed to process block: AlreadyApplied` | The same block came from sync and from gossip. |
| `Failed to process block: ParentMissing` | A block arrived before its parent. A few are normal; a steady stream means the node is behind ([below](#falling-hours-behind)). |

The level names (`ERROR`, `INFO`) carry colour codes in the log and the journal. To count by
level, strip the codes first:

```bash
journalctl -u logos-node --since -1h | sed 's/\x1b\[[0-9;]*m//g' | grep -c ' ERROR '
```

The node also writes hourly files in `/var/lib/logos-node`, keeping the last 10 hours, and the
daemon keeps its own log in `/var/lib/logos-node/.logosctl/logs/daemon.log`.

---

## Falling hours behind

`Online` does not prove the node is keeping up. Since 17 November mine has fallen behind for
hours at a time, then caught up by itself.

The pattern: the node rejects a valid block of the main chain with

```
Failed to process block: BatchDeferredProofVerification(InvalidZkSignatures)
```

puts it in its cache of rejected blocks, and then every block built on it fails with
`ParentMissing` or `orphan block (or its parent) is in the rejected cache`. The worst episode
kept it about 7 hours behind. On my node there were 0 to 7 such rejections a day, rising over
the week; about half of the rejected blocks were on the main chain.

I found no upstream issue for it as of 19 November, and the fix list of the coming 0.3.6
does not mention it. There is nothing to do on your side: the node recovered every time on
its own. I have not tried a restart for it. Check the lag with the command in
[vps.md, Everyday commands](vps.md#everyday-commands), and count the rejections with:

```bash
journalctl -u logos-node --since -1d | grep -c 'Failed to process block: BatchDeferredProofVerification'
```

---

## Epochs and your first block

**An epoch on this testnet is 300,000 slots of one second: 83 hours, about 3.5 days.** The
sources disagree:

| Source | Says |
|---|---|
| Release notes 0.3.4 | "Approximately 3.5h (two epochs) after you receive funds" |
| CLI guide | Tokens become eligible "after 3.5 hours" |
| Docs glossary | An epoch is "about 7.5 days long" |
| The testnet's own settings and the code | `security_param: 1000`, `slot_activation_coeff: 1/30`, epoch = 10 × 1000 × 30 = 300,000 slots |

The node agrees with the last one: `logos call blockchain_module get_time_info` gives the
current epoch and slot, and 300,000 slots per epoch fits.

**When your stake counts.** A token counts for block production once it has existed since the
start of the previous epoch. So tokens mined today count from the start of the epoch after
next: **3.5 to 7 days**. On my node, the tokens mined on 28 October counted from 2 November,
and its first proposal came on 5 November; most of the 2 billion, mined on 2 and 3 November,
counted from 9 November.

**How often you win.** About 10,000 blocks per epoch by design (one every 30 seconds; I
measured fewer, 60 to 110 an hour), shared by stake. The testnet started with about 25.2
trillion units of stake. With 2 billion, the expected count is about 0.8 blocks per epoch: one
every four or five days. Mine proposed six in two weeks, so luck swings it a lot.

**What a block pays.** Each proposed block gives a voucher that you claim with `leader_claim`.
`wallet_get_claimable_vouchers` shows a `reward_amount` of about 15,000 on my node; the docs
do not say what that field means, but if it is the reward per block, it is tiny next to
mining.

---

## Blocks that never make it

My node proposed six blocks between 5 and 19 November. **None of them is on the main
chain.** What I could see of each:

| Proposal | What happened |
|---|---|
| 5 and 10 November | Another node built a block on the same parent some slots later, and the network kept that one |
| 18 November | My node had just rejected a valid block ([above](#falling-hours-behind)) and built on a stale branch |
| 19 November | Another block on the same parent, one slot earlier, arrived 2 seconds before mine |
| 12 and 14 November | Not recorded |

Two upstream bugs may explain some of these, both fixed after 0.3.4:

- **Issue #3964:** nodes that are not Blend core nodes (most nodes are not; check with
  `curl -s 127.0.0.1:8080/blend/info`, `"core_info": null`) sent their proposals late, more
  so the longer the node had run and the more notes it held. Mining leaves about a thousand
  small notes. Fixed by PR #3986.
- **Issue #4119:** some proposals were dropped without ever being sent, but only in epochs
  where newly mined notes start to count. For a node that stopped mining, that is the first
  two epochs after. Fixed by PR #4121.

Both fixes are in **0.3.6**, which was being prepared on 19 November as an in-place update
from 0.3.4. Until then there is nothing to change: leave the node running.

To see your proposals: `journalctl -u logos-node --since -7d | grep 'proposed block'`.

---

## Disk writes

**The node writes about 1.4 TB a day to the disk, while its data stays around 2 GB.**

Measured on my server over five days: 16 MB/s on average, 8 MB/s in a quiet minute. On my
Windows PC under WSL it was 8.6 MB/s, about 0.75 TB a day. The database's own statistics show
why: the node rewrites values of several megabytes on the same keys about once per one-second
slot, and the database's housekeeping (compaction) writes even more on top. It is the node's
write pattern, not the machine.

On a rented server that is the provider's disk. On your own PC it is not: a typical 1 TB
consumer SSD is rated for about 600 TB written. At the rate I measured on my PC, a year uses
almost half of that; at my server's rate, most of it.
**That is why I moved my node off my PC after about a week,** and why the WSL guide is for a trial
run.

Space itself is not a problem: `/var/lib/logos-node` was 2.2 GB on 19 November. About half a
gigabyte of that is the database's text logs (`state/db/LOG*`). They grow about 30 MB a day,
a new file starts at every restart, and the database never deletes the old ones. They are
plain text: with the node stopped, `rm /var/lib/logos-node/state/db/LOG.old.*` frees the
space.

---

## Memory

The docs say 1 GB. What I measured:

- **On WSL, no limit:** about 0.8 GB in the first minute, 3.7 to 3.8 GB once synced, about
  4 GB after two hours, back to about 1 GB after a restart, then slowly up again.
- **On my server, held at 2.8 GB with a hard limit of 3 GB** (because it shares the machine,
  [below](#sharing-a-server-with-another-node)): about 1.1 GB resident plus 0.5 GB in swap,
  and never killed in two weeks.

An open upstream issue (#4113) measures memory growing during the first sync, to about 6 GB
for the whole chain; small machines were killed before reaching `Online`. That is why the
guide asks for 4 GB plus swap.

To check whether the system ever killed it, across all boots:

```bash
journalctl -k -b all | grep -i 'out of memory'
```

---

## Sharing a server with another node

Mine runs next to an Octra validator. Two changes made that work:

**1. Move the API off port 8080**, which the other node already used. Add this at the end of
`/var/lib/logos-node/user_config.yaml`, then restart:

```yaml
api:
  backend:
    listen_address: 127.0.0.1:8081
```

Then use `8081` instead of `8080` everywhere in this guide, including the `API=` line of the
start script in [vps.md Step 11](vps.md#step-11-run-it-under-systemd).

**2. Cap the node's memory** so the other node always wins. Add under `[Service]` in
`/etc/systemd/system/logos-node.service`, then `systemctl daemon-reload` and restart:

```ini
MemoryHigh=2800M
MemoryMax=3G
OOMScoreAdjust=500
```

Above `MemoryHigh` the kernel drops the node's file cache and moves some of its memory to
swap; `MemoryMax` is the hard limit. In two weeks mine sat at the 2.8 GB mark many times and
never reached the hard limit.

---

## Updates

`blockchain_module` updates are installed with `logosctl`, by version number and root hash
([vps.md Step 15](vps.md#step-15-keep-it-updated)). The root hash pins the exact package,
like a checksum. The official guide lists the current pair.

Things to know:

- **Your files carry over.** When I went from 0.3.2 to 0.3.4, the new version rewrote
  `user_config.yaml` in its new, shorter format (8.6 KB down to 805 bytes, only the settings
  that differ from the defaults) and kept the old one as `user_config.yaml.0.3.2.bak`.
  `keystore.yaml` was untouched.
- **A new genesis means an empty state.** 0.3.1 started a new chain; 0.3.2 and 0.3.4 continue
  it. When a release starts a new chain, stop the node, move `/var/lib/logos-node/state`
  aside, and start again. Your keys stay. The testnet can be reset at any time.
- **0.3.6** was in preparation on 19 November (draft PR #4173): an in-place update from
  0.3.4 with the same configuration and database, carrying the fixes for late and dropped
  proposals. It skips 0.3.5, which is a separate network for internal testing.
- **`logosctl` itself** has had no new release since 0.3.2 (6 October), so I have not
  updated it yet. The natural way is to repeat [vps.md Step 6](vps.md#step-6-install-logosctl)
  with the new version and its checksums, with the node stopped; I have not tested it.

---

## Moving a node to another machine

I moved mine from WSL to a server on 5 November, same keys and same chain data, with about a
quarter of an hour of downtime.

1. **Prepare the new machine** with [vps.md Steps 1 to 8](vps.md#step-1-rent-the-server),
   then `logos daemon stop`. **Skip Step 9**: it makes new keys. Check the versions match on
   both machines.
2. **Stop the old node, and check it is stopped** (`logos status` says the daemon is not
   running, and no `logos_host` process is left). Make sure nothing starts it again.
3. **Copy `keystore.yaml`, `user_config.yaml` and `state/`** into `/var/lib/logos-node` on
   the new machine. I piped them through `ssh` with `tar`, so no copy landed on a third disk:

   ```bash
   tar -C /var/lib/logos-node -cf - keystore.yaml user_config.yaml state | \
     ssh root@NEW_SERVER 'tar -C /var/lib/logos-node -xf -'
   ```

4. **On the new machine:** `chown -R logos:logos /var/lib/logos-node`, then
   `chmod 600` on the two YAML files, then [vps.md Step 11](vps.md#step-11-run-it-under-systemd),
   which starts it.

Back within 20 minutes, it skips the hour of bootstrapping. Mine was `Online` 92 seconds after
the start, with the same peer ID and balance. Copying `state/` is optional: without it the
new node simply syncs from scratch.

---

## Cold start crash on WSL

Once, after the PC was switched off without stopping the node, the first start crashed the
daemon. WSL's network was not ready, so the node could not reach its time server and hung in
`Starting`; every `logosctl call` then waited exactly 20 seconds and failed, and after the
third one the daemon died with a segmentation fault (`logosctl` 0.3.2).

Two habits avoid it: wait until the network works before starting the module
(`getent hosts pool.ntp.org` answers), and check the state through the HTTP API rather than
with repeated `logosctl call`. The start script in
[vps.md Step 11](vps.md#step-11-run-it-under-systemd) does both. On 0.3.4 my starts no longer
timed out, but the docs say a long replay can still make calls fail for a while, which is
why the script does not depend on the start call.

---

## SSH hardening does nothing

In SSH configuration, **the first setting wins**, not the last. Files in
`/etc/ssh/sshd_config.d/` are read in alphabetical order, so `50-something.conf` beats
`60-something.conf`.

Stock Ubuntu cloud images usually ship a `50-cloud-init.conf` containing
`PasswordAuthentication yes`. Put your hardening in a file numbered *above* 50 and it is
silently ignored: the config looks right and does nothing.

That is why [vps.md Step 3](vps.md#step-3-lock-down-ssh) writes `10-hardening.conf`, and
why it checks with `sshd -T` (which shows the **effective** result) instead of trusting the
file.

Still saying `yes`? Find what is winning:

```bash
grep -rn PasswordAuthentication /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

---

## Windows PowerShell quoting

PowerShell strips double quotes when passing arguments to a program like `ssh`, so nested
double quotes arrive on the server mangled, and you get errors that make no sense.

**Always: double quotes outside, single quotes inside.**

```powershell
ssh root@SERVER "curl -s 127.0.0.1:8080/cryptarchia/info | jq '.cryptarchia_info.state'"
```

The `logos` shortcut from [vps.md Step 7](vps.md#step-7-create-the-logos-user-and-the-logos-shortcut)
needs no quotes at all:

```powershell
ssh root@SERVER "logos status"
```

---

## Numbers I measured

If your machine is wildly different from these, something is off.

| | |
|---|---|
| `logosctl` download | 4 seconds, 106 MB |
| `package install blockchain_module` | 2 to 3 minutes, silent; an update took 77 seconds |
| First sync (October, 43,000 blocks) | Download 11 minutes, `Online` after 71 minutes |
| Catch-up after five days off | 16,700 blocks in 12 minutes, `Online` after 72 minutes |
| Restart after less than 20 minutes off | `Online` in 53 to 92 seconds |
| Reboot of the server to `Online` | 89 seconds |
| Peers | 140 within a minute behind a home router with nothing forwarded; 150 to 200 on a server |
| Memory | 3.7 to 4 GB once synced with no limit; 1.1 GB plus 0.5 GB swap when held at 2.8 GB |
| CPU, not mining | 0.3 to 0.45 of one core on average; up to about 1.5 cores for hours after some of its own proposals (cause unknown) |
| CPU, mining with the default one thread | 1.5 to 1.8 cores |
| Mining | One claim every minute or so, 1.87 million each; 2 billion in about 16 hours |
| Blocks on the chain | 60 to 110 per hour (about 85 in mid November); 120 by design |
| Epoch | 300,000 slots, about 3.5 days |
| First own proposal | 5 November, 8 days after the first tokens were mined |
| Own proposals that stayed on the chain | 0 of 6 (5 to 19 November, 0.3.4) |
| Disk space | 0.5 GB just synced (October), 2.2 GB in all on 19 November |
| Disk writes | 0.75 TB/day on my PC, 1.4 TB/day on my server |
| Node log | 10 hourly files kept, about 3 MB; daemon log about 85 MB |

---

## Where this guide differs from the official docs

The official server guide (<https://docs.logos.co/run-a-node>) installs all three modules
(blockchain, storage, messaging) and is a correct list of commands. Everything below is
something I added or changed, with the reason.

| # | What I do differently | Why |
|---|---|---|
| 1 | Run the blockchain module only | It is the part with a chain to follow and test tokens to mine. The other two need more open ports. |
| 2 | Open only `3000/udp`, and never 8080 | The other ports belong to the modules I skip. The API has no password. |
| 3 | Harden SSH, and verify with `sshd -T` | Ubuntu's default config silently overrides hardening put in the wrong file. |
| 4 | Check ADX and the system library before installing | The failure otherwise shows up at the very end, as a crash with `signal 4`. |
| 5 | Ask for 4 GB of RAM plus swap, not 1 GB | Mine used up to about 4 GB, and an upstream issue measures more during the first sync. |
| 6 | Add the `logos` shortcut command | Every command must run as the `logos` user from `/var/lib/logos-node`, or the chain data lands in the wrong folder. |
| 7 | Explicit key backup step before mining | The docs say to keep the files private, not to back them up. |
| 8 | Provide the start script and run under systemd from the start | The docs describe the script but do not provide one. Without it, the module is not started again after a restart. |
| 9 | Check the state with the HTTP API, not repeated `logosctl call` | Calls made while the node is busy time out, and on my first install a few in a row crashed the daemon. |
| 10 | Update with the node stopped | That is how I updated; the docs assume a running daemon and give no steps beyond 0.3.2 to 0.3.4. |
| 11 | Say that an epoch is about 3.5 days, so new stake counts after up to a week | The release notes say about 3.5 hours, the glossary 7.5 days. A node that proposes nothing for days looks broken when it is not. |
| 12 | Warn about the disk writes | About 1.4 TB a day. No official page mentions it, and it wears out a home SSD. |
| 13 | Explain the orphaned proposals and the hours of lag | Both happen to a healthy node on 0.3.4 and look like your fault. |
| 14 | Compare the newest block's slot with the clock | `Online` is reported by a node that is hours behind. |
| 15 | No faucet | It is not in the docs any more, an open issue (#4118) reports it failing, and mining is the intended way since testnet v0.3. |
| 16 | Show how to share a server with another node | Port 8080 is a common default, and a node with no memory limit can push the other one out. |

---

*Written from a real deployment. If something here is out of date, please open an issue.*
