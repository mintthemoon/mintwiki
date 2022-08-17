# Kujira Oracle Guide

## Introduction
Validators are required to submit price feeds for the on-chain oracle.
The `price-feeder` app can get price data from multiple providers and
submit oracle votes to perform this duty.


## Requirements
- This guide assumes you are running Ubuntu 22.04.
- `price-feeder` needs access to a running node's RPC and gRPC ports.
  This guide assumes you have it on the same machine and uses `localhost`
  with default ports 26657 and 9090. Change these in `config.toml` as needed.


## User setup
Best practice is to run node software on an isolated unprivileged user.
We'll create the `kujioracle` user in this guide; if your username is different 
change it wherever it appears.

### Create the `kujioracle` user
```bash
sudo useradd -m -s /bin/bash kujioracle
```


## Build environment setup
If you've already followed the [Node Guide](/kujira/node-guide), the only steps 
you need in this section are **Install go toolchain** #2 and #3. Repeating the 
others won't hurt if you want to be safe.

### Install build packages
```bash
sudo apt install -y build-essential git unzip curl
```

### Install go toolchain
1. Download and extract go 1.18.5.
```bash
curl -fsSL https://golang.org/dl/go1.18.5.linux-amd64.tar.gz | sudo tar -xzC /usr/local
```
2. Login as `kujioracle`.
```bash
sudo su -l kujioracle
```
3. Configure environment variables for `kujioracle`.
```bash
cat <<EOF >> ~/.bashrc
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF
source ~/.bashrc
go version  # should output "go version go1.18.5 linux/amd64"
```


## Build `price-feeder`
1. Login as `kujioracle` (skip if you're already logged in).
```bash
sudo su -l kujioracle
```
2. Build `kujirad` v0.5.0. We need the repo to build `price-feeder` and we'll
  use the binary to create the keyring file.
```bash
git clone https://github.com/Team-Kujira/core
cd core
git checkout v0.5.0
make install
cd ..
kujirad version  # should output "0.5.0"
```
3. Build `price-feeder`.
```bash
git clone https://github.com/Team-Kujira/oracle-price-feeder
cd oracle-price-feeder
make install
price-feeder version
```
## Configure `price-feeder`

### Create a wallet
This wallet will be relatively insecure, only store the funds you need to send votes.

1. Login as `kujioracle` (skip if you're already logged in).
```bash
sudo su -l kujioracle
```
2. Create the wallet and a password for the keyring.
```bash
kujirad keys add oracle
```
3. Configure the `keyring-file` directory. This allows `price-feeder` 
to find the keyring.
```bash
mkdir ~/.kujira/keyring-file
mv ~/.kujira/*.info ~/.kujira/*.address ~/.kujira/keyring-file
```

### Set the feeder delegation
This step will authorize your new wallet to send oracle votes on behalf of your validator.
The transaction should be sent from the validator wallet so run on a node where it's available.

- Replace `<oracle_wallet>` with your new wallet address.
- Replace `<validator_wallet>` with you validator wallet name.

```bash
kujirad tx oracle set-feeder <oracle_wallet> --from <validator_wallet> --fees 250ukuji
```


### Create the config
1. Login as `kujioracle` (skip if you're already logged in).
```bash
sudo su -l kujioracle
```
2. Create the config file with your favorite text editor.
    - Replace `<wallet_address>` with your oracle wallet address (e.g. `kujira16jchc8l8hfk98g4gnqk4pld29z385qyseeqqd0`)
    - Replace `<validator_address>` with your validator address (e.g. `kujiravaloper1e9rm4nszmfg3fdhrw6s9j69stqddk7ga2x84yf`)

    ```toml title="~/config.toml"  
    gas_adjustment = 1.5
    gas_prices = "0.00125ukuji"
    enable_server = true
    enable_voter = true
    provider_timeout = "500ms"

    [server]
    listen_addr = "0.0.0.0:7171"
    read_timeout = "20s"
    verbose_cors = true
    write_timeout = "20s"

    [[deviation_thresholds]]
    base = "USDT"
    threshold = "2"

    [[currency_pairs]]
    base = "ATOM"
    providers = [
      "binance",
      "kraken",
      "osmosis",
    ]
    quote = "USD"

    [account]
    address = "<wallet_address>"
    chain_id = "kaiyo-1"
    validator = "<validator_address>"
    prefix = "kujira"

    [keyring]
    backend = "file"
    dir = "/home/kujioracle/.kujira"

    [rpc]
    grpc_endpoint = "localhost:9090"
    rpc_timeout = "100ms"
    tmrpc_endpoint = "http://localhost:26657"

    [telemetry]
    enable_hostname = true
    enable_hostname_label = true
    enable_service_label = true
    enabled = true
    global_labels = [["chain_id", "kaiyo-1"]]
    service_name = "price-feeder"
    type = "prometheus"

    [[provider_endpoints]]
    name = "binance"
    rest = "https://api1.binance.us"
    websocket = "stream.binance.us:9443"
    ```

#### Validate the currency pairs
It's important to only submit price votes for whitelisted denoms in order to avoid slashing.
Check this link to see the currently configured denoms and update `[[currency_pairs]]` if needed:
[kaiyo-1 oracle parameters](https://lcd.kaiyo.kujira.setten.io/oracle/params).

## Run `price-feeder`
1. Login as `kujioracle` (skip if you're already logged in).
```bash
sudo su -l kujioracle
```
2. Run the app, providing the password on `stdin`.
```bash
echo <keyring_password> | price-feeder ~/config.toml
```

## Create a service
A systemd service will keep `price-feeder` running in the background and restart it if it stops.

1. Create the service file with `sudo` using your favorite text editor. 
Replace `<keyring_password>` with the one you created.
```ini title="/etc/systemd/system/kujira-price-feeder.service"
[Unit]
Description=kujira-price-feeder
After=network.target

[Service]
Type=simple
User=kujioracle
ExecStart=/home/kujioracle/go/bin/price-feeder /home/kujioracle/config.toml
Restart=on-abort
LimitNOFILE=65535
Environment="PRICE_FEEDER_PASS=<keyring_password>"

[Install]
WantedBy=multi-user.target
```
2. Reload `systemd` to pick up the new service.
```bash
sudo systemctl daemon-reload
```
3. Start the service.
```bash
sudo systemctl start kujira-price-feeder
```
4. Tail your service logs.
```bash
sudo journalctl -fu kujira-price-feeder
```
5. (Optional) Enable the service. This will set it to start on every boot.
```bash
sudo systemctl enable kujira-price-feeder
```