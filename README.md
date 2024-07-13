# Mantra Chain
![image](https://github.com/user-attachments/assets/3d2c800a-38f7-4e53-a020-e811a7466065)

MANTRA is a Security First RWA Layer 1 Blockchain, capable of adherence to real world regulatory requirements.

## System Requirements
![Screenshot (98)](https://github.com/user-attachments/assets/6cf91a8d-5a7a-4be4-bf8f-98f5a780dcdf)

## Install dependecies
```console
# Install Packages
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y
```
```console
# Install GO
ver="1.21.6"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```
```console
# Install node
cd $HOME && mkdir -p go/bin/
sudo wget -O /usr/lib/libwasmvm.x86_64.so https://github.com/CosmWasm/wasmvm/releases/download/v1.3.1/libwasmvm.x86_64.so
wget https://github.com/MANTRA-Finance/public/raw/main/mantrachain-hongbai/mantrachaind-linux-amd64.zip
unzip mantrachaind-linux-amd64.zip
rm mantrachaind-linux-amd64.zip
mv mantrachaind $HOME/go/bin
```
Run
```console
mantrachaind version --long | grep -e commit -e version
```
result
```
version: 3.0.0
commit: ""
```
## Initiation
```console
mantrachaind init my_name --chain-id=mantra-hongbai-1
mantrachaind config chain-id mantra-hongbai-1
```
## Create/recover wallet
```console
mantrachaind keys add <walletname>
           OR
mantrachaind keys add <walletname> --recover
```
## Download genesis
```console
wget -L -O $HOME/.mantrachain/config/genesis.json "https://raw.githubusercontent.com/111STAVR111/props/main/Mantra/genesis.json"
```
run
```console
sha256sum $HOME/.mantrachain/config/genesis.json
```
result
```
e9ea8a48d43f9a666032b1df21f160b42c069a9b1b5bd71435ab307bd3304b0a
```
## Set up the minimum gas price and Peers/Seeds/Filter peers/MaxPeers
```console
sed -i.bak -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.0002uom\"/;" ~/.mantrachain/config/app.toml
external_address=$(wget -qO- eth0.me) 
sed -i.bak -e "s/^external_address *=.*/external_address = \"$external_address:26656\"/" $HOME/.mantrachain/config/config.toml
peers=""
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.mantrachain/config/config.toml
seeds=""
sed -i.bak -e "s/^seeds =.*/seeds = \"$seeds\"/" $HOME/.mantrachain/config/config.toml
sed -i 's/max_num_inbound_peers =.*/max_num_inbound_peers = 50/g' $HOME/.mantrachain/config/config.toml
sed -i 's/max_num_outbound_peers =.*/max_num_outbound_peers = 50/g' $HOME/.mantrachain/config/config.toml
```
## Pruning
```console
pruning="custom"
pruning_keep_recent="1000"
pruning_keep_every="0"
pruning_interval="10"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.mantrachain/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.mantrachain/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.mantrachain/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.mantrachain/config/app.toml
```
## Indexer
```console
indexer="null" &&
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.mantrachain/config/config.toml
```
## Download addr book
```console
wget -O $HOME/.mantrachain/config/addrbook.json "https://raw.githubusercontent.com/111STAVR111/props/main/Mantra/addrbook.json"
```
## Create a service file
```console
sudo tee /etc/systemd/system/mantrachaind.service > /dev/null <<EOF
[Unit]
Description=mantrachaind
After=network-online.target

[Service]
User=$USER
ExecStart=$(which mantrachaind) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```
## Using stateSync to get latest block height
```console
SNAP_RPC=https://mantra.rpc.t.stavr.tech:443
peers="b6943ba9d189c545d92051250d2a3641f2216b2b@mantra-t.seed.stavr.tech:36056"
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.mantrachain/config/config.toml
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
BLOCK_HEIGHT=$((LATEST_HEIGHT - 1000)); \
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)

echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH

sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"| ; \
s|^(seeds[[:space:]]+=[[:space:]]+).*$|\1\"\"|" $HOME/.mantrachain/config/config.toml
mantrachaind tendermint unsafe-reset-all --home $HOME/.mantrachain
curl -o - -L https://mantra-t.wasm.stavr.tech/wasm-mantra.tar.lz4 | lz4 -c -d - | tar -x -C $HOME/.mantrachain --strip-components 2
wget -O $HOME/.mantrachain/config/addrbook.json "https://raw.githubusercontent.com/111STAVR111/props/main/Mantra/addrbook.json"
sudo systemctl restart mantrachaind && journalctl -fu mantrachaind -o cat
```
## Download snapshot (if you don't want to use stateSync)
```console
cd $HOME
apt install lz4
sudo systemctl stop mantrachaind
cp $HOME/.mantrachain/data/priv_validator_state.json $HOME/.mantrachain/priv_validator_state.json.backup
rm -rf $HOME/.mantrachain/data
curl -o - -L https://mantra-t.snapshot.stavr.tech/mantra-snap.tar.lz4 | lz4 -c -d - | tar -x -C $HOME/.mantrachain --strip-components 2
curl -o - -L https://mantra-t.wasm.stavr.tech/wasm-mantra.tar.lz4 | lz4 -c -d - | tar -x -C $HOME/.mantrachain --strip-components 2
mv $HOME/.mantrachain/priv_validator_state.json.backup $HOME/.mantrachain/data/priv_validator_state.json
wget -O $HOME/.mantrachain/config/addrbook.json "https://raw.githubusercontent.com/111STAVR111/props/main/Mantra/addrbook.json"
```
## Start node
```console
sudo systemctl daemon-reload
sudo systemctl enable mantrachaind
sudo systemctl restart mantrachaind && sudo journalctl -fu mantrachaind -o cat
```
## Validator
```console
mantrachaind tx staking create-validator \
--commission-rate 0.1 \
--commission-max-rate 1 \
--commission-max-change-rate 1 \
--min-self-delegation "1" \
--amount 1000000uom \
--pubkey $(mantrachaind tendermint show-validator) \
--from <wallet> \
--moniker="my_moniker" \
--chain-id mantra-hongbai-1 \
--fees 35uom \
--gas 350000 \
--identity="" \
--website="" \
--details="" -y
```
## Delete node
```console
sudo systemctl stop mantrachaind
sudo systemctl disable mantrachaind
rm /etc/systemd/system/mantrachaind.service
sudo systemctl daemon-reload
cd $HOME
rm -rf .mantrachain
rm -rf $(which mantrachaind)
```

# Commands
## Check node logs
```console
sudo journalctl -fu mantrachaind  -o cat
```
## Check service status
```console
sudo systemctl status mantrachaind
```
## Restart service
```console
sudo systemctl restart mantrachaind
```
## Stop service
```console
sudo systemctl stop mantrachaind
```
## Start service
```console
sudo systemctl start mantrachaind
```
