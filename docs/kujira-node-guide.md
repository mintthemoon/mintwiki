# Kujira Node Guide

## Requirements
This guide is written for Ubuntu 22.04.


## User setup
Best practice is to run node software on an isolated unprivileged user. We'll create the `kuji` user in this guide; if your username is different change it wherever it appears.

### Create the kuji user
```bash
sudo useradd -m -s /bin/bash kuji
```


## Build environment setup

### Install build packages
```bash
sudo apt install -y build-essential git unzip curl
```

### Install go toolchain
1. Download and extract go 1.18.5.
```bash
curl -fsSL https://golang.org/dl/go1.18.5.linux-amd64.tar.gz | sudo tar -xzC /usr/local/go
```
2. Configure environment variables for `kuji`.
```bash
sudo su -l kuji
cat <<EOF >> ~/.bashrc
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF
source ~/.bashrc
go version  # should output 1.18.5
exit
```


## Build node binary
Build and install the binary as the user that will run it.
```bash
sudo su -l kuji
git clone https://github.com/Team-Kujira/core kujira-core
cd kujira-core
git checkout v0.4.1
make install
kujirad version  # should output 0.4.1
exit
```


## Configure the chain
This section is written for mainnet (kaiyo-1), modify the ID and genesis URL as needed.

1. Login as `kuji`.
```bash
sudo su -l kuji
```
2. Initialize config files and dirs. Replace `<moniker>` with a public name for your node.
```bash
kujirad init --chain-id kaiyo-1 <moniker>
```
3. Download the kaiyo-1 `genesis.json`.
```bash
curl -fsSL -o $HOME/.kujira/config/genesis.json https://raw.githubusercontent.com/Team-Kujira/networks/master/mainnet/kaiyo-1.json
```
4. Set the chain-id. This will save it in `client.toml` so you won't need `--chain-id` again.
```bash
kujirad config chain-id kaiyo-1
```
5. Set some defaults in `config.toml` and `app.toml`.
```bash
sed -i 's/^timeout_commit =.*/timeout_commit = "1500ms"/' $HOME/.kujira/config/config.toml
sed -i 's/^minimum-gas-prices =.*/minimum-gas-prices = "0.00125ukuji,0.00125ibc\/295548A78785A1007F232DE286149A6FF512F180AF5657780FC89C009E2C348F,0.000125ibc\/27394FB092D2ECCD56123C74F36E4C1F926001CEADA9CA97EA622B25F41E5EB2,0.00125ibc\/47BD209179859CDE4A2806763D7189B6E6FE13A17880FE2B42DE1E6C1E329E23,0.00125ibc\/EFF323CC632EC4F747C61BCE238A758EFDB7699C3226565F7C20DA06509D59A5"/' $HOME/.kujira/config/app.toml
```
6. (Optional) Configure some seeds. This will help your node find peers.
```bash
sed -i 's/^seeds =.*/seeds = "70c1d37dd9337d7162d67f49340805360e33b2c5@seed.kujira.mintserve.org:20656,ade4d8bc8cbe014af6ebdf3cb7b1e9ad36f412c0@seeds.polkachu.com:18656"/' $HOME/.kujira/config/config.toml
```

## Start the node
Congrats! You're ready to go.

1. (Optional) [Statesync](/kujira-statesync) if you want a head start over syncing from scratch.
2. Start syncing blocks!
```bash
kujirad start
```

## Create a service
A systemd service will keep `kujirad` running in the background and restart it if it stops.

1. Create the service file with `sudo` using your favorite text editor.
```ini title="/etc/systemd/system/kujirad.service"
[Unit]
Description=kujirad
After=network.target

[Service]
Type=simple
User=kuji
ExecStart=/home/kuji/go/bin/kujirad start
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
sudo systemctl start kujirad
```
4. Tail your service logs.
```bash
sudo journalctl -fu kujirad
```
5. (Optional) Enable the service. This will set it to start on every boot.
```bash
sudo systemctl enable kujirad
```
