# Kujira Validator Guide

## Requirements
This guide assumes you have followed the [Node Guide](/kujira/node-guide).

## Create a wallet
1. Create the wallet and a password for the keyring. Replace `<wallet>` with any name you like, just make sure to remember it!
```bash
kujirad keys add <wallet>
```
2. Store your wallet seed and keyring password somewhere safe. Use the command below if you ever need to recover it.
```bash
kujirad keys add --recover <wallet>
```


## Create a validator
- Replace `<moniker>` with your validator display name.
- Replace `<wallet>` with the name of your validator wallet.
- Set the `--commission` values to your desired commission rates.
- Modify the chain-id from `kaiyo-1` if mainnet isn't the target.

```bash
kujirad tx staking create-validator \
    --moniker=<moniker> \
    --amount=1000000ukuji \
    --pubkey=$(kujirad tendermint show-validator) \
    --from=<wallet> \
    --chain-id=kaiyo-1 \
    --commission-max-change-rate=0.02 \
    --commission-max-rate=0.15 \
    --commission-rate=0.05 \
    --min-self-delegation=1
```

## Backup your keys!
It's very important to keep a secure copy of your validator's private key, 
which will allow you to recreate the validator if anything happens to your server.
Take a backup of this file: `$HOME/.kujira/config/priv_validator_key.json`.
If you restore it to a new node, it will begin signing blocks as your validator.
Be very careful! Signing from two nodes at the same time will result in a hard slash 
and jailing.