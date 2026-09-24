# Gno.land Validator Failover Procedure (Cold Standby)

**Objective:** Safely transition validator operations to a cold standby node in the event of a critical hardware or network failure, with a strict zero-tolerance policy for double-signing (slashing).

## Prerequisites
- A fully synced **Cold Standby** node running the exact same `gnoland` binary version.
- The cold standby does **NOT** contain the `priv_validator_key.json` (consensus key) during normal operations to structurally prevent split-brain scenarios.

## Step 1: Absolute Termination of the Active Node
Never rely on simple SSH shutdowns or network timeouts, as a temporary network split can lead to the old node waking up and double-signing.
1. Log into the hosting provider's web console (e.g., Contabo).
2. Execute a **Hard ACPI Power-Off** or completely **Destroy** the virtual machine instance.
3. Verify the node is completely offline on the blockchain explorer and cannot power back on autonomously.

## Step 2: State File Management (`priv_validator_state.json`)
The state file tracks the last signed block and round. It is critical to migrate this to prevent signing past blocks.
- **Scenario A (Disk is accessible):** Boot into rescue mode, extract `priv_validator_state.json` from the dead node, and securely transfer it to the standby node's config directory.
- **Scenario B (Disk is destroyed/unrecoverable):** Open `priv_validator_state.json` on the standby node and manually edit it. Set the `height` strictly to a block number higher than the last known block signed by the validator (verify via explorer). Set `round` and `step` to `0`.

## Step 3: Consensus Key Migration
1. Retrieve the encrypted backup of `priv_validator_key.json` from the offline air-gapped storage.
2. Decrypt it and securely transfer it to the standby node's configuration directory (e.g., `~/.gno/config/`).
3. Ensure file permissions are strictly set to `0400` and ownership is assigned to the non-root node user.

## Step 4: Pre-Flight Sync Check
Before activating the key, query the local RPC to ensure the cold standby is perfectly caught up to the network tip.

```bash
curl -s http://localhost:26657/status | grep catching_up
```
## Step 5: Activation and Monitoring

Start the validator process on the standby node.
```
sudo systemctl restart gnoland
```
Monitor the logs closely to confirm the node has resumed proposing and signing blocks without errors.
```
sudo journalctl -u gnoland -f -o cat
```
