# gno.land Mainnet (gnoland-1) Full Node Setup Guide

<img width="1398" height="728" alt="image" src="https://github.com/user-attachments/assets/acd453cc-4b02-425f-9954-076126c058d4" />

This guide walks through setting up a gno.land mainnet (chain-id `gnoland-1`, launch: 2026-09-12) full node from scratch. It includes the real issues I hit and fixed on my own server (Go version mismatch, GNOROOT error, config file location, seed format) so you do not have to run into the same ones.

> Mainnet is a **fresh chain**, not a hardfork of betanet. The validator set launched closed at genesis with 4 founding organizations (Gnocore, OnBloc, Samourai Crew, Berty). Adding new validators depends on GovDAO approval. This guide only covers **full node** setup.

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
| RPC | https://rpc.gno.land |
| Web | https://gno.land |
| Seed 1 | `g15rcv5yqef3kvnmueqvkyw8y05sd40jz9p3n5su@seed-1.gno.land:26656` |
| Seed 2 | `g1ck2yeyvvnpl92237gcea0z68jx07a4nnyvuaan@seed-2.gno.land:26656` |
| Official Release | https://github.com/gnolang/gno/releases/tag/chain/mainnet |
| Genesis SHA256 | `ea22691003130eae3ba975b7d16460706b5d75ce6c04ae82c0c4faeab7de91f0` |

Warning: the seed addresses must use the full `node_id@host:port` format. If you enter only `seed-1.gno.land` or `seed-1.gno.land:26656`, the node will not connect to any peer and will hang forever on "Blockpool has no peers". We found this the hard way, the IDs are ready above so you do not have to.

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

If the output is below 1.22 (for example an older Ubuntu package shipping 1.21.x), the mainnet binary will not build or will not run. Upgrade it:

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

Update these with your own values:

```bash
echo 'export WALLET="your-wallet-name"' >> $HOME/.bash_profile
echo 'export MONIKER="your-node-name"' >> $HOME/.bash_profile
echo 'export GNOLAND_PORT="26"' >> $HOME/.bash_profile
source $HOME/.bash_profile
```

`GNOLAND_PORT` avoids conflicts if you are already running another chain or testnet node on the same server (for example, use 58 instead of the default 26: RPC on 58657, P2P on 58656).

## Step 4: Clone the Source and Build the Binary

```bash
cd $HOME
git clone https://github.com/gnolang/gno.git
cd gno && git checkout chain/mainnet
make -C gno.land install.gnoland install.gnokey
```

Verify:

```bash
gnoland version
gnokey version
```

Warning: if you are already running another gno chain (a testnet) on the same server, `make install` will overwrite the previous binary. Back it up under a different name first:
```bash
cp $HOME/go/bin/gnoland $HOME/go/bin/gnoland-old-chain-name
```

## Step 5: Node Configuration

```bash
cd $HOME
gnoland config init --config-path $HOME/gnoland-data/config/config.toml
gnoland secrets init --data-dir $HOME/gnoland-data/secrets/
```

Warning: the config file location matters. The `gnoland start` command looks for the config at `<data-dir>/config/config.toml`. Write it directly into the `config/` subfolder as shown above, otherwise you will hit "unable to load config; config file does not exist".

Apply the settings:

```bash
gnoland config set rpc.laddr tcp://0.0.0.0:${GNOLAND_PORT}657 --config-path $HOME/gnoland-data/config/config.toml
gnoland config set p2p.laddr tcp://0.0.0.0:${GNOLAND_PORT}656 --config-path $HOME/gnoland-data/config/config.toml
gnoland config set proxy_app tcp://127.0.0.1:${GNOLAND_PORT}658 --config-path $HOME/gnoland-data/config/config.toml
gnoland config set moniker "$MONIKER" --config-path $HOME/gnoland-data/config/config.toml
gnoland config set application.prune_strategy syncable --config-path $HOME/gnoland-data/config/config.toml
gnoland config set p2p.external_address "$(wget -qO- eth0.me):${GNOLAND_PORT}656" --config-path $HOME/gnoland-data/config/config.toml
gnoland config set p2p.seeds "g15rcv5yqef3kvnmueqvkyw8y05sd40jz9p3n5su@seed-1.gno.land:26656,g1ck2yeyvvnpl92237gcea0z68jx07a4nnyvuaan@seed-2.gno.land:26656" --config-path $HOME/gnoland-data/config/config.toml
gnoland config set p2p.persistent_peers "" --config-path $HOME/gnoland-data/config/config.toml
```

## Step 6: Download and Verify the Genesis File

```bash
cd $HOME/gnoland-data/config
wget -O genesis.json.gz https://github.com/gnolang/gno/releases/download/chain%2Fmainnet/genesis.json.gz
gunzip genesis.json.gz
sha256sum genesis.json
```

Expected output:
```
ea22691003130eae3ba975b7d16460706b5d75ce6c04ae82c0c4faeab7de91f0  genesis.json
```

Warning: if the hash does not match, do not continue. Download the file again. The genesis file is large (about 56 MB compressed, larger uncompressed) because it contains 3.26 million account balances.

## Step 7: Create Your Wallet

New wallet:
```bash
gnokey add $WALLET
```
Critical: the mnemonic phrase will be shown. Save it somewhere secure and offline right away.

To recover an existing wallet:
```bash
gnokey add $WALLET --recover
```

## Step 8: Create the Systemd Service

```bash
sudo tee /etc/systemd/system/gnoland.service > /dev/null <<EOF
[Unit]
Description=gno.land Mainnet Node
After=network-online.target

[Service]
User=$USER
WorkingDirectory=$HOME
Environment=HOME=$HOME
Environment=GNOROOT=$HOME/gno
ExecStart=$(which gnoland) start \\
  --chainid gnoland-1 \\
  --genesis $HOME/gnoland-data/config/genesis.json \\
  --data-dir $HOME/gnoland-data/ \\
  --skip-genesis-sig-verification
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable gnoland
```

Warning: do not skip the `Environment=GNOROOT=$HOME/gno` line. Without it the node will crash with "panic: gno was unable to determine GNOROOT". GnoVM uses this variable to locate the standard library files.

The `--skip-genesis-sig-verification` flag is officially required for mainnet (stated in the release notes). Without it, the node rejects the genesis.

## Step 9: Firewall

```bash
sudo ufw allow ${GNOLAND_PORT}656/tcp comment "gnoland P2P"
sudo ufw allow ${GNOLAND_PORT}657/tcp comment "gnoland RPC"
```

The P2P port (656) must be open to the outside, otherwise other nodes cannot connect to you and your peer count stays low.

## Step 10: Start and Verify Sync

```bash
sudo systemctl restart gnoland
sudo journalctl -u gnoland -f
```

Note: after "InitChainer: standard libraries loaded" the logs may go quiet for a few minutes. This is normal, the 3.26M-account genesis state is being processed. Do not stop the node during this time, if it gets killed mid-way (SIGTERM timeout leading to SIGKILL) the process starts over from scratch.

Check peer connections:
```bash
curl -s http://localhost:${GNOLAND_PORT}657/net_info | jq .result.n_peers
```
You should see a number greater than 0 (typically around 10 to 15 if the seeds were entered correctly).

Check sync status:
```bash
curl -s http://localhost:${GNOLAND_PORT}657/status | jq .result.sync_info
```
When `catching_up` shows `false`, you are fully synced. Since mainnet is still new (launched September 12, 2026), sync time is short and should complete within a few hours.

---

## Useful Commands

```bash
# Service management
sudo systemctl status gnoland
sudo systemctl restart gnoland

# Live logs
sudo journalctl -u gnoland -f --no-hostname -o cat

# Sync status
curl -s http://localhost:${GNOLAND_PORT}657/status | jq .result.sync_info

# Peer count
curl -s http://localhost:${GNOLAND_PORT}657/net_info | jq .result.n_peers

# Node ID
gnoland secrets get node_id --data-dir $HOME/gnoland-data

# Wallet balance
gnokey query -remote "https://rpc.gno.land" auth/accounts/YOUR-G1-ADDRESS
```

## Common Errors: Quick Reference

| Error | Cause | Fix |
|---|---|---|
| `unable to determine GNOROOT` | `GNOROOT` not set in the service file | Add `Environment=GNOROOT=$HOME/gno` |
| `config file ... does not exist` | config.toml in the wrong folder | Move it to `<data-dir>/config/config.toml` |
| `Blockpool has no peers` (persistent) | Seeds not in `id@host:port` format | Use the full seed addresses above |
| Genesis hash mismatch | Corrupted or stale download | Re-download the file and compare against the official hash |
| Previous chain's binary got overwritten after `make install` | Multiple chains on the same server | Back up binaries under different names before building |

---

## Note: Validator Registration (Register) Step

This guide currently covers everything up to **full node setup**. Sending the `Register` call through `r/gnops/valopers` requires enough `ugnot` to cover the gas fee. Since transfers are locked at genesis under mainnet's §126 rule and there is no faucet, we do not have tokens for this yet.

Once tokens are obtained and our mainnet validator selection is finalized through the GovDAO process (Legal Review, Technical Review, approval), the validator creation and `Register` steps will be added to this guide as a separate section.

---

**This guide was prepared by ZeycaNode, drawing on operational experience from the gno.land Test13, Topaz, Sapphire, Pearl, and Mainnet processes.**
