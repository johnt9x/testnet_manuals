```
MONIKER="YOUR_MONIKER_GOES_HERE"
```
# Install dependencies
# Update system and install build tools
```
sudo apt -q update
sudo apt -qy install curl git jq lz4 build-essential
sudo apt -qy upgrade
```
Install Go
```
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.19.6.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
```
Download and build binaries
# Clone project repository
```
cd $HOME
rm -rf sao-consensus
git clone https://github.com/SaoNetwork/sao-consensus.git
cd sao-consensus
git checkout testnet1
```
# Build binaries
```
make build
```
# Prepare binaries for Cosmovisor
```
mkdir -p $HOME/.sao/cosmovisor/genesis/bin
mv build/linux/saod $HOME/.sao/cosmovisor/genesis/bin/
rm -rf build
```
# Create application symlinks
```
ln -s $HOME/.sao/cosmovisor/genesis $HOME/.sao/cosmovisor/current
sudo ln -s $HOME/.sao/cosmovisor/current/bin/saod /usr/local/bin/saod
Install Cosmovisor and create a service
# Download and install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.4.0
```
# Create service
```
sudo tee /etc/systemd/system/saod.service > /dev/null << EOF
[Unit]
Description=sao-testnet node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.sao"
Environment="DAEMON_NAME=saod"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.sao/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable saod
```
Initialize the node
# Set node configuration
```
saod config chain-id sao-testnet1
saod config keyring-backend test
saod config node tcp://localhost:49657
```
# Initialize the node
```
saod init $MONIKER --chain-id sao-testnet1
```
# Download genesis and addrbook
```
curl -Ls https://snapshots.kjnodes.com/sao-testnet/genesis.json > $HOME/.sao/config/genesis.json
curl -Ls https://snapshots.kjnodes.com/sao-testnet/addrbook.json > $HOME/.sao/config/addrbook.json
```
# Add seeds
```
sed -i -e "s|^seeds *=.*|seeds = \"59cef823c1a426f15eb9e688287cd1bc2b6ea42d@152.70.126.187:26656,a5298771c624a376fdb83c48cc6c630e58092c62@192.18.136.151:26656,af7259853f202391e624c612ff9d3de1142b4ca4@52.77.248.130:26656,c196d06c9c37dee529ca167701e25f560a054d6d@3.35.136.39:26656\"|" $HOME/.sao/config/config.toml
```
# Set minimum gas price
```
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.0001sao\"|" $HOME/.sao/config/app.toml
```
# Set pruning
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.sao/config/app.toml
```
# Set custom ports
```
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:49658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:49657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:49060\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:49656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":49660\"%" $HOME/.sao/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:49317\"%; s%^address = \":8080\"%address = \":49080\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:49090\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:49091\"%; s%^address = \"0.0.0.0:8545\"%address = \"0.0.0.0:49545\"%; s%^ws-address = \"0.0.0.0:8546\"%ws-address = \"0.0.0.0:49546\"%" $HOME/.sao/config/app.toml
```
Download latest chain snapshot
(....)
Start service and check the logs
```
sudo systemctl start saod && sudo journalctl -u saod -f --no-hostname -o cat
```
