# Horcrux Guide

## Introduction
Horcrux is a multi-party computation signing service for Tendermint. It allows you to
split your private key into multiple shares that can sign as your validator when
a specified threshold has been reached.

!!! warning "HERE BE DRAGONS"
    This is experimental software and messing with signing is risky! This guide is written
    to be as safe as possible but no guarantees can be made and only you are responsible
    for the results of following it. For advanced users; please only attempt this if
    you've done the research to understand it. Try it on a testnet first.

### Advantages
#### Security
- The compromise of a single signer won't expose your whole key.
- Keys don't need to be stored on the server running the node.
- Signers can talk to sentries over private networks with limited access.

#### Stability
- Continues to work when signers go down if enough stay active to reach the threshold. 
- Multiple sentry nodes can be configured for high availability (great for updates!).

### Architecture
There are many ways horcrux can be configured. The number of signers and sentries can vary
and each signer configures which sentries it talks to separately. This guide will setup
a 3-signer cluster with a signing threshold of 2. All signers connect to all sentries
for the best possible fault tolerance. At least one sentry is required, at least two is
recommended for the high availability benefits, and the more the merrier!


## Requirements
### Servers
- 3 servers with minimum requirements for signers: 1 CPU/1GB RAM/10GB SSD
- 1+ servers with minimum requirements for validating your chain. More is better!
- This guide is written for Ubuntu 22.04.

### Software
This guide is written for horcrux v2.0.0 and should work for any chain that version supports.
Do your research to make sure horcrux is compatible with your chain.

### Networking
#### Signer peer-to-peer
Horcrux signers communicate with each other to reach consensus and sign transactions
over port 2222/tcp by default. It's best to have the servers communicate over a private 
network for enhanced security.

#### Node priv_validator_laddr
Signing transactions is done with the remote signing feature in Tendermint. This guide
will use the conventional port 1234/tcp. A private network is also recommended here;
the next best option is to allow access from only the signer IPs with a firewall.

#### Latency
While horcrux is efficient it adds a few steps to the process of signing a block. 
It's important for your signers to have relatively short routes to the sentries, and even 
more important that the signers have low latency between themselves. Exact requirements 
will vary by chain.

## Prepare the sentries
Configure your sentries as regular full nodes -- you can use the Node Guide for chains that
mintwiki supports. Leave them running during this process so they're synced and ready when
we need them.

## Prepare the signers
Run the steps in this section on each of your 3 signer nodes.

### Create a user
Best practice is to run signer software on an isolated unprivileged user. We'll create the
`horcrux` user in this guide; if your username is different change it wherever it appears.

```bash
sudo useradd -m -s /bin/bash horcrux
```

### Install horcrux
Download and install `horcrux` v2.0.0 from the binary release.
```bash
curl -fsSL https://github.com/strangelove-ventures/horcrux/releases/download/v2.0.0/horcrux_2.0.0_linux_amd64.tar.gz | sudo tar -xzC /usr/local/bin -- horcrux
sudo chown root:root /usr/local/bin/horcrux
horcrux version  # output should contain '"version": "2.0.0"'
```

### Initialize config
This command will be slightly different for each signer. Individual configs are given below.

- Replace `<chain_id>` with the ID for your chain.
- Replace `<sentry_X_ip>` with the IP of your sentry. All sentries should be in the list.
- Replace `<signer_X_ip>` with the IP of your signer.
- The timeout is `1500ms`, this may not be the optimal setting for all chains.

#### Signer 1
```bash
horcrux config init <chain_id> \
    "tcp://<sentry_1_ip>:1234,tcp://<sentry_2_ip>:1234" \
    --cosigner \
    --listen "tcp://<signer_1_ip>:2222" \
    --peers "tcp://<signer_2_ip>:2222|2,tcp://<signer_3_ip>:2222|3" \
    --keyfile "/home/horcrux/.horcrux/share.json" \
    --threshold 2 \
    --timeout "1500ms"
```

#### Signer 2
```bash
horcrux config init <chain_id> \
    "tcp://<sentry_1_ip>:1234,tcp://<sentry_2_ip>:1234" \
    --cosigner \
    --listen "tcp://<signer_2_ip>:2222" \
    --peers "tcp://<signer_1_ip>:2222|1,tcp://<signer_3_ip>:2222|3" \
    --keyfile "/home/horcrux/.horcrux/share.json" \
    --threshold 2 \
    --timeout "1500ms"
```

#### Signer 3
```bash
horcrux config init <chain_id> \
    "tcp://<sentry_1_ip>:1234,tcp://<sentry_2_ip>:1234" \
    --cosigner \
    --listen "tcp://<signer_3_ip>:2222" \
    --peers "tcp://<signer_1_ip>:2222|1,tcp://<signer_2_ip>:2222|2" \
    --keyfile "/home/horcrux/.horcrux/share.json" \
    --threshold 2 \
    --timeout "1500ms"
```

### Create a service
A systemd service will keep `horcrux` running in the background and restart it if it stops.

1. Create the service file with `sudo` using your favorite text editor.
```ini title="/etc/systemd/system/horcrux.service"
[Unit]
Description=horcrux
After=network.target

[Service]
Type=simple
User=horcrux
ExecStart=/usr/local/bin/horcrux cosigner start
Restart=on-abort
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target  
```
2. Reload `systemd` to pick up the new service.
```bash
sudo systemctl daemon-reload
```

## Create key shares
Now it's time to split up your private key. You need a machine with your `priv_validator_key.json`
present and horcrux installed. If you have a running validator node that you're converting you can 
do this on the server.

```bash
horcrux create-shares <path_to_key>/priv_validator_key.json 2 3
```

The above command will create 3 files in your current directory: `private_share_1.json`,
`private_share_2.json`, and `private_share_3.json`. Each file should be moved to the 
corresponding signer (the order is important) at `/home/horcrux/.horcrux/share.json`.

Clean up when you're done by removing the share files and the original key from the server.
Make sure you have a backup! Backing up the share files will ensure you can recover from a
single signer failure, otherwise loss of one key will force you to recreate them all.

## Start signing with horcrux
This is the big moment! It's time to stop your validator and prepare your signers to take over.
You'll have some downtime while things are set up, but if all goes well it will be short and
it's best not to take chances on this step.

!!! danger "DANGER ZONE"
    Leaving your validator up or doing things in the wrong order can result in double signing.
    Be careful!

### Configure signer state
Each signer keeps track of the last block height the validator key was used to sign and also the
last sign height for its own share. This protects against double signing so you should configure
the state files to match the last sign height from your validator.

1. Read the state file from your validator. Horcrux uses a different format from Tendermint so 
the command below will output it in a way the signers can read.
    - Stop your validator first!
    - Install `jq` if you don't have it already.
```bash
sudo apt install -y jq
```
    - Replace `<path_to_state>` with the path to your state file (usually a `data` folder).
```bash
PRIV_VALIDATOR_STATE="<path_to_state>/priv_validator_state.json"
cat <<EOF
{
    "height": $(cat $PRIV_VALIDATOR_STATE | jq '.height'),
    "round": "$(cat $PRIV_VALIDATOR_STATE | jq '.round')",
    "step": $(cat $PRIV_VALIDATOR_STATE | jq '.step')
}
EOF
```
    - The output should look like this with different numbers:
```json
{
    "height": "2399984",
    "round": "0",
    "step": 3
}
```
2. Copy the state file output from step 1 to two places on each signer. Replace
`<chain_id>` with your chain-id.
    - `/home/horcrux/.horcrux/state/<chain_id>_priv_validator_state.json`
    - `/home/horcrux/.horcrux/state/<chain_id>_share_sign_state.json`


### Start horcrux service
The signers are ready, enable and start the service then watch the logs on each one.
```bash
sudo systemctl enable --now horcrux
sudo journalctl -fu horcrux
```
The service will have continuous errors trying to dial the sentry ports which we haven't
set up yet, this is expected but it should keep running. Leave the logs open and you can
see them connect as the sentries are configured.

### Configure sentries
The final step! As soon as the first sentry is configured and connected you should start
signing blocks again. These steps are the same for each sentry.

1. Configure the listen address. Replace `<chain_home>` with the path to your chain's config.
```bash
sed -i \
    's#priv_validator_laddr =.*#priv_validator_laddr = "tcp://0.0.0.0:1234"#g' \
    <chain_home>/config/config.toml
```
2. Restart the service and watch the logs. Replace `<node>` with your node service name (e.g. `kujirad`).
If successful, the top of the logs should contain the output `this node is a validator`.
```bash
sudo systemctl restart <node>
sudo journalctl -fu <node>
```

## THE END
You made it, congrats! It's not easy to get this far. If things didn't go as planned,
first reread the guide to see where you might have gone wrong and if you're still stuck
you can take a look at the official docs:
[Horcrux migrating.md](https://github.com/strangelove-ventures/horcrux/blob/main/docs/migrating.md).
