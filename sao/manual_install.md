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
curl -Ls https://ss-t.sao.nodestake.top/addrbook.json > $HOME/.sao/config/addrbook.json
curl -Ls https://ss-t.sao.nodestake.top/genesis.json > $HOME/.sao/config/genesis.json 
```
# Add seeds
```
sed -i -e "s|^seeds *=.*|seeds = \"be6cd3a90f4ccbef6795af61055b8b0a4de172ed@86.48.16.205:49656,1ed54d64859edbfe8109155c0cf6bdb04e592cb6@142.132.248.253:65528,87aae9e66b092c79c6e5e1a7c64ec21128359f7e@144.76.97.251:37656,841ae6ca44f1d51076c75ed3753e429775cb2ad9@134.209.76.124:26656,4f7898c70637f2a5c65ea909afcd47c10f090863@213.133.100.172:27544,00c031b6c1aaf3557618c9af37455fe67b7fff9c@185.188.249.18:15656,7fe67df2d13d1b229a0e24504e7c1afe5d3c6936@143.198.204.248:27656,c9b8e6019f398935cbda6f85a915447e67aac802@178.63.52.213:49656,a5298771c624a376fdb83c48cc6c630e58092c62@192.18.136.151:26656,a5261e9fba12d7a59cd1d4515a449e705734c39b@46.101.144.90:27656,b8429de484a1cf6108d57dc69fc02cd8b7592e01@157.230.245.237:09656,5bf4920fac1647e12a24c0ae5af4b3ca19db2bb2@57.128.86.7:26656,613db6dc57f294bd3238b241cdb66697cfe45c4b@207.154.251.199:27656\"|" $HOME/.sao/config/config.toml
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
# Download latest chain snapshot
```
SNAP_NAME=$(curl -s https://ss-t.sao.nodestake.top/ | egrep -o ">20.*\.tar.lz4" | tr -d ">")
curl -o - -L https://ss-t.sao.nodestake.top/${SNAP_NAME}  | lz4 -c -d - | tar -x -C $HOME/.sao
```
# Start service and check the logs
```
sudo systemctl start saod && sudo journalctl -u saod -f --no-hostname -o cat
```
# Creat or recover wallet after sync fasle
Check sync info
```
saod status 2>&1 | jq .SyncInfo
```
Add new key 
```
saod keys add wallet 
```
Recover existing key 
```
saod keys add wallet --recover
```
Query wallet balance 
```
saod q bank balances $(saod keys show wallet -a)
```
# Creat-validator
```
saod tx staking create-validator \
--amount=900000sao \
--pubkey=$(saod tendermint show-validator) \
--moniker YOURNAME \
--identity YOURKEYBASEID \
--chain-id sao-testnet1 \
--commission-rate 0.10 \
--commission-max-rate 0.20 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--gas 2000000 \
--gas-prices 0.0025sao \
--from wallet \
-y
```
# Remove sao node
```
cd $HOME
sudo systemctl stop saod
sudo systemctl disable saod
sudo rm /etc/systemd/system/saod.service
sudo systemctl daemon-reload
rm -f $(which saod)
rm -rf $HOME/.sao
rm -rf $HOME/sao-consensus
```
