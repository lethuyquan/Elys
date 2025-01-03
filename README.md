Manual Installation
Official Documentation
Recommended Hardware: 4 Cores, 8GB RAM, 200GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

Node Name
Wallet
Port
38
Pruning
Pruning Keep Recent
100
Pruning Interval
19
# install go, if needed
cd $HOME
VER="1.21.3"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin

# set vars
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export ELYS_CHAIN_ID="elysicstestnet-1"" >> $HOME/.bash_profile
echo "export ELYS_PORT="38"" >> $HOME/.bash_profile
source $HOME/.bash_profile

# download binary
cd $HOME
rm -rf bin
mkdir bin && cd bin
wget https://github.com/elys-network/elys/releases/download/v1.5.0/elysd-v1.5.0-linux-amd64.tar.gz
tar -xvf elysd-v1.5.0-linux-amd64.tar.gz
chmod +x $HOME/bin/elysd
sudo mv $HOME/bin/elysd $HOME/go/bin

# config and init app
elysd init $MONIKER --chain-id $ELYS_CHAIN_ID
sed -i \
-e "s/chain-id = .*/chain-id = \"${ELYS_CHAIN_ID}\"/" \
-e "s/keyring-backend = .*/keyring-backend = \"os\"/" \
-e "s/node = .*/node = \"tcp:\/\/localhost:${ELYS_PORT}657\"/" $HOME/.elys/config/client.toml

# download genesis and addrbook
wget -O $HOME/.elys/config/genesis.json https://server-4.itrocket.net/testnet/elys/genesis.json
wget -O $HOME/.elys/config/addrbook.json  https://server-4.itrocket.net/testnet/elys/addrbook.json

# set seeds and peers
SEEDS="ae7191b2b922c6a59456588c3a262df518b0d130@elys-testnet-seed.itrocket.net:38656"
PEERS="0977dd5475e303c99b66eaacab53c8cc28e49b05@elys-testnet-peer.itrocket.net:38656,7f4a326cd0e3942203e5479f550657e09356c73c@135.181.86.200:26656,a851c1ca75aa067d795dab48c83833bf2cd3dc0c@37.27.55.100:21656,92bdb02828a5479771ce5d2ac9bbdf4d243fa304@51.178.76.62:36656,7c7a172b5d99f6bf2a91468e6d16c41c677233ee@65.108.234.115:28856,20407fc4733b0bad9b4f5e74f48a535d210259f8@65.21.116.24:26656,e1b058e5cfa2b836ddaa496b10911da62dcf182e@164.152.161.168:36656,4771b02434d797b9728a072b2373e7146fc7bb01@136.243.17.170:33657,0f9a0d0b74377b6330053131eb31b8e97d527bee@37.27.81.152:26656,ae7191b2b922c6a59456588c3a262df518b0d130@65.108.231.124:38656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.elys/config/config.toml

# set custom ports in app.toml
sed -i.bak -e "s%:1317%:${ELYS_PORT}317%g;
s%:8080%:${ELYS_PORT}080%g;
s%:9090%:${ELYS_PORT}090%g;
s%:9091%:${ELYS_PORT}091%g;
s%:8545%:${ELYS_PORT}545%g;
s%:8546%:${ELYS_PORT}546%g;
s%:6065%:${ELYS_PORT}065%g" $HOME/.elys/config/app.toml

# set custom ports in config.toml file
sed -i.bak -e "s%:26658%:${ELYS_PORT}658%g;
s%:26657%:${ELYS_PORT}657%g;
s%:6060%:${ELYS_PORT}060%g;
s%:26656%:${ELYS_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${ELYS_PORT}656\"%;
s%:26660%:${ELYS_PORT}660%g" $HOME/.elys/config/config.toml

# config pruning
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.elys/config/app.toml 
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.elys/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.elys/config/app.toml

# set minimum gas price, enable prometheus and disable indexing
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.0003uelys,0.001ibc/F082B65C88E4B6D5EF1DB243CDA1D331D002759E938A0F5CD3FFDC5D53B3E349,0.001ibc/C4CFF46FD6DE35CA4CF4CE031E643C8FDC9BA4B99AE598E9B0ED98FE3A2319F9"|g' $HOME/.elys/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.elys/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.elys/config/config.toml

# create service file
sudo tee /etc/systemd/system/elysd.service > /dev/null <<EOF
[Unit]
Description=Elys node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.elys
ExecStart=$(which elysd) start --minimum-gas-prices="0.0018ibc/2180E84E20F5679FCC760D8C165B60F42065DEF7F46A72B447CFF1B7DC6C0A65,0.00025ibc/E2D2F6ADCC68AA3384B2F5DFACCA437923D137C14E86FB8A10207CF3BED0C8D4,0.00025uelys" --home $HOME/.elys
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

# reset and download snapshot
elysd tendermint unsafe-reset-all --home $HOME/.elys
if curl -s --head curl https://server-4.itrocket.net/testnet/elys/elys_2025-01-02_631926_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-4.itrocket.net/testnet/elys/elys_2025-01-02_631926_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.elys
    else
  echo "no snapshot found"
fi

# enable and start service
sudo systemctl daemon-reload
sudo systemctl enable elysd
sudo systemctl restart elysd && sudo journalctl -u elysd -fo cat
Automatic Installation
pruning: custom: 100/0/19 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/elys/autoinstall/)
Create wallet
# to create a new wallet, use the following command. don’t forget to save the mnemonic
elysd keys add $WALLET

# to restore exexuting wallet, use the following command
elysd keys add $WALLET --recover

# save wallet and validator address
WALLET_ADDRESS=$(elysd keys show $WALLET -a)
VALOPER_ADDRESS=$(elysd keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile

# check sync status, once your node is fully synced, the output from above will print "false"
elysd status 2>&1 | jq 

# before creating a validator, you need to fund your wallet and check balance
elysd query bank balances $WALLET_ADDRESS 
Node Sync Status Checker
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.elys/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://elys-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Invalid block height data. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  if [ "$blocks_left" -lt 0 ]; then
    blocks_left=0
  fi

  echo -e "\033[1;33mNode Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"

  sleep 5
done
Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, uelys
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
cd $HOME
# Create validator.json file
echo "{\"pubkey\":{\"@type\":\"/cosmos.crypto.ed25519.PubKey\",\"key\":\"$(elysd comet show-validator | grep -Po '\"key\":\s*\"\K[^"]*')\"},
    \"amount\": \"1000000uelys\",
    \"moniker\": \"test\",
    \"identity\": \"\",
    \"website\": \"\",
    \"security\": \"\",
    \"details\": \"I love blockchain ❤️\",
    \"commission-rate\": \"0.1\",
    \"commission-max-rate\": \"0.2\",
    \"commission-max-change-rate\": \"0.01\",
    \"min-self-delegation\": \"1\"
}" > validator.json
# Create a validator using the JSON configuration
elysd tx staking create-validator validator.json \
    --from $WALLET \
    --chain-id elysicstestnet-1 \
	--gas auto --gas-adjustment 1.5 --fees 300uelys
	
Monitoring
If you want to have set up a monitoring and alert system use our cosmos nodes monitoring guide with tenderduty

Security
To protect you keys please don`t share your privkey, mnemonic and follow basic security rules

Set up ssh keys for authentication
You can use this guide to configure ssh authentication and disable password authentication on your server

Firewall security
Set the default to allow outgoing connections, deny all incoming, allow ssh and node p2p port

sudo ufw default allow outgoing 
sudo ufw default deny incoming 
sudo ufw allow ssh/tcp 
sudo ufw allow ${ELYS_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop elysd
sudo systemctl disable elysd
sudo rm -rf /etc/systemd/system/elysd.service
sudo rm $(which elysd)
sudo rm -rf $HOME/.elys
sed -i "/ELYS_/d" $HOME/.bash_profile
