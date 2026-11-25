<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/3d8e64e0-5803-4a82-a71f-aae04d7430ae" />

# FLOP LABS Testnet Validator Application Guide

## What This Guide Is

A walkthrough for running a **keyless RPC replica** node on FLOP Labs' Devnet. That is: a node without a wallet, without signing keys, that doesn't vote, but verifies and relays blocks. It's used to produce **sync & restart evidence** for the validator application process.

**Why you need it:**

- You've submitted (or plan to submit) a testnet validator application (`github.com/flop-labs/network/issues/new?template=operator-application.yml`).
- The form asks for "Sync and restart evidence": chain spec hash, finalized heights, peer ID, restart continuity, etc.
- To produce this evidence, you need a keyless RPC node running on Devnet.
- Once the evidence is produced, it is posted as a comment on your GitHub issue → the team reviews it → your validator promotion queue position strengthens.

**What this is not:**

- This is not a **validator** setup. Validator setup requires the signed operator kit, which is pending.
- This is not a **miner** setup. No GPU, no inference.
- This is not a **rewarded node**. Devnet does not pay out directly.

**Who it's for:**

- FLOP testnet or devnet validator candidates
- Operators comfortable with Linux + Docker via SSH
- People who can wait 6-12 hours for sync
- Someone who can keep a server up until evidence is produced (a cheap VPS is fine)

**Time expectations:**

- Setup: 20-40 minutes
- Peer discovery: 1-5 minutes
- Sync (from genesis to tip): **6-12 hours** (archive, \~57 GB)
- Observation commands: 5 minutes
- GitHub comment: 2 minutes

## Server and Provider Requirements

### Hardware (Minimum)

| Component | Minimum | Recommended |
| --- | --- | --- |
| CPU | 4 vCPU | 8 vCPU |
| RAM | 16 GB | 32 GB |
| Disk | 100 GB SSD | 300 GB NVMe (archive grows) |
| Network | 1 Gbps | 1 Gbps |
| OS | Ubuntu 22.04 or 24.04 | Ubuntu 24.04 |
| GPU | Not needed | Not needed |

### Critical: **The NAT Trap**

Your server **must have a public IP directly assigned to its network interface**. On NAT'ed VPSes (QEMU SLIRP, "shared IP" style plans), the libp2p peer handshake never completes and the node stays stuck at 0 peers.

Before buying — or as soon as you buy — verify:

```sh
ip -4 addr show
PUB_IP=$(curl -s https://ipinfo.io/ip)
echo "Public IP: $PUB_IP"
ip -4 addr show | grep -q "$PUB_IP" && echo "✅ DIRECT PUBLIC IP (proceed)" || echo "❌ NAT (this VPS will not work)"
```

If you see `❌ NAT`, change the VPS. To avoid wasted money, ask the provider for "direct public IP on the interface" before purchasing.

### Use Caution With Hetzner

There have been issues with crypto nodes on Hetzner in the past — in 2022 they shut down Solana validators en masse. Their current policy may have changed; if you want to use them, check the provider's current terms of service. The same diligence applies to the other major cloud providers (AWS, GCP, Azure, DigitalOcean).

### Watch Out for NAT'ed Plans

Even Contabo has some older "VPS SSD" plans that come NAT'ed (QEMU SLIRP). Make sure to pick a **Cloud VPS** or **Dedicated Server**. If you see IPs like `10.0.2.15` or `192.168.x.x` on your interface, it's NAT.

## Step 1: Prepare the Server

SSH into your server as root. Install Docker + Compose + helper tools:

```sh
# Skip if Docker is already there
docker --version && docker compose version || {
  curl -fsSL https://get.docker.com | sh
  systemctl enable --now docker
}

# Extra packages
apt update
apt install -y jq curl openssl ufw

# Working directory
mkdir -p ~/flop-devnet-rpc && cd ~/flop-devnet-rpc
```

After the last command you should be in `/root/flop-devnet-rpc` (verify with `pwd`).

## Step 2: Download and Verify the Package

The FLOP team publishes setup files in a pinned gist revision:

```sh
cd ~/flop-devnet-rpc

files=https://gist.githubusercontent.com/sv/7c8b93f3868bc15e3b6eabe0578057e3/raw/21adaa99c601003278d1682bf937696f21d31432
for file in docker-compose.rpc.yml node-entrypoint.sh rpc_ready.py rpc-pool.env.example rpc-origin.Caddyfile; do
  curl --fail --location --output "$file" "$files/$file"
done
```

**SHA-256 verification** (security critical):

```sh
sha256sum --check <<'SUMS'
6487cd84d5a257659297ad7b8fdd55978e6d8327bfe029d9460645da2591345d  docker-compose.rpc.yml
7279e81c0d6de4b6c59733b64132f929b8b7dd5825b8f37433d09940a93f983d  node-entrypoint.sh
6a066b7540628c7db4f455d7bfd2540db54355a4c0a2959d97032b76bcf641cd  rpc_ready.py
4f7295735f75b5a7b4c0babe3be4b0eef687decab99ba1612f91ee6f3b04d1bb  rpc-pool.env.example
e184d4f38f054dbe7947efde4d85e6a8964f02e594666ad42b9216dd98a4b2d5  rpc-origin.Caddyfile
SUMS
```

You should see 5 lines of `OK`. If any line says `FAILED`, stop and re-download.

## Step 3: Download the Chain Spec

The Devnet chain spec lives on the official explorer:

```sh
cd ~/flop-devnet-rpc

curl --fail --location --output chain.raw.json \
  https://explorer.devnet.flop.network/validator-downloads/flop_devnet_ts-99c172c095340ace.raw.json

printf '%s  %s\n' \
  3b213c38e7678285bc38fbdfded5dbe3d6acbd16e836b14aaaeef2f6ae1cd16e \
  chain.raw.json | sha256sum --check
```

You should see `OK`.

**Genesis hash verification** — confirm your chain spec is on the same chain as live Devnet:

```sh
curl -s https://rpc.devnet.flop.network -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"chain_getBlockHash","params":[0],"id":1}' | jq -r .result
```

Expected: `0x99c172c095340ace980631f9123544c6bebd4839f3437e463533422d8be2d631`. This **must match exactly**. If it does not, the chain has been reset and you need a new spec.

## Step 4: Fill the Env File

Copy from the example and fill in the public IP and bind IP:

```sh
cd ~/flop-devnet-rpc
cp -n rpc-pool.env.example rpc-pool.env

# Auto-detect public IP
PUB_IP=$(curl -s https://ipinfo.io/ip)
echo "Public IP: $PUB_IP"

# HOST_IP=0.0.0.0 (listen on all interfaces), PUBLIC_ADDR=/ip4/PUB_IP/tcp/30333
sed -i "s|^HOST_IP=.*|HOST_IP=0.0.0.0|" rpc-pool.env
sed -i "s|^PUBLIC_ADDR=.*|PUBLIC_ADDR=/ip4/$PUB_IP/tcp/30333|" rpc-pool.env

# Verify
grep -E '^(HOST_IP|PUBLIC_ADDR|CHAIN|FLOP_NODE_IMAGE)=' rpc-pool.env
```

**Notes:**

- `HOST_IP=0.0.0.0` — Docker will bind the port on all interfaces. Required on NAT'ed providers, safe on direct-IP hosts too.
- `PUBLIC_ADDR=/ip4/<public_ip>/tcp/30333` — other peers will use this address to reach you.
- **Use an IP, not a hostname, in `PUBLIC_ADDR`.** DNS can cause issues.
- Keep `FLOP_NODE_IMAGE` on the pinned sha; don't switch to `latest`.

## Step 5: Expand BOOTNODES (Critical)

This is the most important step. The default env ships with **only one** bootnode, and it alone struggles to accept new keyless RPC peers. You'll get stuck at 0 peers.

The fix: add other public Devnet nodes to the BOOTNODES list. Several operators have publicly published their P2P addresses; you can collect and add them.

```sh
cd ~/flop-devnet-rpc

# Official bootnode + 3 additional public peers
sed -i 's|^BOOTNODES=.*|BOOTNODES=/ip4/139.180.223.117/tcp/30333/p2p/12D3KooWACzHKN2YkLtSwjTYXYtMHcVNWnJhG3hRdUkqa5DzBu9X /ip4/136.244.77.30/tcp/30333/p2p/12D3KooWJczvBNuzXNkD4AuiD4dwrKqVwXYzK2bYeDe4vBMLM2yT /ip4/43.143.199.81/tcp/30333/p2p/12D3KooWMhfWgpxzRo5XSb3nfe7yjJok8NxruqF1u1CqFGHaCqjb /ip4/65.109.36.231/tcp/30333/p2p/12D3KooWJZBres7eDNVSLH9HPxd1mLUpMiqHiBRAvDyxiAe98DHs|' rpc-pool.env

grep ^BOOTNODES rpc-pool.env
```

**Important rules:**

- Every address must follow the format `/ip4/<IP>/tcp/<port>/p2p/<peer_id>`. **Addresses without a peer ID** are rejected by substrate and the node will refuse to start.
- Addresses are separated by **spaces** (not commas).
- These entries may go stale over time — some operators retire their nodes. As long as at least 2-3 of them are reachable, you're fine.

**Finding fresh public peers:**

You can collect current peer addresses by browsing other operator applications on GitHub: `github.com/flop-labs/network/issues?q=is%3Aissue+operator+devnet+validator` → open recent issues → copy addresses from the `Advertised public P2P multiaddress` field.

## Step 6: Firewall and Port 30333

The P2P port (30333/TCP) must both be listening and reachable from outside.

**If UFW is active:**

```sh
ufw status
# If active:
ufw allow 22/tcp    # don't lock yourself out of SSH
ufw allow 30333/tcp
ufw status
```

**Provider firewall panel:**

Some providers (Contabo, Vultr, OVH) have an additional firewall at the panel level. Allow 30333/TCP inbound there as well.

**Indirect test:**

After setup, open `https://portchecker.co` and test `<public_ip>:30333`. It should say `Open`.

## Step 7: Start With Compose and Check Peers

Validate the config + first run:

```sh
cd ~/flop-devnet-rpc
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml config --quiet && echo "config OK"

docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml up -d

sleep 20
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml ps
```

`ps` should show **both containers as `Up`**:

- `flop-devnet-rpc-pool-rpc-1`
- `flop-devnet-rpc-pool-ready-1`

If the rpc container is in `Restarting` or `Created` state, there's a problem:

```sh
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml logs --tail=30 rpc
```

The logs will show the issue (usually an env file or BOOTNODES error).

**Peer check (after 60-90 seconds):**

```sh
sleep 90
curl -s http://127.0.0.1:9619/ready | jq
```

Expected output:

```json
{
  "rpc_ok": true,
  "best": <growing number>,
  "finalized": <growing number>,
  "highest": 4627000+,
  "peers": 3+,
  "ready": false,
  "reasons": ["sync lag ..."]
}
```

**Key fields:**

- `peers` must be **1+** — if still 0 after 2 minutes, revisit BOOTNODES
- `best` should keep growing with every poll
- `ready: false` + `reasons: ["sync lag ..."]` is NORMAL — means sync is in progress
- You may see `(unhealthy)` on Docker's status — don't worry, it'll turn `healthy` when sync completes

## Step 8: Wait for Sync to Complete

Archive sync can take **6-12 hours**. Devnet is \~57 GB and has 4.6M+ blocks. The actual time depends on:

- Server speed
- Peer count and quality (more peers = faster)
- Network bandwidth

**Periodic check:**

```sh
curl -s http://127.0.0.1:9619/ready | jq '{best, finalized, highest, peers, best_frozen_s, reasons}'
```

**Expectations:**

- `peers` steady or growing (8+ ideal)
- `best` constantly increasing
- `highest - best` shrinking
- `ready` eventually becomes `true`, `reasons` empties

**Rough ETA:**

```sh
# How many blocks arrived between two measurements
FIRST=$(curl -s http://127.0.0.1:9619/ready | jq .best)
sleep 300
SECOND=$(curl -s http://127.0.0.1:9619/ready | jq .best)
echo "Blocks in 5 min: $((SECOND - FIRST))"
echo "Approx remaining: $(curl -s http://127.0.0.1:9619/ready | jq '(.highest - .best) / 1000')K blocks"
```

**Things not to do while syncing:**

- `docker compose down -v` — deletes the volume and chain data
- Reboot the server (sync pauses, resumes)
- Change the image
- Edit `rpc-pool.env` (requires restart)

**Safe operations:**

- `docker compose logs -f rpc` — tail logs
- `docker compose ps` — status check
- Ready endpoint queries

**When sync is done:**

```json
{
  "rpc_ok": true,
  "best": 4700000+,
  "finalized": 4700000+,
  "highest": 4700000+,
  "peers": 8+,
  "ready": true,
  "reasons": []
}
```

Once you see `ready: true`, move on to evidence.

## Step 9: Observation and Restart Evidence

Once sync is complete (ready: true), you're ready to produce evidence. These commands come straight from Section 2 of the official gist.

**Helper function + constants:**

```sh
cd ~/flop-devnet-rpc

rpc() {
  curl --fail --silent --show-error --max-time 15 "$1" \
    -H 'Content-Type: application/json' \
    --data "$(jq -cn --arg method "$2" --argjson params "$3" \
      '{jsonrpc:"2.0",id:1,method:$method,params:$params}')" | jq -er '.result'
}

LOCAL=http://127.0.0.1:9944/
FLOP=https://rpc.devnet.flop.network/
GENESIS=0x99c172c095340ace980631f9123544c6bebd4839f3437e463533422d8be2d631
```

**Read finalized height:**

```sh
finalized_height() {
  local head params hex
  head=$(rpc "$1" chain_getFinalizedHead '[]')
  params=$(jq -cn --arg head "$head" '[$head]')
  hex=$(rpc "$1" chain_getHeader "$params" | jq -er .number)
  printf '%d\n' "$hex"
}
```

**Single observation function:**

```sh
observe() {
  local local_height reference_height common_height local_hash reference_hash
  test "$(rpc "$LOCAL" chain_getBlockHash '[0]')" = "$GENESIS"
  test "$(rpc "$FLOP" chain_getBlockHash '[0]')" = "$GENESIS"
  curl --fail --silent --show-error --max-time 5 http://127.0.0.1:9619/ready >/dev/null
  local_height=$(finalized_height "$LOCAL")
  reference_height=$(finalized_height "$FLOP")
  common_height=$local_height
  if (( reference_height < common_height )); then common_height=$reference_height; fi
  local_hash=$(rpc "$LOCAL" chain_getBlockHash "[$common_height]")
  reference_hash=$(rpc "$FLOP" chain_getBlockHash "[$common_height]")
  test "$local_hash" = "$reference_hash"
  printf '\nUTC=%s local_finalized=%s reference_finalized=%s common_height=%s hash=%s peer=%s\n' \
    "$(date -u +%FT%TZ)" "$local_height" "$reference_height" "$common_height" \
    "$local_hash" "$(rpc "$LOCAL" system_localPeerId '[]')"
}
```

**3 observations + restart:**

```sh
# Save peer ID (must be unchanged after restart)
peer_before=$(rpc "$LOCAL" system_localPeerId '[]')
echo "Peer ID: $peer_before"

# Observation 1
observe

# Wait 60 seconds
sleep 60

# Observation 2
observe

# Restart
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml restart rpc ready

# Wait for readiness (1-3 minutes)
while ! curl -sf http://127.0.0.1:9619/ready | jq -e '.ready == true' >/dev/null 2>&1; do
  echo "Waiting for readiness..."
  sleep 10
done

# Observation 3 (after restart)
observe

# Peer ID unchanged?
test "$(rpc "$LOCAL" system_localPeerId '[]')" = "$peer_before" && echo "✅ Peer ID unchanged" || echo "❌ Peer ID changed"
```

**Save the three observations** — you'll paste them into the GitHub issue.

Five critical conditions:

1. **Local and reference genesis hashes must match** (the test command verifies)
2. **Finalized heights must advance** across observations 1 → 2 → 3
3. **Common-height hashes must match** between LOCAL and FLOP (both signed the same block)
4. **Finality must keep advancing after restart**
5. **Peer ID must not change** (means volume and node key were preserved)

If any of these fail, the evidence isn't valid — you'll need to fix the issue first.

## Step 10: Open the Operator Application Form (If You Haven't Yet)

If you haven't yet opened a validator candidate application, do that first. If you've completed the Devnet keyless RPC sync, you can fill in the evidence straight into the first application.

### Two Strategies: When to Apply

**A) Apply first, add evidence later (for urgency)**

- Open the application before the server is even ready, select `planning a host`
- Secure your spot in the first-50 seat queue right away
- Add observations as a comment once sync finishes (Step 11)
- **Pro:** earlier application = earlier promotion
- **Con:** evidence field is empty at first, needs an update later

**B) Sync first, then full application (cleaner)**

- Complete Steps 1-9, have the node synced and observations ready
- Open the application with evidence already filled in
- Select `sync and restart evidence ready` as the preparation stage
- **Pro:** single clean submission, immediately reviewable
- **Con:** sync takes 6-12 hours; during that time others may move ahead in queue

**Which one:** If you're not in a rush, **B** is tidier. If queue position is critical (e.g. first-50 seats), **A** is safer.

### Form URL

```
https://github.com/flop-labs/network/issues/new?template=operator-application.yml&network=devnet
```

Sign in with your GitHub account. The template loads automatically. Title should say "Validator / RPC operator application".

### Fields

| Field | What to enter for Devnet |
| --- | --- |
| **Title** | `operator: <team-id> devnet validator candidate` |
| **Team ID** | Your public team name (e.g. `NakoTurk`). Reuse the same name everywhere. |
| **Network** | `devnet` (dropdown) |
| **Requested role** | `validator candidate` (dropdown). For RPC pool too, pick `both` |
| **Existing application or introduction** | Leave empty if no prior issue |
| **Planned capacity and host** | Fill the template below |
| **Preparation stage** | If sync is done: `sync and restart evidence ready`. If not: `planning a host` |
| **Sync and restart evidence** | If sync done: observation outputs from Step 9 (use Step 11's template). If not: `"Will update within 24 hours after keyless RPC sync completes"` |
| **Questions or blockers** | Write blockers if any, otherwise `none` |
| **Before submitting** | 3 checkboxes — tick all |

### Planned Capacity and Host (Template)

Paste this block and replace with your own values:

```
Requested validator seats / existing FLOP seats: 1 requested / 0 existing
RPC nodes offered (active / standby): 1 active keyless RPC replica / 0 standby
Provider / region: <Contabo Cloud VPS, Germany (Nurnberg)>
CPU / RAM / SSD capacity / OS: <8 vCPU / 32 GB / 300 GB NVMe / Ubuntu 24.04>
Public host label or P2P address (if prepared): <team-name-rpc-01>, /ip4/<public_ip>/tcp/30333/p2p/<peer_id>
```

### Submit

1. Fill in all fields
2. Tick all 3 checkboxes
3. Click "Create"
4. Save the issue URL (e.g. `github.com/flop-labs/network/issues/XXXX`)
5. "Watch" the issue so you get notified of team replies

### Once Sync Is Done

If you opened the application while sync was still running, come back once it finishes and apply Step 11 — you'll add observation outputs as a comment.

### Watch Out

- **Never** paste a seed phrase, keystore password, session token, or `.env` contents into the form
- `Team ID` is hard to change later; pick carefully
- **One issue per team per network** — don't open a second one, update the existing one

## Step 11: Add Evidence to the GitHub Issue

Once your observations pass, add them as a comment on your validator application issue. Template:

```markdown
### Sync and restart evidence — <date>

**Host and image**
- Public host label: `<your label>`
- Advertised P2P: `/ip4/<public_ip>/tcp/30333/p2p/<your_peer_id>`
- Node image digest: `ghcr.io/flop-labs/flop-core/node-devnet@sha256:84498fd73db2464eb13088ab34d369ddf7cddd22a45c810d140c9ff8ec16c91d`
- Guide revision: `gist sv/7c8b93f3868bc15e3b6eabe0578057e3 @ 21adaa99c601003278d1682bf937696f21d31432`

**Chain identity**
- Chain spec SHA-256: `3b213c38e7678285bc38fbdfded5dbe3d6acbd16e836b14aaaeef2f6ae1cd16e`
- Expected genesis: `0x99c172c095340ace980631f9123544c6bebd4839f3437e463533422d8be2d631`
- Local and reference genesis match: verified

**Observation 1**
UTC=<time> local_finalized=<height> reference_finalized=<height> common_height=<height> hash=<hash> peer=<peer_id>

**Observation 2** (after ~60s)
UTC=<time> local_finalized=<height> reference_finalized=<height> common_height=<height> hash=<hash> peer=<peer_id>

**Restart**
- `docker compose restart rpc ready` executed; readiness returned to true
- Same Compose project and volume retained
- No Tailscale on this host

**Observation 3** (after restart)
UTC=<time> local_finalized=<height> reference_finalized=<height> common_height=<height> hash=<hash> peer=<peer_id>

**Continuity checks**
- Finality advancing: observations 1 → 2 → 3 heights strictly increasing
- Common height hashes match between local and reference at each observation
- Peer ID unchanged after restart: `<peer_id>`
```

**Important:**

- Replace every `<...>` with real values from your observation outputs
- Seeds, keystore passwords, session tokens — **never** post these. Public IP and peer ID are already public.
- Post it as a comment at the bottom of the issue, don't edit the description (others should see it)
- Click `Comment`, then "Watch" for team replies

## Troubleshooting and Common Errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| `cannot assign requested address` | VPS is NAT'ed | Change the VPS (need direct IP) |
| rpc container `Restarting` + "Peer id is missing" | BOOTNODES contains an address without a peer ID | Keep only entries in full `/p2p/12D3KooW...` form |
| `rpc_ok: false` + URLError | rpc container not up, so ready can't reach it | Check with `docker compose logs rpc` |
| `peers: 0` after 5 min+ | Pinned bootnode rejecting handshake | Expand BOOTNODES with additional public peers (Step 5) |
| `peers: 0` + logs saying `Failed to connect using existing socket` | Same (handshake rejected) | Expand BOOTNODES |
| `(unhealthy)` on `ps` | Sync is running, healthcheck trips on lag | Normal — ignore. Goes `healthy` when sync completes |
| portchecker `Closed` | Provider firewall blocks 30333 | Allow inbound at the panel or check UFW |
| `genesis hash mismatch` | Chain was reset | Download a fresh spec |
| `no space left on device` | Disk full | Migrate to a larger VPS or `docker compose down` + delete |
| `Discovered new external address` shows wrong IP | `PUBLIC_ADDR` is wrong | Fix the env, restart |

**Enable debug logs (for diagnostics):**

```sh
cd ~/flop-devnet-rpc
if grep -q "^NODE_LOG=" rpc-pool.env; then
  sed -i "s|^NODE_LOG=.*|NODE_LOG=sync=debug,sub-libp2p=debug,libp2p_tcp=debug|" rpc-pool.env
else
  echo "NODE_LOG=sync=debug,sub-libp2p=debug,libp2p_tcp=debug" >> rpc-pool.env
fi
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml down
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml up -d
sleep 45
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml logs --tail=100 rpc 2>&1 | grep -iE "(discover|dial|peer|connect|error|fail)" | tail -30
```

Once you've found the issue, turn debug off (saves disk + CPU):

```sh
sed -i "/^NODE_LOG=/d" rpc-pool.env
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml down
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml up -d
```

## Daily Operations and Next Steps

### Keeping the Node Running

It's useful to keep the node up for a while after sync completes:

- Reachable while the team reviews your evidence
- Ready if they ask for additional proof
- You can reuse the same server once the Testnet operator kit is published (but Testnet will need **a new package, new spec, and new data dir**)

**Everyday commands:**

```sh
cd ~/flop-devnet-rpc
C="docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml"

# Status
$C ps
$C logs --tail=50 rpc

# Ready endpoint
curl -s http://127.0.0.1:9619/ready | jq

# Quick peer-count check
watch -n 10 'curl -s http://127.0.0.1:9619/ready | jq ".peers"'

# Restart (if needed)
$C restart rpc ready

# Stop (volume preserved)
$C down

# Nuke completely (DANGEROUS, deletes chain data)
$C down -v
```

### Disk Usage Tracking

Archive grows continuously. Check weekly:

```sh
df -h /
du -sh /var/lib/docker/volumes/flop-devnet-rpc-pool_rpc-data
```

Watch when it drops below 50 GB free — ask the provider to expand disk before it fills.

### Node Key Backup and Restore

The node key (`secret_ed25519`) determines your server's peer ID. If this file is lost or changes, the peer ID also changes, which invalidates your restart continuity evidence. Always back it up in case you lose the server.

**Backup:**

```sh
docker compose stop rpc
cp ~/flop-devnet-rpc/data/chains/flop_devnet_ts/network/secret_ed25519 ~/node-key-backup
chmod 600 ~/node-key-backup
docker compose start rpc
```

Download it to your local machine (run this from your own computer, outside SSH):

```sh
scp root@<server-ip>:~/node-key-backup ~/flop-node-key-backup
```

Store the file somewhere encrypted (password manager, encrypted USB). This file is the private key for your peer ID — **never share it, never paste it into a repo or GitHub issue.**

**Restore (on a new server):**

Run Steps 1-6 normally, then **before Step 7 (compose up)** place the node key:

```sh
mkdir -p ~/flop-devnet-rpc/data/chains/flop_devnet_ts/network/
cp ~/node-key-backup ~/flop-devnet-rpc/data/chains/flop_devnet_ts/network/secret_ed25519
chmod 600 ~/flop-devnet-rpc/data/chains/flop_devnet_ts/network/secret_ed25519
```

Then start with `docker compose up -d`. Verify the peer ID is unchanged:

```sh
docker compose logs rpc 2>&1 | grep "Local node identity"
```

The `12D3KooW...` value in the output must exactly match your previous peer ID. If it differs, the node key wasn't copied to the right place.

### Shutting the Node Down

If you're moving to a different server, shut the old node down cleanly:

```sh
cd ~/flop-devnet-rpc
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml down

# To keep data (so you can restart later): do nothing, volume persists

# To wipe everything:
docker compose --env-file rpc-pool.env -f docker-compose.rpc.yml down -v
```

### Moving to Testnet

Once the FLOP team publishes the **Testnet operator kit** (likely this week or next):

1. You can stop this Devnet node (evidence is already submitted)
2. The new package will be for Testnet, with a different chain spec and different bootnodes
3. You can run a Testnet node **in a separate directory on the same server** (e.g. `~/flop-testnet-rpc/`)
4. Peer ID separate, keystore separate — no mixing
5. Post Testnet keyless RPC evidence as a comment on your application issue

### Next Milestones

Validator promotion steps:

1. Devnet keyless RPC (this guide) → evidence produced, added to application ✅
2. Testnet keyless RPC (once operator kit ships) → same flow on Testnet
3. **Signed validator kit** (once the FLOP team invites you) → validator key, registration, stake
4. Testnet validator activation → enter the ranking within the 14-day grace period

### Official Resources

- **Devnet RPC guide:** `https://gist.github.com/sv/7c8b93f3868bc15e3b6eabe0578057e3`
- **Public Devnet RPC endpoint:** `https://rpc.devnet.flop.network`
- **Testnet status:** `https://docs.flop.network/networks/testnet`
- **Validator prep:** `https://docs.flop.network/testnet-validator-prep/`
- **Operator application form:** `https://github.com/flop-labs/network/issues/new?template=operator-application.yml`
- **Official verification:** `https://flop.finance/official/`
- **Official bot:** `https://ask.flop.finance`
- **GitHub repo:** `https://github.com/flop-labs/network`
