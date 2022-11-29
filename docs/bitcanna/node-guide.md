# Bitcanna Node Guide

## Requirements
This guide is written for Ubuntu 22.04.


## User setup
Best practice is to run node software on an isolated unprivileged user. We'll create the `bcna` user in this guide; if your username is different change it wherever it appears.

### Create the bcna user
```bash
sudo useradd -m -s /bin/bash bcna
```


## Build environment setup

### Install build packages
```bash
sudo apt install -y build-essential git unzip curl
```

### Install go toolchain
1. Download and extract go 1.18.5.
```bash
curl -fsSL https://golang.org/dl/go1.18.5.linux-amd64.tar.gz | sudo tar -xzC /usr/local
```
2. Login as `bcna`.
```bash
sudo su -l bcna
```
3. Configure environment variables for `bcna`.
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


## Build node binary
1. Login as `bcna` (skip if you're already logged in).
```bash
sudo su -l bcna
```
2. Build and install the binary as the user that will run it.
```bash
git clone https://github.com/BitCannaGlobal/bcna.git
cd bcna
git checkout v1.5.3
make install
bcnad version  # should output "1.5.3"
```


## Configure chain
This section is written for mainnet (bitcanna-1); modify the ID, genesis, and seeds as needed.

1. Login as `bcna` (skip if you're already logged in).
```bash
sudo su -l bcna
```
2. Initialize config files and dirs. Replace `<moniker>` with a public name for your node.
```bash
bcnad init --chain-id bitcanna-1 <moniker>
```
3. Download the bitcanna-1 `genesis.json`.
```bash
curl -fsSL -o $HOME/.bcna/config/genesis.json https://raw.githubusercontent.com/BitCannaGlobal/bcna/main/genesis.json
```
4. Set the chain-id. This will save it in `client.toml` so you won't need `--chain-id` again.
```bash
bcnad config chain-id bitcanna-1
```
6. (Optional) Configure some seeds. This will help your node find peers.
```bash
sed -i 's/^seeds =.*/seeds = "d6aa4c9f3ccecb0cc52109a95962b4618d69dd3f@seed1.bitcanna.io:26656,23671067d0fd40aec523290585c7d8e91034a771@seed2.bitcanna.io:26656"/' $HOME/.bcna/config/config.toml
```

## Start the node
Congrats! You're ready to go.

1. (Optional) [Statesync](/bitcanna/statesync) if you want a head start over syncing from scratch.
2. Start syncing blocks!
```bash
bcnad start
```

## Create a service
A systemd service will keep `bcnad` running in the background and restart it if it stops.

1. Create the service file with `sudo` using your favorite text editor.
```ini title="/etc/systemd/system/bcnad.service"
[Unit]
Description=bcnad
After=network.target

[Service]
Type=simple
User=bcna
ExecStart=/home/bcna/go/bin/bcnad start
Restart=on-abort
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target  
```
2. Reload `systemd` to pick up the new service.
```bash
sudo systemctl daemon-reload
```
3. Start the service.
```bash
sudo systemctl start bcnad
```
4. Tail your service logs.
```bash
sudo journalctl -fu bcnad
```
5. (Optional) Enable the service. This will set it to start on every boot.
```bash
sudo systemctl enable bcnad
```
