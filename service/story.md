# 🚀 Story Validator Installation Guide

**Story Testnet** (Aeneid).

---

## 🛠️ System Requirements

- OS: Ubuntu 20.04 / 22.04
- RAM: 8GB+
- CPU: 4 cores+
- Storage: 200GB SSD

---

## ⚙️ Install Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```
---
## 🧩 Install Go
```
cd $HOME
VER=\"1.22.5\"
wget \"https://golang.org/dl/go$VER.linux-amd64.tar.gz\"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf \"go$VER.linux-amd64.tar.gz\"
rm \"go$VER.linux-amd64.tar.gz\"
echo \"export PATH=$PATH:/usr/local/go/bin:~/go/bin\" >> ~/.bash_profile
source $HOME/.bash_profile
```
---
## 🔧 Setup Environment Variables
```
echo \"export MONIKER=\\\"YourNodeName\\\"\" >> $HOME/.bash_profile
echo \"export STORY_CHAIN_ID=\\\"aeneid\\\"\" >> $HOME/.bash_profile
echo \"export STORY_PORT=\\\"52\\\"\" >> $HOME/.bash_profile
source $HOME/.bash_profile
```
---
## 📦 Download & Build Binaries
```
cd $HOME
git clone https://github.com/piplabs/story-geth.git
cd story-geth
git checkout v1.0.2
make geth
mv build/bin/geth  $HOME/go/bin/
```
```
cd $HOME
git clone https://github.com/piplabs/story
cd story
git checkout v1.3.0
go build -o story ./client
mv $HOME/story/story $HOME/go/bin/
```
---
### 🧱 Init Node
```
story init --moniker $MONIKER --network $STORY_CHAIN_ID
```

# set seeds and peers
```
SEEDS="46b7995b0b77515380000b7601e6fc21f783e16f@story-testnet-seed.itrocket.net:52656"
PEERS="01f8a2148a94f0267af919d2eab78452c90d9864@story-testnet-peer.itrocket.net:52656,381b10bd04853b375f9a1d42983a0fbfa753e4aa@37.59.22.162:26656,b66b0df0720b38f9405f8a5c50511365ec621b2a@135.181.181.59:62656,7160dec63da82b56e1ce59a93c057c05e361cf85@135.181.117.37:64656,db6791a8e35dee076de75aebae3c89df8bba3374@65.109.50.22:56656,a8d01e154197d799637eca4f0f369dc215db6b70@144.76.111.9:26656,9d34ab3819aa8baa75589f99138318acfa0045f5@95.217.119.251:30900,5eb1d392045c1159a94bdb49916341ac4a787500@157.180.93.155:52656,34c910cde040e983f076195175ed2fe7d447b486@152.53.102.226:26656,dfb96be7e47cd76762c1dd45a5f76e536be47faa@65.108.45.34:32655,ae49103a54f77effa438978ad8a7ba09b6f20da0@144.76.202.120:35656,311cd3903e25ab85e5a26c44510fbc747ab61760@152.53.87.97:36656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.story/story/config/config.toml
```
---
# set custom ports in story.toml file
```
sed -i.bak -e "s%:1317%:${STORY_PORT}317%g;
s%:8551%:${STORY_PORT}551%g" $HOME/.story/story/config/story.toml
```
---
# set custom ports in config.toml file
```
sed -i.bak -e "s%:26658%:${STORY_PORT}658%g;
s%:26657%:${STORY_PORT}657%g;
s%:26656%:${STORY_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${STORY_PORT}656\"%;
s%:26660%:${STORY_PORT}660%g" $HOME/.story/story/config/config.toml
```
---
# enable prometheus and disable indexing
```
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.story/story/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.story/story/config/config.toml
```
---
## 🧰 Systemd Services
```
sudo tee /etc/systemd/system/story-geth.service > /dev/null <<EOF
[Unit]
Description=Story Geth daemon
After=network-online.target

[Service]
User=$USER
ExecStart=$HOME/go/bin/geth --aeneid --syncmode full --http --http.api eth,net,web3,engine --http.vhosts '*' --http.addr 0.0.0.0 --http.port ${STORY_PORT}545 --authrpc.port ${STORY_PORT}551 --ws --ws.api eth,web3,net,txpool --ws.addr 0.0.0.0 --ws.port ${STORY_PORT}546
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```
```
sudo tee /etc/systemd/system/story.service > /dev/null <<EOF
[Unit]
Description=Story Service
After=network.target

[Service]
User=$USER
WorkingDirectory=$HOME/.story/story
ExecStart=$(which story) run

Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```
---
# 💾 Snapshot Restore
**backup priv_validator_state.json**
```
cp $HOME/.story/story/data/priv_validator_state.json $HOME/.story/story/priv_validator_state.json.backup
```
**remove old data and unpack Story snapshot**
```
rm -rf $HOME/.story/story/data
curl https://server-3.itrocket.net/testnet/story/story_2025-07-02_6198074_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.story/story
```
**restore priv_validator_state.json**
```
mv $HOME/.story/story/priv_validator_state.json.backup $HOME/.story/story/data/priv_validator_state.json
```
**delete geth data and unpack Geth snapshot**
```
rm -rf $HOME/.story/geth/aeneid/geth/chaindata
mkdir -p $HOME/.story/geth/aeneid/geth
curl https://server-3.itrocket.net/testnet/story/geth_story_2025-07-02_6198074_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.story/geth/aeneid/geth
```
---
## enable and start geth, story
```
sudo systemctl daemon-reload
sudo systemctl enable story story-geth
sudo systemctl restart story-geth && sleep 5 && sudo systemctl restart story
```
---
# check logs
```
journalctl -u story -u story-geth -f -o cat
```
