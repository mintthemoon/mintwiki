# Horcrux Guide

## Introduction
Horcrux is a multi-party computation signing service for Tendermint. It allows you to
split your private key into multiple shares that can sign as your validator when
a specified threshold has been reached.

!!! warning "HERE BE DRAGONS"
    This is experimental software and messing with signing is risky! This guide is written
    to be as safe as possible but no guarantees can be made and only you are responsible
    for the results of following it. For advanced users, please only attempt this if
    you've done the research to understand it.

### Advantages
#### Security
- The compromise of a single node won't expose your whole key.
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
- 1+ servers with minimum requirements for your chain's node software. More is better!
- This guide is written for Ubuntu 22.04.

### Networking
#### Signer peer-to-peer
Horcrux signers communicate with each other to reach consensus and sign transactions
over port 2222/tcp by default. It's best to have the servers communicate over a private 
network for enhanced security.

#### Node priv_validator_laddr
Signing transactions is done with the PrivValidator remote signing feature in Tendermint.
This guide will use the conventional port 1234/tcp. A private network is also recommended here;
the next best option is to allow access from only the signer IPs with a firewall.

#### Latency
Validating is sensitive to latency, and while horcrux is very efficient it adds a few
steps to the process of signing a block. It's important for your signers to have relatively short
routes to the sentries, and even more important that the signers have low latency between each other.
Exact requirements will very per chain.

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

### Initialize the config
This config will be slightly different for each signer! Individual configs are given below.

- Replace `<chain_id>` with the ID for your chain.
- Replace `<sentry_X_ip>` with the IP of your sentry. All sentries should be in the list.
- Replace `<signer_X_ip>` with the IP of your signer.
- The recommended timeout is `1500ms`, this may be too long for some chains. Change if needed.

#### Signer 1
```bash
horcrux config init \
        <chain_id> \
        "tcp://<sentry_1_ip>:1234,tcp://<sentry_2_ip>:1234" \
        --cosigner \
        --overwrite \
        --listen "tcp://<signer_1_ip>:2222" \
        --peers "tcp://<signer_2_ip>:2222|2,tcp://<signer_3_ip>:2222|3" \
        --keyfile "/home/horcrux/.horcrux/share.json" \
        --threshold 2 \
        --timeout "1500ms"
```

#### Signer 2
```bash
horcrux config init \
        <chain_id> \
        "tcp://<sentry_1_ip>:1234,tcp://<sentry_2_ip>:1234" \
        --cosigner \
        --overwrite \
        --listen "tcp://<signer_2_ip>:2222" \
        --peers "tcp://<signer_1_ip>:2222|1,tcp://<signer_3_ip>:2222|3" \
        --keyfile "/home/horcrux/.horcrux/share.json" \
        --threshold 2 \
        --timeout "1500ms"
```

#### Signer 3
```bash
horcrux config init \
        <chain_id> \
        "tcp://<sentry_1_ip>:1234,tcp://<sentry_2_ip>:1234" \
        --cosigner \
        --overwrite \
        --listen "tcp://<signer_3_ip>:2222" \
        --peers "tcp://<signer_1_ip>:2222|1,tcp://<signer_2_ip>:2222|2" \
        --keyfile "/home/horcrux/.horcrux/share.json" \
        --threshold 2 \
        --timeout "1500ms"
```

## Create the key shares
Now it's time to split up your private key! If you have an existing `priv_validator_key.json`
on a validator node, it makes sense to install horcrux there for this step since your private
key is already exposed on the server.

