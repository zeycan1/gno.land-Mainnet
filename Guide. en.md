# gno.land Mainnet (gnoland-1) Full Node Setup Guide
<img width="1398" height="728" alt="image" src="https://github.com/user-attachments/assets/ed304a69-4a93-4b40-b239-9fb5eb9ef526" />

This guide walks through setting up a gno.land mainnet (chain-id `gnoland-1`, launch: 2026-09-12) full node from scratch. Every step here is a command we actually ran and verified on our own server (ZeycaNode), not a guess on paper. It includes the real errors we hit and fixed (Go version mismatch, GNOROOT error, config file location, seed format) so you do not have to run into the same ones.

> Mainnet is a fresh chain, not a hardfork of betanet. The validator set launched closed at genesis with 4 founding organizations (Gnocore, OnBloc, Samourai Crew, Berty). Adding new validators depends on GovDAO approval. This guide covers full node setup and, optionally, exposing your RPC publicly.

---

## Hardware Requirements

| Component | Minimum | Recommended |
|---|---|---|
| OS | Ubuntu 22.04+ | Ubuntu 24.04 |
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Disk | 200 GB SSD | 500 GB NVMe SSD |
| Network | 100 Mbps | 1 Gbps |

## Network Info

| Type | Value |
|---|---|
| Chain ID | `gnoland-1` |
| Official RPC | https://rpc.gno.land |
| Web | https://gno.land |
| Seed 1 | `g15rcv5yqef3kvnmueqvkyw8y05sd40jz9p3n5su@seed-1.gno.land:26656` |
| Seed 2 | `g1ck2yeyvvnpl92237gcea0z68jx07a4nnyvuaan@seed-2.gno.land:26656` |
| Official Release | https://github.com/gnolang/gno/releases/tag/chain/mainnet |
| Genesis SHA256 | `ea22691003130eae3ba975b7d16460706b5d75ce6c04ae82c0c4faeab7de91f0` |

Warning: the seed addresses must use the full `node_id@host:port` format. Using only the hostname, or a hostname with a port but no ID, means the node will not connect to any peer and will hang forever on "Blockpool has no peers". We found this the hard way during our own setup, the correct IDs and ports are ready above.

Bonus: you can query ZeycaNode's own running mainnet node read-only while your own node is still syncing:
- RPC: https://gnoland-mainnet-rpc.zeycanode.com
- API: https://gnoland-mainnet-api.zeycanode.com

---

## Step 1: System Update and Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git wget htop tmux build-essential jq make lz4 gcc unzip \
  screen cmake perl automake autoconf libtool libssl-dev zstd pv
```

## Step 2: Install Go (1.22+ REQUIRED)

```bash
go version
```

If the output is below 1.22 (for example an older Ubuntu package shipping 1.21.x, which is what we hit ourselves), the mainnet binary will not build or will not run. Upgrade it:

```bash
cd $HOME
VER="1.23.0"
wget "https://go.dev/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"

echo 'export PATH=/usr/local/go/bin:$HOME/go/bin:$PATH' >> ~/.bash_profile
source ~/.bash_profile
go version   # should show go1.23.0
```

## Step 3: Set Your Variables

Update these with your own values. `NODE_PORT` matters especially if you already run another chain or testnet node on the same server, it avoids port conflicts:

```bash
echo 'export WALLET="your-wallet-name"' >> $HOME/.bash_profile
echo 'export MONIKER="your-node-name"' >> $HOME/.bash_profile
echo 'export NODE_PORT="58"' >> $HOME/.bash_profile
source $HOME/.bash_profile
```

`NODE_PORT=58` gives RPC on 58657 and P2P on 58656. If nothing else runs on this server, the default `26` works fine too (RPC 26657, P2P 26656).

## Step 4: Clone the Source and Check Out the Mainnet Tag

We clone into a dedicated folder (`gno-mainnet`) so it never collides with another chain's source or build folder:

```bash
cd $HOME
git clone https://github.com/gnolang/gno.git gno-mainnet
cd gno-mainnet && git checkout chain/mainnet
```

## Step 5: Build the Binary in Isolation

`make install` overwrites any existing `gnoland`/`gnokey` binaries on the system. So we build the mainnet binaries under their own names (`gnoland-mainnet`, `gnokey-mainnet`), which also avoids conflicts if another chain runs on the same server:

```bash
cd $HOME/gno-mainnet
make -C gno.land build
cp gno.land/build/gnoland $HOME/go/bin/gnoland-mainnet
cp gno.land/build/gnokey  $HOME/go/bin/gnokey-mainnet
export PATH="$HOME/go/bin:$PATH"
```

Verify:

```bash
gnoland-mainnet version
gnokey-mainnet version
```

## Step 6: Create the Node Configuration

We keep the data in its own directory (`mainnet-data`), separate from the source folder:

```bash
cd $HOME
gnoland-mainnet config init --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet secrets init --data-dir $HOME/mainnet-data/secrets/
```

Warning: the config file's final location matters. `gnoland-mainnet start` looks for the config at `<data-dir>/config/config.toml`, but `config init` initially writes it one level up, at `<data-dir>/config.toml`. We will move it into the right folder in Step 7 once the genesis file is downloaded, otherwise you will hit "unable to load config; config file does not exist".

Apply the settings:

```bash
gnoland-mainnet config set rpc.laddr tcp://0.0.0.0:${NODE_PORT}657 --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet config set p2p.laddr tcp://0.0.0.0:${NODE_PORT}656 --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet config set proxy_app tcp://127.0.0.1:${NODE_PORT}658 --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet config set moniker "$MONIKER" --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet config set application.prune_strategy syncable --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet config set p2p.external_address "$(wget -qO- eth0.me):${NODE_PORT}656" --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet config set p2p.seeds "g15rcv5yqef3kvnmueqvkyw8y05sd40jz9p3n5su@seed-1.gno.land:26656,g1ck2yeyvvnpl92237gcea0z68jx07a4nnyvuaan@seed-2.gno.land:26656" --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet config set p2p.persistent_peers "" --config-path $HOME/mainnet-data/config.toml
```

## Step 7: Download and Verify the Genesis File, Then Move the Config

```bash
mkdir -p $HOME/mainnet-data/config
cd $HOME/mainnet-data/config
wget -O genesis.json.gz https://github.com/gnolang/gno/releases/download/chain%2Fmainnet/genesis.json.gz
gunzip genesis.json.gz
sha256sum genesis.json
```

Expected output:
```
ea22691003130eae3ba975b7d16460706b5d75ce6c04ae82c0c4faeab7de91f0  genesis.json
```

Warning: if the hash does not match, do not continue, download the file again. The genesis file is large (about 56 MB compressed) because it contains 3.26 million account balances.

Now move the config.toml from Step 6 into the same folder as the genesis file:

```bash
mv $HOME/mainnet-data/config.toml $HOME/mainnet-data/config/config.toml
ls $HOME/mainnet-data/config/   # both config.toml and genesis.json should be here
```

## Step 8: Create Your Wallet

New wallet:
```bash
gnokey-mainnet add $WALLET --home $HOME/mainnet-data
```
Critical: the mnemonic phrase will be shown. Save it somewhere secure and offline right away.

To recover an existing wallet (for example, the one you used on testnets):
```bash
gnokey-mainnet add $WALLET --recover --home $HOME/mainnet-data
```

Verify:
```bash
gnokey-mainnet list --home $HOME/mainnet-data
```

## Step 9: Create the Systemd Service

```bash
sudo tee /etc/systemd/system/gno-mainnet.service > /dev/null <<SERVICEEOF
[Unit]
Description=gno.land Mainnet Node
After=network-online.target

[Service]
User=$USER
WorkingDirectory=$HOME
Environment=HOME=$HOME
ExecStart=$HOME/go/bin/gnoland-mainnet start \
  --chainid gnoland-1 \
  --genesis $HOME/mainnet-data/config/genesis.json \
  --data-dir $HOME/mainnet-data/ \
  --skip-genesis-sig-verification
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
SERVICEEOF

sudo systemctl daemon-reload
sudo systemctl enable gno-mainnet
```

Note: we do not set `GNOROOT` here because `gnoland-mainnet` was built right next to its `$HOME/gno-mainnet` source checkout, and GnoVM can locate the standard library automatically from there. If you run the binary from a different location than the source folder, or you hit "panic: gno was unable to determine GNOROOT", add this line under `[Service]`:
```
Environment=GNOROOT=$HOME/gno-mainnet
```

The `--skip-genesis-sig-verification` flag is officially required for mainnet (stated in the release notes). Without it, the node rejects the genesis.

## Step 10: Firewall

```bash
sudo ufw allow ${NODE_PORT}656/tcp comment "gno-mainnet P2P"
sudo ufw allow ${NODE_PORT}657/tcp comment "gno-mainnet RPC"
```

The P2P port must be open to the outside, otherwise other nodes cannot connect to you and your peer count stays low. Only open the RPC port publicly if you actually need external access and are not setting up an nginx proxy.

## Step 11: Start and Verify Sync

```bash
sudo systemctl restart gno-mainnet
sudo journalctl -u gno-mainnet -f
```

Note: after "InitChainer: standard libraries loaded" the logs may go quiet for a few minutes. This is normal, the 3.26M-account genesis state is being processed. Do not stop the node during this time, if it gets killed mid-way (SIGTERM timeout leading to SIGKILL) the process starts over from scratch.

Check peer connections:
```bash
curl -s http://localhost:${NODE_PORT}657/net_info | jq .result.n_peers
```
You should see a number greater than 0 (typically around 10 to 15 if the seeds were entered correctly).

Check sync status:
```bash
curl -s http://localhost:${NODE_PORT}657/status | jq .result.sync_info
```
When `catching_up` shows `false`, you are fully synced.

---

## Optional: Exposing Your RPC Publicly (nginx + Cloudflare)

If you want to serve your node over HTTPS on your own domain (we did this for mainnet-rpc/api.zeycanode.com):

```bash
sudo tee /etc/nginx/sites-available/gnoland-mainnet-rpc > /dev/null <<NGINXEOF
server {
    listen 80;
    server_name YOUR-RPC-DOMAIN;

    location / {
        proxy_pass http://127.0.0.1:58657;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;

        add_header Access-Control-Allow-Origin * always;
        add_header Access-Control-Allow-Methods "GET, POST, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Content-Type" always;
    }
}
NGINXEOF

sudo ln -s /etc/nginx/sites-available/gnoland-mainnet-rpc /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d YOUR-RPC-DOMAIN
```

Warning: if you use Cloudflare and your DNS record is in "Proxied" mode (orange cloud), certbot's HTTP-01 challenge usually fails with a 522 error. Temporarily switch the DNS record to "DNS only" (grey cloud) while issuing the certificate, then switch it back to "Proxied" and set Cloudflare's SSL mode to Full (strict).

Warning: make sure the file under `sites-enabled` is an actual symlink, not a standalone copy, otherwise future edits to `sites-available` never take effect (this happened to us on our explorer, we spent hours wondering why changes would not show up).

---

## Useful Commands

```bash
# Service management
sudo systemctl status gno-mainnet
sudo systemctl restart gno-mainnet

# Live logs
sudo journalctl -u gno-mainnet -f --no-hostname -o cat

# Sync status
curl -s http://localhost:${NODE_PORT}657/status | jq .result.sync_info

# Peer count
curl -s http://localhost:${NODE_PORT}657/net_info | jq .result.n_peers

# Node ID
gnoland-mainnet secrets get node_id --data-dir $HOME/mainnet-data

# Validator pubkey (you will need this later for Register)
gnoland-mainnet secrets get validator_key --data-dir $HOME/mainnet-data

# Wallet balance
gnokey-mainnet query -remote "https://rpc.gno.land" auth/accounts/YOUR-G1-ADDRESS --home $HOME/mainnet-data
```

## Common Errors: Quick Reference

| Error | Cause | Fix |
|---|---|---|
| `unable to determine GNOROOT` | Binary running from a different location than the source clone | Add `Environment=GNOROOT=$HOME/gno-mainnet` to the service file |
| `config file ... does not exist` | config.toml not inside the `config/` subfolder | Move it to `<data-dir>/config/config.toml` (Step 7) |
| `Blockpool has no peers` (persistent) | Seeds not in `id@host:port` format, or missing the port | Use the full seed addresses from Step 6 |
| Genesis hash mismatch | Corrupted or stale download | Re-download the file and compare against the official hash |
| Previous chain's binary got overwritten after `make install` | Multiple chains on the same server | Build binaries under separate names like `gnoland-mainnet` (Step 5) |
| nginx changes never take effect | The file under `sites-enabled` is a copy, not a symlink | Remove it and create a proper symlink |
| Explorer/browser shows stale data | Server sends no `Cache-Control` header, browser caches heuristically | Add `add_header Cache-Control "no-cache, no-store, must-revalidate" always;` in nginx |

---

## Note: Validator Registration (Register) Step

This guide currently covers everything up to **full node setup**. Sending the `Register` call through `r/gnops/valopers` requires enough `ugnot` to cover the gas fee. Since transfers are locked at genesis under mainnet's §126 rule and there is no faucet, we do not have tokens for this yet. Registering before the official validator onboarding process (KYC and Technical Review) is complete is also not recommended.

Once tokens are obtained and our mainnet validator selection is finalized through the GovDAO process (Legal Review, Technical Review, approval), the validator creation and `Register` steps will be added to this guide as a separate section.

---

**This guide was prepared by ZeycaNode, drawing on operational experience from the gno.land Test13, Topaz, Sapphire, Pearl, and Mainnet processes.**
