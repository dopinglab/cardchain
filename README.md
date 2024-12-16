Manual Installation
Official Documentation
Recommended Hardware: 4 Cores, 8GB RAM, 200GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.19.3"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

**set vars**
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export CARDCHAIN_CHAIN_ID="cardtestnet-12"" >> $HOME/.bash_profile
echo "export CARDCHAIN_PORT="31"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
wget -O Cardchaind https://github.com/DecentralCardGame/Cardchain/releases/download/v0.16.0/Cardchaind
chmod +x Cardchaind
sudo mv Cardchaind /usr/local/bin
```

**config and init app**
```
Cardchaind config node tcp://localhost:${CARDCHAIN_PORT}657
Cardchaind config keyring-backend os
Cardchaind config chain-id cardtestnet-12
Cardchaind init "test" --chain-id cardtestnet-12
```

**download genesis and addrbook**
```
wget -O $HOME/.cardchaind/config/genesis.json https://server-4.itrocket.net/testnet/cardchain/genesis.json
wget -O $HOME/.cardchaind/config/addrbook.json  https://server-4.itrocket.net/testnet/cardchain/addrbook.json
```

**set seeds and peers**
```
SEEDS="947aa14a9e6722df948d46b9e3ff24dd72920257@cardchain-testnet-seed.itrocket.net:31656"
PEERS="99dcfbba34316285fceea8feb0b644c4dc67c53b@cardchain-testnet-peer.itrocket.net:31656,ca809647d5d73ef9247e94df133b1fd40ccce827@144.217.68.182:26656,d0bb15daba08a7c84c45d8ee48daeadf42d08a6a@185.144.99.114:26656,d0e4edcdd73a7578b10980b3739a5b7218b7e86f@212.23.222.109:26256,3db8323a132c4dee2e1896fd533787b8a23d95b2@194.163.174.151:26656,734563b2bf39ddcc2c672d3e41aad1d259aca4c7@213.199.35.238:26656,86fe149f801ac75213179be5b56fbd1a1e545c43@202.61.225.157:20656,86484b4d411e2cec602c88ae1262a39ab58ea470@207.180.249.47:26656,8f665b8a8fb1a3c95fb43575a06b114d9ced4a98@84.247.138.148:26656,77aad2057be5289f44c4aa5e9df7c26b86038904@65.109.53.24:31656,1c236641285af7dac88600e43c2ca6505fbea454@65.108.2.180:31656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.cardchaind/config/config.toml
```

**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${CARDCHAIN_PORT}317%g;
s%:8080%:${CARDCHAIN_PORT}080%g;
s%:9090%:${CARDCHAIN_PORT}090%g;
s%:9091%:${CARDCHAIN_PORT}091%g;
s%:8545%:${CARDCHAIN_PORT}545%g;
s%:8546%:${CARDCHAIN_PORT}546%g;
s%:6065%:${CARDCHAIN_PORT}065%g" $HOME/.cardchaind/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${CARDCHAIN_PORT}658%g;
s%:26657%:${CARDCHAIN_PORT}657%g;
s%:6060%:${CARDCHAIN_PORT}060%g;
s%:26656%:${CARDCHAIN_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${CARDCHAIN_PORT}656\"%;
s%:26660%:${CARDCHAIN_PORT}660%g" $HOME/.cardchaind/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.cardchaind/config/app.toml 
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.cardchaind/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.cardchaind/config/app.toml
```

**set minimum gas price, enable prometheus and disable indexing**
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.0ubpf"|g' $HOME/.cardchaind/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.cardchaind/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.cardchaind/config/config.toml
```

**create service file**
```
sudo tee /etc/systemd/system/Cardchaind.service > /dev/null <<EOF
[Unit]
Description=Cardchain node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.cardchaind
ExecStart=$(which Cardchaind) start --home $HOME/.cardchaind
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

**reset and download snapshot**
```
Cardchaind tendermint unsafe-reset-all --home $HOME/.cardchaind
if curl -s --head curl https://server-4.itrocket.net/testnet/cardchain/cardchain_2024-12-01_2165013_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-4.itrocket.net/testnet/cardchain/cardchain_2024-12-01_2165013_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.cardchaind
    else
  echo "no snapshot found"
fi
```


**enable and start service**
```
sudo systemctl daemon-reload
sudo systemctl enable Cardchaind
sudo systemctl restart Cardchaind && sudo journalctl -u Cardchaind -f
Automatic Installation
pruning: custom: 100/0/19 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/cardchain/autoinstall/)
```

Create wallet
# to create a new wallet, use the following command. don’t forget to save the mnemonic
Cardchaind keys add $WALLET

# to restore exexuting wallet, use the following command
Cardchaind keys add $WALLET --recover

# save wallet and validator address
WALLET_ADDRESS=$(Cardchaind keys show $WALLET -a)
VALOPER_ADDRESS=$(Cardchaind keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile

# check sync status, once your node is fully synced, the output from above will print "false"
Cardchaind status 2>&1 | jq 

# before creating a validator, you need to fund your wallet and check balance
Cardchaind query bank balances $WALLET_ADDRESS 
Node Sync Status Checker
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.cardchaind/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://cardchain-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

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
Amount, ubpf
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
Cardchaind tx staking create-validator \
--amount 1000000ubpf \
--from $WALLET \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--pubkey $(Cardchaind tendermint show-validator) \
--moniker "test" \
--identity "" \
--website "" \
--details "I love blockchain ❤️" \
--chain-id cardtestnet-12 \
--gas auto --gas-adjustment 1.5 \
-y
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
sudo ufw allow ${CARDCHAIN_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop Cardchaind
sudo systemctl disable Cardchaind
sudo rm -rf /etc/systemd/system/Cardchaind.service
sudo rm $(which Cardchaind)
sudo rm -rf $HOME/.cardchaind
sed -i "/CARDCHAIN_/d" $HOME/.bash_profile
