# Pryzm-Node
Node Tutorial


# Manual Installation

Mari kita menjelajahi proses instalasi manual saat kita menyiapkan sebuah node dengan Cosmovisor!

## Install Dependencies

Update paket sistem dan instal peralatan build
```bash
sudo apt -q update
sudo apt -qy install curl git jq lz4 build-essential fail2ban ufw
sudo apt -qy upgrade
```
# Secure Setup
Langkah ini bersifat opsional, dan Anda dapat memilih untuk melanjutkan untuk mengonfigurasi moniker.

Atur ssh untuk pengguna pryzm, gantilah YOUR_PUBLIC_SSH_KEY dengan milik Anda sendiri!

```bash
sudo adduser pryzm --disabled-password -q
sudo usermod -aG sudo pryzm
sudo -u pryzm bash -c 'mkdir -p ~/.ssh && echo "YOUR_PUBLIC_SSH_KEY" >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys'
```


# Install Go
Instal versi go 1.20.3
```bash
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.20.3.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
```

# Download Binaries
Unduh pryzm binaries yang sudah dikompilasi
```bash
cd $HOME
wget https://storage.googleapis.com/pryzm-zone/core/0.11.1/pryzmd-0.11.1-linux-amd64
sudo mv pryzmd-0.11.1-linux-amd64 pryzmd
sudo chmod +x pryzmd
```

# Persiapkan binaries untuk cosmovisor

```
mkdir -p $HOME/.pryzm/cosmovisor/genesis/bin
mv pryzmd $HOME/.pryzm/cosmovisor/genesis/bin/

```

# Buat symlink

```
sudo ln -s $HOME/.pryzm/cosmovisor/genesis $HOME/.pryzm/cosmovisor/current -f
sudo ln -s $HOME/.pryzm/cosmovisor/current/bin/pryzmd /usr/local/bin/pryzmd -f
```

# Cosmovisor Setup

```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0
```


# Create Service
```
sudo tee /etc/systemd/system/pryzm.service > /dev/null << EOF
[Unit]
Description=pryzm node service
After=network-online.target
 
[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.pryzm"
Environment="DAEMON_NAME=pryzmd"
Environment="UNSAFE_SKIP_BACKUP=true"
 
[Install]
WantedBy=multi-user.target
EOF
```

# Enable Service


```
sudo systemctl daemon-reload
sudo systemctl enable pryzm


```


# Initialize Node

```
pryzmd config chain-id indigo-1
pryzmd config keyring-backend test
pryzmd config node tcp://localhost:23257
pryzmd init $MONIKER --chain-id indigo-1

```

# Download Genesis & Addrbook
```
curl -Ls https://31.220.72.81/pryzm-testnet/genesis.json > $HOME/.pryzm/config/genesis.json
curl -Ls https://31.220.72.81/pryzm-testnet/addrbook.json > $HOME/.pryzm/config/addrbook.json
```

# Configure Seeds & Configure Gas Prices
```
sed -i -e "s|^seeds *=.*|seeds = \"521ec79c5451b9e214044a95584352203bc9b8a6@31.220.72.81:23210\"|" $HOME/.pryzm/config/config.toml
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.015upryzm,0.01factory/pryzm15k9s9p0ar0cx27nayrgk6vmhyec3lj7vkry7rx/uusdsim\"|" $HOME/.pryzm/config/app.toml
```


# Pruning Setting
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.pryzm/config/app.toml

```
# Configure state sync
```
sudo systemctl stop pryzm
cp $HOME/.pryzm/data/priv_validator_state.json $HOME/.pryzm/priv_validator_state.json.backup
pryzmd tendermint unsafe-reset-all --keep-addr-book --home $HOME/.pryzm
```


# Configure state sync
```
STATE_SYNC_RPC=https://31.220.72.81:443
STATE_SYNC_PEER=521ec79c5451b9e214044a95584352203bc9b8a6@31.220.72.81:23256
LATEST_HEIGHT=$(curl -s $STATE_SYNC_RPC/block | jq -r .result.block.header.height)
SYNC_BLOCK_HEIGHT=$((LATEST_HEIGHT - 2000))
SYNC_BLOCK_HASH=$(curl -s "$STATE_SYNC_RPC/block?height=$SYNC_BLOCK_HEIGHT" | jq -r .result.block_id.hash)
 
echo $LATEST_HEIGHT $SYNC_BLOCK_HEIGHT $SYNC_BLOCK_HASH
 
sed -i \
  -e "s|^enable *=.*|enable = true|" \
  -e "s|^rpc_servers *=.*|rpc_servers = \"$STATE_SYNC_RPC,$STATE_SYNC_RPC\"|" \
  -e "s|^trust_height *=.*|trust_height = $SYNC_BLOCK_HEIGHT|" \
  -e "s|^trust_hash *=.*|trust_hash = \"$SYNC_BLOCK_HASH\"|" \
  -e "s|^persistent_peers *=.*|persistent_peers = \"$STATE_SYNC_PEER\"|" \
  $HOME/.pryzm/config/config.toml
```

# Backup state data
```
mv $HOME/.pryzm/priv_validator_state.json.backup $HOME/.pryzm/data/priv_validator_state.json
```

# Start Service
```
sudo systemctl start pryzm
```
# Install Pryzm Feeder
Untuk dapat menjalankan pryzm feeder, Anda memerlukan 2 alat, yaitu docker dan docker compose.

# Install docker

```
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl software-properties-common && curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null && sudo apt update && sudo apt-cache policy docker-ce && sudo apt install -y docker-ce

```
# Install docker compose

```
mkdir -p ~/.docker/cli-plugins/ && curl -SL https://github.com/docker/compose/releases/download/v2.3.3/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose && chmod +x ~/.docker/cli-plugins/docker-compose

```
# Create Pryzm Feeder Wallet
Tentukan alamat dompet feeder harga Pryzm Anda (pertimbangkan untuk membuat yang baru jika diinginkan)

# Generate new key
```
pryzmd keys add feeder

```
# Recover key
```
pryzmd keys add feeder --recover

```

# Request Faucet
Minta sejumlah dana dari Pryzm Faucet dengan alamat dompet feeder pryzm Anda


# Configure Pryzm Feeder
Unduh file feeder pryzm yang diperlukan
```
cd $HOME && mkdir -p $HOME/pryzmfeeder && cd $HOME/pryzmfeeder && wget https://storage.googleapis.com/pryzm-zone/feeder/config.yaml https://storage.googleapis.com/pryzm-zone/feeder/init.sql https://storage.googleapis.com/pryzm-zone/feeder/docker-compose.yml
docker pull europe-docker.pkg.dev/pryzm-zone/core/pryzm-feeder:0.3.4 timescale/timescaledb:2.13.0-pg16

```


# Configuration Setting
Edit tiga baris pada config.yml dengan mengisi detail spesifik Anda:
```
feeder: "your_pryzm_feeder_wallet_address"
feederMnemonic: "your_pryzm_feeder_mnemonic"
validator: "your_valoper_address"
```

# Configure PostgreSQL
Untuk dapat Menginisialisasi db untuk pryzm feeder Anda harus menginstal PostgreSQL

```
sudo apt install postgresql postgresql-contrib
sudo service postgresql start

```

# Enable PostgreSQL access
```
sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD 'postgres';" -c "\q" && psql -U postgres -h localhost -W -c "\i $HOME/pryzmfeeder/init.sql"

```


# Link Validator and Feeder

```
pryzmd tx oracle delegate-feed-consent [your_pryzm_feeder_wallet_address] --fees 2000factory/pryzm15k9s9p0ar0cx27nayrgk6vmhyec3lj7vkry7rx/uusdsim,3000upryzm --from wallet
```


# Start Pryzm Feeder

```
screen -S pryzmfeeder
docker run --name=pryzm-feeder --network host --restart=always -v "$(pwd)/config.yaml:/app/config.yaml" -v "$(pwd)/logs:/app/logs" europe-docker.pkg.dev/pryzm-zone/core/pryzm-feeder:0.3.4
```

# Let's f*cking goooo!
```
sudo systemctl start pryzm
```


# Managing keys


```
#Generate new key
pryzmd keys add wallet

#Recover key
pryzmd keys add wallet --recover

#List all key
pryzmd keys list

#Delete key
pryzmd keys delete wallet

#Export key
pryzmd keys export wallet

#Import key
pryzmd keys import wallet wallet.backup

#Query wallet balances
pryzmd q bank balances $(pryzmd keys show wallet -a)
```

# Managing validators

```
pryzmd tx staking create-validator \
--amount 1000000upryzm \
--pubkey $(pryzmd tendermint show-validator) \
--moniker "your-moniker-name" \
--identity "your-keybase-id" \
--details "your-details" \
--website "your-website" \
--security-contact "your-email" \
--chain-id indigo-1 \
--commission-rate 0.05 \
--commission-max-rate 0.20 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--from wallet \
--gas-adjustment 1.4 \
--gas auto \
--gas-prices 0.015upryzm \
-y
```


# Edit validator

```
pryzmd tx staking edit-validator \
--new-moniker "your-moniker-name" \
--identity "your-keybase-id" \
--details "your-details" \
--website "your-website" \
--security-contact "your-email" \
--chain-id indigo-1 \
--commission-rate 0.05 \
--from wallet \
--gas-adjustment 1.4 \
--gas auto \
--gas-prices 0.015upryzm \
-y
```

```
#Unjail validator
pryzmd tx slashing unjail --from wallet --chain-id indigo-1 --gas-adjustment 1.4 --gas auto --gas-prices 0.015upryzm -y

#Validator jail reason
pryzmd query slashing signing-info $(pryzmd tendermint show-validator)

#List active validator
pryzmd q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_BONDED")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl

#List incative validator
pryzmd q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_UNBONDED")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl

#View validator details
pryzmd q staking validator $(pryzmd keys show wallet --bech val -a)
```


# Remove node

```
cd $HOME
sudo systemctl stop pryzm
sudo systemctl disable pryzm
sudo rm /etc/systemd/system/pryzm.service
sudo systemctl daemon-reload
sudo rm -f $(which pryzmd)
sudo rm -rf $HOME/.pryzm
sudo rm -rf $HOME/go
```





