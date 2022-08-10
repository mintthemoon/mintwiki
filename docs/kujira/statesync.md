# Kujira Statesync

## Introduction
Statesync helps you get up and running quickly by syncing the application state at a recent height and skipping historical data.

## Quickstart
The node should not be running for this process (stop the service if you have one).

1. Login as `kuji`.
```bash
sudo su -l kuji
```
2. (Optional) If your node has existing synced data, reset it. Backup your keys!
```bash
kujirad --home $HOME/.kujira tendermint unsafe-reset-all
```
3. Modify `config.toml` with the current statesync settings.
```bash
RPC=https://rpc-kujira.mintthemoon.xyz:443
LATEST_HEIGHT=$(curl -s $RPC/block | jq -r .result.block.header.height)
TRUST_HEIGHT=$((LATEST_HEIGHT - 2000))
TRUST_HASH=$(curl -s "$RPC/block?height=$TRUST_HEIGHT" | jq -r .result.block_id.hash)
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$RPC,$RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$TRUST_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" "$HOME/.kujira/config/config.toml"
```
4. Start statesyncing! This command will stop automatically once the node is synced.
```bash
kujirad start --halt-height $LATEST_HEIGHT
```
5. Restore the original `config.toml` to disable statesync.
```bash
mv $HOME/.kujira/config/config.toml.bak $HOME/.kujira/config/config.toml
```
6. Statesync is complete, start your node!
```bash
kujirad start
```