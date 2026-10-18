# TRU Blockchain  ·  Docker Node Deployment Guide

**TRU Core 0.08.5 · Mainnet · Public-node guide · Revision 2 · 18 October 2026**

Run a TRU full node on Linux with Docker, without compiling C++ source. The default installation supports CPU or NVIDIA-compatible images, outbound blockchain synchronization, optional incoming P2P connections, encrypted wallet storage, and local authenticated RPC. It also preserves the **PQ01/WOTS+ quantum-wallet sidecar**, should you use the quantum functions.

> **Important:** This guide was checked against the provided `TRU-Core` 0.08.5 source and the previous Docker README. The **actual GHCR image tags, binary paths, and entrypoint were not independently verified against a pulled image**. Check them as described in Step 2 before following the remaining commands. This guide is for **Linux Docker Engine**; Windows/macOS Docker Desktop networking differs.

> **Activation:** TRU 0.08.5 has a configured mainnet consensus activation at **block 50,000** in the supplied node output. Confirm the running node with `getconsensus09info`; do not treat the installed version alone as proof of activation. Do **not** submit PQ01-protected mainnet spends before activation.

---

## Contents

1. What this installation does
2. Prerequisites, image choice, and verification
3. Host folders and data ownership
4. Create a safe `tru.conf`
5. Bootstrap a new encrypted wallet (once only)
6. Back up the wallet before starting
7. Run a CPU or GPU node (Docker bridge mode)
8. Keep the node running and manage its lifecycle
9. Verify synchronization, peers, consensus, and chain audit
10. Ports, NAT, CGNAT, Starlink, and public peer visibility
11. Local authenticated RPC and host-based miners
12. Optional local block explorer
13. Quantum wallet state and backup precautions
14. Updating 0.08.5 or later versions safely
15. Migrating from the old three-file Docker deployment
16. Restore/recovery guidance
17. Troubleshooting
18. Security checklist
19. Reference commands and safe uninstall

---

## 1. What this installation does

A regular node validates and synchronizes the TRU chain and communicates with other TRU P2P peers. You do **not** need an oracle identity, AI provider, mining hardware, marketplace credentials, or token-evolution configuration to run a normal node.

| Component | Port / location | Access | Persistent? |
|---|---|---|---|
| TRU P2P | TCP `21833` | Public **if** operator and network permit | Peers recorded with chain data |
| TRU authenticated JSON-RPC | TCP `21832` | **Loopback inside container** by default | Authentication cookie may be regenerated when container is recreated |
| Optional block explorer | TCP `8001` | Optional; publish **host-loopback only** | Derived from node |
| Wallet | `~/tru/wallet/` → `/wallet` | Private operator files | **Yes — irreplaceable** |
| Blockchain database/logs | Docker volume `tru-data` → `/app/data` | Private node data | **Yes — can generally be re-synced** |
| Config | `~/tru/tru.conf` → `/app/tru.conf` | Private local file | **Yes** |

The wallet passphrase is required for encrypted-wallet startup. **This interactive guide does not configure unattended unlocking or automatic boot-time startup.**

### Wallet files: three initially, more later

A newly bootstrapped encrypted wallet produces:

```text
wallet_seed.dat.enc
tru.dat.enc
tru.dat.public
```

Quantum `receive` adds **`tru.dat.pq01`**. Wallet journals and other companion files may also be produced by later operations. **Back up the entire wallet directory**, not only the original three files. Losing the passphrase or the required current wallet files may make funds unrecoverable. Restoring stale PQ01 state risks reuse of a WOTS+ one-time key.

---

## 2. Prerequisites, image choice, and verification

You need Linux with a working Docker Engine, network access, space for chain growth, and an interactive terminal for the encrypted-wallet passphrase. GPU mode additionally needs the NVIDIA driver and NVIDIA Container Toolkit.

```bash
# Check Docker access
docker --version
docker info >/dev/null && echo 'DOCKER_ACCESS=PASS'

# GPU operators only
nvidia-smi
```

Choose **one** image. These are the tag names from the supplied original README; confirm that the publisher has made the tags public:

**CPU:**

```bash
export TRU_IMAGE='ghcr.io/tokenizedrealutility/tru-node:v0.08.5'
```

**NVIDIA GPU:**

```bash
export TRU_IMAGE='ghcr.io/tokenizedrealutility/tru-node-gpu:v0.08.5'
```

Pull and inspect the image:

```bash
docker pull "$TRU_IMAGE"
docker image inspect "$TRU_IMAGE" \
  --format 'Entrypoint={{json .Config.Entrypoint}} Cmd={{json .Config.Cmd}} WorkDir={{.Config.WorkingDir}}'
```

**Required preflight check:** This revised guide runs Core with `--workdir /wallet`, so that its relative wallet name `tru.dat` and its quantum sidecar `tru.dat.pq01` live together on a persistent host bind mount. The image's **entrypoint must invoke the Core by an absolute executable path** (for example `/app/<actual-core-binary>`), independent of working directory. If it uses a relative `./binary` entrypoint, or no entrypoint, **STOP** and adjust the invocation for the actual image before continuing. Do not assume an executable name. The original README established `/app/tru-cli` as the client path; you can verify that with the image publisher.

Use a numbered release tag, not `latest`; for exact reproducibility in production, the operator may additionally pin a verified `sha256` digest. If `docker pull` fails, check the **publisher and published tag** instead of silently swapping to another version.

> Environment variables such as `TRU_IMAGE` exist only in the current shell. Export them again in a new SSH session.

---

## 3. Host folders and data ownership

```bash
umask 077
mkdir -p "$HOME/tru/wallet"
chmod 700 "$HOME/tru/wallet"
mkdir -p "$HOME/tru"
```

Expected layout:

```text
~/tru/
├── tru.conf             # node config
└── wallet/              # complete encrypted wallet and PQ01 state

Docker named volume: tru-data    # chain, peer database, node logs
```

The example uses the container image's default user, which may be `root`. Files created on the host bind mount may then be owned by root. Use `sudo` for **local backup/read-only inspection** when required, or adapt the container image to a tested non-root user/volume-ownership scheme. Do not weaken wallet permissions to `777`.

---

## 4. Create a safe `tru.conf`

**New installation only:** This command **overwrites** `~/tru/tru.conf`. If the machine already has a working configuration, copy it to a safe backup and edit it instead of running this overwrite command.

```bash
# Only for first-time installs without an existing config
test ! -e "$HOME/tru/tru.conf" || { echo 'Config exists: edit it; do not overwrite.'; false; }
```

If the test passes, create the file:

```bash
cat > "$HOME/tru/tru.conf" <<'CONF'
# ============================================================
# TRU CORE 0.08.5 — MAINNET PUBLIC NODE
# ============================================================

[network]
# Local authenticated JSON-RPC; not a public API
rpcport=21832
rpcbind=127.0.0.1
rpcAllowRemote=0
rpcMaxConnections=32

# Peer-to-peer blockchain traffic
p2pPort=21833
listen=1
addnode=137.184.68.43:21833

# Enable multi-node cumulative-work automatic reorganization.
# This affects chain behavior and is not a casual UI setting.
automaticReorg=1

# RPC spending is disabled for this public-node example.
# Enable ONLY on a trusted local operator node, after review.
TRU_RPC_WALLET_SEND_ENABLE=0
# TRU_RPC_WALLET_SEND_MAX_ATOMS=100000000

# Usually leave unset. Never use a LAN address like 192.168.x.x.
# externalip=PUBLIC_IPV4_IF_EXPLICITLY_NEEDED

[default]
# Local legacy miner/oracle client default
node.ip=127.0.0.1
node.port=21832

# Optional oracle settings (not needed for normal synchronization)
# oracle.address=YOUR_TRU_ADDRESS
# oracle.wallet=tru.dat

[agent_checkpoint]
enabled=0
# policy_file=/path/to/private/policy.json

[chain]
chainName=TRUMain
networkType=mainnet
CONF

chmod 600 "$HOME/tru/tru.conf"
cat "$HOME/tru/tru.conf"
```

`[chain]` values are retained from the prior deployment format. **Do not assume they switch the compiled binary to regtest/testnet without verifying that implementation.** Confirm the live network through the node's chain identity and consensus output.

### Network settings explained

| Setting | Purpose |
|---|---|
| `rpcport=21832` | Local JSON-RPC listening port |
| `rpcbind=127.0.0.1` | **Inside-container** loopback binding in bridge mode |
| `rpcAllowRemote=0` | Refuses non-loopback RPC binding |
| `rpcMaxConnections=32` | Maximum configured RPC connections |
| `p2pPort=21833` | TRU P2P service port |
| `listen=1` | Allow incoming P2P when Docker/network permit |
| `addnode=137.184.68.43:21833` | Initial bootstrap peer |
| `automaticReorg=1` | Operator opt-in to production automatic cumulative-work reorg; keep backups and run consensus tests |
| `TRU_RPC_WALLET_SEND_ENABLE=0` | Disable money-moving Core RPC operations by default; required `1` for applicable quantum wallet RPCs |
| `externalip` | Optional address used by current P2P logic for peer/self filtering; **does not bypass NAT/CGNAT** |

> **Do not put `externalip` at the end of the file using `echo >> tru.conf`:** that would place it under `[chain]` instead of `[network]`. In most home-node configurations, leave it unset entirely.

---

## 5. Bootstrap a new encrypted wallet — ONCE ONLY

**If this is an upgrade or an existing wallet, SKIP this whole step and go to Step 6/7 or the migration section.** Bootstrap creates an entirely **new identity**. Never bootstrap to replace a wallet you want to keep.

Check for existing wallet artifacts:

```bash
sudo ls -la "$HOME/tru/wallet"
docker ps -a --filter 'name=^/tru-bootstrap$'
docker ps -a --filter 'name=^/tru$'
```

For a **genuinely new wallet directory**, verify the important paths do not already exist:

```bash
if sudo find "$HOME/tru/wallet" -mindepth 1 -maxdepth 1 -print -quit | grep -q .; then
  echo 'STOP: wallet directory is not empty. Do not bootstrap a new wallet here.'
else
  echo 'EMPTY_WALLET_DIR=PASS'
fi
```

Continue **only if the result is `EMPTY_WALLET_DIR=PASS`**. If a `tru-bootstrap` container already exists, inspect it before attempting another bootstrap. Do **not** run `docker rm -f tru-bootstrap` automatically.

Launch wallet bootstrap directly into the **persistent wallet directory**:

```bash
docker run -it \
  --name tru-bootstrap \
  --workdir /wallet \
  --mount "type=bind,src=$HOME/tru/wallet,dst=/wallet" \
  --mount "type=bind,src=$HOME/tru/tru.conf,dst=/app/tru.conf,readonly" \
  "$TRU_IMAGE" \
  --conf /app/tru.conf \
  --logdir /tmp \
  --bootstrap-encrypted-wallet
```

Choose a long unique passphrase and confirm it. The source's bootstrap function is interactive; it generates `tru.dat.enc`, `tru.dat.public`, and `wallet_seed.dat.enc` in the current working directory (`/wallet`) without printing private material.

Verify all three files before removing the bootstrap container:

```bash
for f in wallet_seed.dat.enc tru.dat.enc tru.dat.public; do
  sudo test -f "$HOME/tru/wallet/$f" || echo "MISSING: $f"
done
sudo ls -la "$HOME/tru/wallet"

# These *plaintext* files should not exist:
sudo test ! -e "$HOME/tru/wallet/tru.dat" && echo 'PLAINTEXT_WALLET_ABSENT=PASS'
sudo test ! -e "$HOME/tru/wallet/wallet_seed.dat" && echo 'PLAINTEXT_SEED_ABSENT=PASS'
```

Only after verifying success:

```bash
docker rm tru-bootstrap
```

There is **no `docker cp` step** in the revised installation: the bootstrap wrote directly into the bound host folder.

---

## 6. Back up the entire wallet — before node startup

```bash
BACKUP="$HOME/tru-wallet-$(date +%Y%m%d-%H%M%S).tgz"
sudo tar -czf "$BACKUP" -C "$HOME/tru" wallet
sudo chown "$(id -u):$(id -g)" "$BACKUP"
chmod 600 "$BACKUP"
ls -lh "$BACKUP"
```

**Copy the archive off this computer** (offline storage or your trusted encrypted backup destination). A second copy on the same disk does not protect against disk failure. Protect the passphrase separately. Never publish the backup or wallet artifacts in a repository, chat, or support ticket.

Back up again after generating PQ01 descriptors or performing a signed quantum transaction, taking care not to revert to an **older** quantum signing state.

---

## 7. Start the node — default Docker bridge networking

Docker's **bridge network** is the recommended, least-surprising default for public peer operators. Publish only P2P; run the authenticated CLI **inside the container** with `docker exec`.

### 7A. CPU image

If your image is already configured as the CPU image in Step 2:

```bash
docker run -it \
  --name tru \
  --workdir /wallet \
  -p 21833:21833/tcp \
  --mount "type=volume,src=tru-data,dst=/app/data" \
  --mount "type=bind,src=$HOME/tru/tru.conf,dst=/app/tru.conf,readonly" \
  --mount "type=bind,src=$HOME/tru/wallet,dst=/wallet" \
  "$TRU_IMAGE" \
  --conf /app/tru.conf \
  --datadir /app/data/utxo \
  --cli
```

### 7B. NVIDIA GPU image

Use this **instead of**, not in addition to, 7A. It uses the selected GPU image and requires NVIDIA Container Toolkit:

```bash
docker run -it \
  --gpus all \
  --name tru \
  --workdir /wallet \
  -p 21833:21833/tcp \
  --mount "type=volume,src=tru-data,dst=/app/data" \
  --mount "type=bind,src=$HOME/tru/tru.conf,dst=/app/tru.conf,readonly" \
  --mount "type=bind,src=$HOME/tru/wallet,dst=/wallet" \
  "$TRU_IMAGE" \
  --conf /app/tru.conf \
  --datadir /app/data/utxo \
  --cli
```

**Important notes:**

- Choose CPU **or** GPU; they both use the same logical node name `tru`.
- You should be prompted for your encrypted-wallet passphrase.
- The Docker volume `tru-data` persists block/UTXO data and node logs across ordinary container removal/recreation.
- The bind mount `~/tru/wallet:/wallet` persists **all** wallet companion files, including `tru.dat.pq01`.
- Do not run `docker rm -f tru` just to try again; inspect or stop the existing container first.
- Bridge-mode Core RPC binds to **container** `127.0.0.1`. This guide intentionally **does not** publish `21832` to the host: Docker port forwarding cannot reach a process bound only to the container's loopback address. Use `docker exec` for administrative RPC, or Step 11 if you require Linux host-loopback RPC access.

---

## 8. Detach, reattach, stop, start, and restart

While attached to the interactive Core menu, **Ctrl+C may shut the node down**. To leave it running, press:

```text
Ctrl+P, then Ctrl+Q
```

Return to the menu:

```bash
docker attach tru
```

Inspect status/logs without attaching:

```bash
docker ps --filter 'name=^/tru$'
docker logs --tail 80 tru
docker logs -f tru                # Ctrl+C stops following logs, NOT Core
```

Graceful stop: attach and use the Core's Exit menu command. If the menu is inaccessible:

```bash
docker stop -t 60 tru
```

Restart a **stopped existing** container and enter the passphrase:

```bash
docker start -ai tru
```

Last resort only:

```bash
docker kill tru
```

A hard kill can interrupt wallet and database writes. Do not delete unknown lock files automatically; first confirm no Core process uses the same data directory.

### Automatic restart after reboot?

Not with this **interactive** encrypted-wallet setup. After a host reboot, run `docker start -ai tru`, type the passphrase, then detach. Using `--restart=always` or `unless-stopped` without a separately reviewed unlock strategy may make Docker report `Up` while Core is awaiting its passphrase and not serving peers. **Never put the passphrase in a command line or plaintext service file to automate this.**

---

## 9. Verify synchronization, peers, consensus, and chain audit

All commands below are executed from the **Docker host** but run the **authenticated CLI inside the container**. The image's CLI uses the server's local RPC cookie/token when available.

```bash
# Node version, current height, connections, wallet and audit status
docker exec tru /app/tru-cli raw getinfo '{}'

# Peer IPs, direction, handshakes, versions, remote heights
docker exec tru /app/tru-cli raw getpeerinfo '{}'

# Simple friendly CLI summaries
docker exec tru /app/tru-cli getblockcount
docker exec tru /app/tru-cli getconnectioncount
docker exec tru /app/tru-cli getchaininfo

# Activation candidate height, opcode/market features
docker exec tru /app/tru-cli raw getconsensus09info '{}'

# Explicit hash/PoW audit of active in-memory chain
docker exec tru /app/tru-cli raw auditchain '{}'
```

**Interpretation:**

- `getpeerinfo` with `handshakeComplete=true` means the P2P handshake completed; a connected peer can still be behind in block height.
- Compare both node heights **and** `bestblockhash` at the same block height when investigating a possible divergence.
- `getconsensus09info` evaluates activation for the next candidate block. Under the configured **50,000** rule, behavior changes at the boundary; test blocks **49,999 → 50,000 → 50,001** with matching binaries.
- `auditchain` reports `scope: in_memory_block_hash_and_pow_walk` and `transaction_replay: false`: a PASS is **not** a full independent transaction replay or full UTXO-state audit. The `same_tip` field describes whether the tip was unchanged through that check.

If peers connect but height stalls:

```bash
docker logs tru 2>&1 | \
  grep -Ei 'peer|handshake|sync|block|height|reorg|error|reject' | tail -120
```

Before a scheduled consensus activation, verify more than one independent node reports compatible version, candidate height, and block hash. Do not mistake a quick audit for comprehensive consensus-boundary testing.

---

## 10. Network ports, NAT/CGNAT, Starlink, and peer visibility

| Port | Use | Default exposure in this guide |
|---|---|---|
| `21833/tcp` | P2P | Published on all host interfaces by Docker |
| `21832/tcp` | Authenticated JSON-RPC | **Not published; container-only localhost** |
| `8001/tcp` | Optional explorer | Off by default |

Test outbound connectivity **from the Docker host**:

```bash
nc -vz -w 5 137.184.68.43 21833
```

`nc` success means the TCP port is reachable; it does **not** prove the TRU handshake or block synchronization succeeded. Check `getpeerinfo` next.

Check exposed ports:

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'
sudo ss -lntp | grep -E '21832|21833|8001' || true
```

### DigitalOcean / public VPS

For inbound P2P connections, permit **TCP 21833** in any applicable host firewall and DigitalOcean Cloud Firewall:

```bash
sudo ufw status
# Only if UFW is in use and you WANT public inbound P2P:
sudo ufw allow 21833/tcp
```

Do **not** permit inbound TCP `21832` publicly. Port 8001 should remain loopback-restricted unless deliberately placed behind an appropriately secured HTTP front end.

### Home router / NAT / Starlink

A home node can usually make **outbound** P2P connections, sync blocks, and relay transactions without port forwarding. Accepting unsolicited **inbound** connections requires a reachable endpoint and forwarding/firewall allowance. Starlink residential IPv4 commonly operates behind CGNAT: forwarding a router port will not bypass ISP CGNAT. Setting `externalip` to a private LAN address (`192.168.x.x`, `10.x.x.x`) will not fix this.

### Why isn't my node visible on the explorer?

An explorer's peer table normally reflects **that explorer node's locally observed peers**, not a global registry of every TRU node. Outbound-only nodes can be fully synchronized yet absent from another node's explorer view. A peer showing `handshakeComplete=true` is connected to the inspecting Core even if it is still thousands of blocks behind.

---

## 11. Authenticated RPC and host-based miners

**Security in the supplied 0.08.5 Core:** RPC uses a **Bearer token** or private cookie and rejects direct browser-origin requests at its privileged `/rpc` endpoint. It is **not** unauthenticated; nevertheless, do **not** expose privileged RPC to the internet. Core also refuses non-loopback RPC binding with spending enabled.

### A. Recommended: use RPC from inside the container

```bash
docker exec tru /app/tru-cli raw getinfo '{}'
docker exec tru /app/tru-cli raw getpeerinfo '{}'
```

No `-p 21832:21832` mapping is needed or recommended in bridge mode. The CLI should be able to read the private RPC cookie or use the appropriate in-container environment provided by the image.

### B. Optional: Linux host-loopback RPC for miners/tools

**Experienced Linux operators only.** If local host applications genuinely need `127.0.0.1:21832`, an alternative is Docker **host networking**. This shares the host's network namespace, allowing Core's `rpcbind=127.0.0.1` to refer to host localhost. Do not combine host networking with `-p` options; Docker ignores publishing in host mode. **Stop the existing `tru` node first.**

Run the **same** CPU or GPU command from Step 7 with these changes:

1. Replace `-p 21833:21833/tcp` with `--network host`.
2. Retain `rpcbind=127.0.0.1` and `rpcAllowRemote=0` in `tru.conf`.
3. Preserve **the exact same** wallet directory, `tru-data` volume, image version, and `--workdir /wallet`.
4. Do **not** activate the explorer without checking and restricting its bind address; host networking may expose it on public interfaces.
5. Check `sudo ss -lntp | grep -E '21832|21833'` on the Docker host and confirm RPC listens on `127.0.0.1:21832` only.

Only one Core should use this wallet/database at a time. If the same host runs native Core already, check for port conflicts before starting Docker in host mode.

Miner examples (only once you've separately installed the corresponding miner on the Linux host and host-loopback RPC works):

```bash
# Example CPU miner invocation — check miner build's flags first
./tru_miner_cpu --node-ip 127.0.0.1 --node-port 21832 \
  --mineraddr YOUR_OWN_TRU_ADDRESS --threads 4

# Example GPU miner invocation — check miner build's flags first
./tru_miner --node-ip 127.0.0.1 --node-port 21832 \
  --mineraddr YOUR_OWN_TRU_ADDRESS
```

**Do not** enable wallet spending solely because a miner needs block-template RPC. If a separate application requires `TRU_RPC_WALLET_SEND_ENABLE=1`, assess that application and spending cap first. Privileged RPC permissions should follow the least-privilege principle.

---

## 12. Optional local explorer

The Core accepts `--enable-explorer --explorer-port 8001` as command-line options. In **bridge mode**, you may append this host-loopback-only mapping to the `docker run` command:

```bash
-p 127.0.0.1:8001:8001/tcp \
```

And after `--cli` add:

```bash
--enable-explorer --explorer-port 8001
```

The explorer's actual listener must be verified in the running build. From the Docker host test `curl -I http://127.0.0.1:8001/` (not every endpoint necessarily supports HEAD) or browse `http://127.0.0.1:8001/`. For a remote VPS, use an SSH tunnel instead of opening the port to the public internet:

```bash
ssh -L 8001:127.0.0.1:8001 USER@YOUR_VPS
```

Browse `http://127.0.0.1:8001/` from the SSH client computer. Do not enable the explorer blindly with `--network host`.

---

## 13. PQ01 / WOTS+ quantum wallet precautions

The 0.08.5 Core implements a native quantum transaction path using `OP_CHECKWOTSVERIFY` and a wallet-owned encrypted sidecar.

**Before activation** you may prepare quantum receive descriptors (assuming the node has an unlocked wallet and you deliberately enable the necessary local RPC wallet gate). **Funding or spending PQ01 outputs should wait for the configured activation, confirmed on the running chain.**

### Wallet-side files

The quantum wallet creates:

```text
/wallet/tru.dat.pq01
```

It is stored in the host's `~/tru/wallet/tru.dat.pq01` because Core uses `tru.dat` relative to its working directory, and the sidecar is exactly that wallet path plus `.pq01`.

**Never** bind-mount only the three original wallet files if using quantum functionality. Do not create an empty `.pq01` yourself; the quantum wallet initializes and encrypts it. An empty placeholder would be rejected as an invalid sidecar.

### Viewing quantum descriptors

The quantum Core RPC operations are intentionally gated: they require a local-loopback request, the spending opt-in in Core's **running** process, and the proper unlocked wallet state. Only an operator who understands the implications should change `TRU_RPC_WALLET_SEND_ENABLE=0` to `1` in `tru.conf` and then restart the node.

```bash
# Once deliberately enabled and unlocked:
docker exec tru /app/tru-cli quantum receive
docker exec tru /app/tru-cli quantum list
```

Repeat `quantum receive` to obtain a **new, distinct** descriptor. Each WOTS+ signing key is **one-use**. A successful receive command is not a transfer and does not prove that the consensus opcode is active.

**After activation**, follow the separate RPC/CLI quantum transaction guide to verify the input, choose the fee, sign with `quantum fund` / `send` / `exit`, and submit signed bytes. This path currently handles **plain TRU**, not arbitrary FT/NFT/SFT/NCFT token transfers. Never send change back to a used WOTS+ descriptor. The wallet's current implementation spends the entire selected output minus fee.

### PQ01 backup rule

Always back up the **entire current wallet directory**, and especially the up-to-date `.pq01`, after creating a descriptor and after producing a signed transaction. **Never roll back to a pre-signing `.pq01`** if it would permit reusing the same WOTS+ key. A backup copied while a transaction is being signed may be inconsistent; arrange a controlled backup process. Do not let a generic VM snapshot restore stale WOTS state without review.

---

## 14. Upgrade / replace the Docker image safely

**Do not rerun the wallet bootstrap.** A correct upgrade preserves the same wallet bind mount and `tru-data` volume. Also, confirm that the new release is compatible with the active mainnet rules before changing a node near block 50,000.

```bash
# Inspect current image and mount locations
docker inspect tru --format='Image={{.Config.Image}}'
docker inspect tru --format='{{range .Mounts}}{{println .Type .Source "->" .Destination}}{{end}}'

# Verify peers/consensus BEFORE stopping
docker exec tru /app/tru-cli raw getinfo '{}'
docker exec tru /app/tru-cli raw getconsensus09info '{}'
```

1. Back up the entire wallet folder (Step 6); preserve the latest quantum sidecar. If a wallet is actively signing, stop safely before final backup.
2. Read the new release notes and confirm correct image name/tag. `docker pull` the selected release.
3. Stop Core gracefully: `docker stop -t 60 tru`.
4. Confirm wallet state is present on the host **before removing** the old container.
5. Remove the **stopped** old container: `docker rm tru`.
6. Run the Step 7 command with the updated `TRU_IMAGE`, identical wallet mount, identical data volume, identical config, and `--workdir /wallet`.
7. Unlock, confirm `getinfo`, `getpeerinfo`, consensus and tip hash; run `auditchain` as an additional hash/PoW audit.

Do not delete Docker volume `tru-data`. Do not run two Core instances against the same volume or wallet concurrently.

---

## 15. Migrating an EXISTING three-file Docker node (important)

The older guide used `--workdir /app` and **three individual wallet-file mounts**. If your node already uses that setup, changing straight to `/wallet` without recovering sidecars could strand the existing quantum state inside the old container.

**Do not run the Step 5 bootstrap. Do not remove the old container first.**

1. Identify the existing container and confirm it is the node you intend to migrate:

```bash
docker ps -a
docker inspect tru --format='{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

2. Make an off-host backup of the three existing encrypted wallet files **and all host-side wallet companion files**. If the node is doing quantum transactions, pause those operations and arrange a controlled stop.

3. Stop gracefully:

```bash
docker stop -t 60 tru
```

4. **While the stopped container still exists**, inspect whether it contains the quantum sidecar or other wallet files in `/app`. The previous deployment wrote `tru.dat.pq01` beside `/app/tru.dat`:

```bash
# This copies out the sidecar if it exists in the OLD container.
# Run ONLY when ~/tru/wallet/tru.dat.pq01 does not already exist.
if sudo test -e "$HOME/tru/wallet/tru.dat.pq01"; then
  echo 'Existing host PQ01 sidecar: DO NOT overwrite; reconcile state manually.'
else
  docker cp tru:/app/tru.dat.pq01 "$HOME/tru/wallet/tru.dat.pq01" && \
    echo 'PQ01 sidecar recovered from old container.' || \
    echo 'No sidecar copied; if quantum was never used this may be normal.'
fi
```

5. Inspect any other wallet journals/sidecars in the stopped container, and recover legitimate state as needed. **Do not** copy an old file over a newer file with the same name; consult the actual file history. Root-owned files on the host may require `sudo` for inspection.

6. Confirm the encrypted wallet files exist on the host:

```bash
for f in wallet_seed.dat.enc tru.dat.enc tru.dat.public; do
  sudo test -f "$HOME/tru/wallet/$f" || echo "STOP: missing $f"
done
```

7. Back up the **reconciled** wallet directory. Only then remove the stopped old container and relaunch with `--workdir /wallet` and the **whole-directory** bind mount from Step 7. Retain the existing `tru-data` volume.

8. Verify the same wallet address, chain tip, block height, and (if used) the restored quantum descriptor list. Do not attempt a WOTS+ spend if the sidecar provenance is uncertain.

> This migration requires careful handling for real funds. If an old container was already removed and `.pq01` was not preserved on the host, there is no guaranteed safe automated recovery path from these instructions. Do not mint a replacement descriptor and assume it owns funds locked to the lost one.

---

## 16. Restore and disaster recovery

### Reinstall Docker / host without changing wallet identity

1. Install Docker, select the correct compatible Core image and config.
2. Restore the **whole wallet directory** from an authentic, sufficiently current backup.
3. Retain the original passphrase and never overwrite an existing newer `.pq01` with an older copy.
4. Recreate or restore `tru-data`; if unavailable, chain data can ordinarily be downloaded again from peers.
5. Start via Step 7. Compare wallet addresses, node version, chain identity, and reported tip against independent peers.

Example **only on a new blank destination**:

```bash
mkdir -p "$HOME/tru"
sudo tar -xzf /path/to/YOUR_VERIFIED_WALLET_BACKUP.tgz -C "$HOME/tru"
sudo chmod 700 "$HOME/tru/wallet"
sudo ls -la "$HOME/tru/wallet"
```

Never restore an older wallet snapshot over newer live quantum signing state. If the wallet has already signed PQ01 spends since that backup, stop and reconcile with expert review before continuing.

---

## 17. Troubleshooting

### A. `docker pull` fails (`manifest unknown`, `denied`, or `not found`)

The GHCR namespace/tag may be wrong, absent, private, or not published. Compare with the publisher's actual release. **Do not assume a public package exists because the tag is shown in a README.**

### B. `exec: ./... no such file` after changing `--workdir`

The image's entrypoint is relative to `/app`. This guide **requires an absolute Core entrypoint**; inspect `docker image inspect "$TRU_IMAGE"`, then use a verified absolute entrypoint. Do not switch back to `-w /app` without redesigning sidecar persistence.

### C. Container says `Up` but no RPC or peers work

It may be awaiting its passphrase. Reattach:

```bash
docker attach tru
```

### D. `Conflict. The container name "/tru" is already in use`

```bash
docker ps -a --filter 'name=^/tru$'
```

Start a stopped node with `docker start -ai tru`, or stop it cleanly and **verify backups** before replacing.

### E. `port is already allocated`

```bash
sudo ss -lntp | grep -E '21832|21833|8001' || true
docker ps
```

Only one listener can own a particular host port/address combination.

### F. Host `127.0.0.1:21832` fails, but Docker CLI works

This is **expected** with bridge mode and Core bound to container `127.0.0.1`. Use `docker exec` or the Linux host-network alternative (Step 11). Do not “fix” this by exposing a wallet-enabled privileged RPC service on `0.0.0.0`.

### G. RPC `401 Unauthorized` or missing cookie

0.08.5 RPC is token-authenticated. The client needs access to the server's RPC cookie/token from the same container environment. Check `docker exec tru /app/tru-cli raw getinfo '{}'` and inspect cookie-path diagnostics in startup logs **without printing or sharing the cookie's contents**. A client running outside the container does not automatically inherit the container's private cookie.

### H. `getpeerinfo` shows connections but a low block height

A completed handshake does not mean fully synced. Monitor `getblockcount` and compare tip hashes at matching heights. Sync can take time, and another peer may also be behind.

### I. TCP test succeeds but explorer doesn't list my node

A TCP port test only checks a socket. The explorer is not a global peer directory; confirm the actual TRU handshake and session observations.

### J. Docker creates a directory instead of a wallet file

This happens with certain missing bind-mount source paths in the **old individual-file** setup. The new whole-directory mount avoids that class of mistake, but you must still verify the encrypted files **exist** before starting.

### K. `Quantum sidecar missing` / `Invalid quantum sidecar envelope`

Confirm `/wallet/tru.dat.pq01` is the **actual current encrypted sidecar**. Do not create a blank file, overwrite with an old snapshot, or retry signing with guessed replacement state. Recover the correct current file and validate wallet ownership first.

### L. `docker run --gpus all` fails

Confirm `nvidia-smi` on the host, then check/install the NVIDIA Container Toolkit per its official installation instructions. CPU images do not require GPUs.

### M. UFW or DigitalOcean Cloud Firewall blocks P2P

Permit only TCP `21833` as needed. Check both local and provider firewalls. Residential CGNAT cannot generally be solved by local port forwarding.

---

## 18. Security checklist

- [ ] I verified the image publisher, exact version tag, and entrypoint.
- [ ] RPC `21832` is **not exposed publicly**.
- [ ] I understand that RPC has bearer authentication **and still must be private**.
- [ ] My wallet directory is a persistent host mount and not inside an ephemeral container layer.
- [ ] My wallet directory, passphrase, current `.pq01`, and any companion files have a protected off-host recovery plan.
- [ ] I will not restore a stale PQ01 state that could reuse one-time signing material.
- [ ] I am running only **one Core** against this wallet and database.
- [ ] `automaticReorg=1` is an intentional choice and consensus updates have been tested.
- [ ] I will verify activation with `getconsensus09info` and validate across independent peers.
- [ ] I will not publish wallet files, private keys, RPC cookies, backups, or seed files.
- [ ] Public P2P exposure, Docker firewall rules, and provider firewall rules are deliberate.

---

## 19. Useful commands and safe uninstall

```bash
# Container status
docker ps -a --filter 'name=^/tru$'

# Chain and peer status
docker exec tru /app/tru-cli raw getinfo '{}'
docker exec tru /app/tru-cli raw getpeerinfo '{}'
docker exec tru /app/tru-cli raw getconsensus09info '{}'
docker exec tru /app/tru-cli raw auditchain '{}'

# Logs
docker logs --tail 100 tru
docker logs -f tru

# Reattach, stop, restart
docker attach tru
docker stop -t 60 tru
docker start -ai tru

# Image and mounts
docker inspect tru --format='{{.Config.Image}}'
docker inspect tru --format='{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'

# Port diagnostics
sudo ss -lntp | grep -E '21832|21833|8001' || true
nc -vz -w 5 137.184.68.43 21833
```

### Remove only the container, keep all data

```bash
docker stop -t 60 tru
docker rm tru
```

The persistent host wallet directory and `tru-data` volume remain intact.

### Permanently deleting data

**DANGER: Destructive operations are intentionally not supplied as copy-and-paste commands.** Deleting `~/tru/wallet` can permanently destroy spend authority; deleting `tru-data` forces a re-sync and may remove useful diagnostics or chain state. Make verified offline backups and confirm exact resource ownership before intentional deletion.

---

### Project

**TRU — Tokenized Real Utility**  
Website/explorer: https://tokenizedrealutility.com/  
Mainnet P2P example: `137.184.68.43:21833`  
Core release documented: **0.08.5**  

