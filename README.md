
# **Lava-Network-Node-Guide**


![alt text](https://i.hizliresim.com/e5qbrvl.png)


**Minimum Sistem Gereksinimleri**
```
- 4 CPU
- 8GB RAM
- 160GB DISK
- Ubuntu 20.04+

```


- **Paket yüklemeleri**

```
sudo apt update
sudo apt install -y unzip logrotate git jq sed wget curl coreutils systemd
```

- **Go Kurulumu**

```
sudo rm -rvf /usr/local/go/
wget https://golang.org/dl/go1.19.3.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.19.3.linux-amd64.tar.gz
rm go1.19.3.linux-amd64.tar.gz
echo "export PATH=\$PATH:/usr/local/go/bin" >>~/.profile
echo "export PATH=\$PATH:\$(go env GOPATH)/bin" >>~/.profile
source ~/.profile
```


- **Go Versiyon Kontrol**

```
go version

  Note: Go verison 1.19.3 vermeli
```


- **Testnet-1 Kurulumu**

```
git clone https://github.com/lavanet/lava-config.git
cd lava-config/testnet-1
source setup_config/setup_config.sh
echo "Lava config file path: $lava_config_folder"
mkdir -p $lavad_home_folder
mkdir -p $lava_config_folder
cp default_lavad_config_files/* $lava_config_folder
cp genesis_json/genesis.json $lava_config_folder/genesis.json
```

- **Lava Testnet Katılımı ve Cosmovisor Kurulumu**

```
go install github.com/cosmos/cosmos-sdk/cosmovisor/cmd/cosmovisor@v1.0.0
mkdir -p $lavad_home_folder/cosmovisor
wget https://lava-binary-upgrades.s3.amazonaws.com/testnet/cosmovisor-upgrades/cosmovisor-upgrades.zip
unzip cosmovisor-upgrades.zip
cp -r cosmovisor-upgrades/* $lavad_home_folder/cosmovisor
echo "# Setup Cosmovisor" >> ~/.profile
echo "export DAEMON_NAME=lavad" >> ~/.profile
echo "export CHAIN_ID=lava-testnet-1" >> ~/.profile
echo "export DAEMON_HOME=$HOME/.lava" >> ~/.profile
echo "export DAEMON_ALLOW_DOWNLOAD_BINARIES=true" >> ~/.profile
echo "export DAEMON_LOG_BUFFER_SIZE=512" >> ~/.profile
echo "export DAEMON_RESTART_AFTER_UPGRADE=true" >> ~/.profile
echo "export UNSAFE_SKIP_BACKUP=true" >> ~/.profile
source ~/.profile
```
```
$lavad_home_folder/cosmovisor/genesis/bin/lavad init \
my-node \
--chain-id lava-testnet-1 \
--home $lavad_home_folder \
--overwrite
cp genesis_json/genesis.json $lava_config_folder/genesis.json
```
- **Cosmovisor Kontrol**
```
cosmovisor version

Note: Lütfen cosmovisor'ın bir hata atacağını unutmayın Sorun değil. Aşağıdaki hata atılacak, lstat /home/ubuntu/.lava/cosmovisor/current/upgrade-info.json böyle bir dosya veya dizini yok
```

- **systemd birim dosyasını oluşturun**

```
echo "[Unit]
Description=Cosmovisor daemon
After=network-online.target
[Service]
Environment="DAEMON_NAME=lavad"
Environment="DAEMON_HOME=${HOME}/.lava"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
Environment="DAEMON_LOG_BUFFER_SIZE=512"
Environment="UNSAFE_SKIP_BACKUP=true"
User=$USER
ExecStart=${HOME}/go/bin/cosmovisor start --home=$lavad_home_folder --p2p.seeds $seed_node
Restart=always
RestartSec=3
LimitNOFILE=infinity
LimitNPROC=infinity
[Install]
WantedBy=multi-user.target
" >cosmovisor.service
sudo mv cosmovisor.service /lib/systemd/system/cosmovisor.service
```


-**Cosmovisor hizmetini önyüklemede çalışacak şekilde yapılandırın ve başlatın**

```
sudo systemctl daemon-reload
sudo systemctl enable cosmovisor.service
sudo systemctl restart systemd-journald
sudo systemctl start cosmovisor
```


-**Cosmovisor Kontrol**

Hizmetin durumunu kontrol edin
```
sudo systemctl status cosmovisor

```

Hizmet günlüklerini görüntülemek için - (çıkmak için CTRL+C tuşlarına basın)
```
sudo journalctl -u cosmovisor -f
```

Düğüm Durumu ve Senkronizasyon Kontrolü için
```
$HOME/.lava/cosmovisor/current/bin/lavad status | jq
```
#Senkronizasyon için latest_block_height değeri son blockta ve catching_up false olmalı



- **Cüzdan Oluşturma**

```
current_lavad_binary="$HOME/.lava/cosmovisor/current/bin/lavad"
ACCOUNT_NAME="name_here"
$current_lavad_binary keys add $ACCOUNT_NAME
```
#Mnemonic'i not ettiğinizden emin olun, çünkü onsuz cüzdanı kurtaramazsınız.!



- **Faucetten Token İsteme**
```
Lava Discord https://discord.gg/lavanetxyz
```
#Lava Discord Faucet kanalından $request <cüzdan-adresiniz> ile token isteyin



- **Staking gerçekleştirmek için hesabınızda para olduğunu doğrulayın**

```
YOUR_ADDRESS=$($current_lavad_binary keys show -a $ACCOUNT_NAME)
$current_lavad_binary query \
    bank balances \
    $YOUR_ADDRESS \
    --denom ulava
```


- **Düğümünüzün eşitlemeyi bitirdiğini ve ağa yakalandığını doğrulayın**
- 
```
$current_lavad_binary status | jq .SyncInfo.catching_up
```
#catching_up false olana kadar bekleyin!



- **Validator oluşturma**

#<<moniker_node>> Doğrulayıcınız için seçtiğiniz, insanların okuyabileceği bir adla değiştirin.
```
$current_lavad_binary tx staking create-validator \
    --amount="50000000ulava" \
    --pubkey=$($current_lavad_binary tendermint show-validator --home "$HOME/.lava/") \
    --moniker="<<moniker_node>>" \
    --chain-id=lava-testnet-1 \
    --commission-rate="0.10" \
    --commission-max-rate="0.20" \
    --commission-max-change-rate="0.01" \
    --min-self-delegation="10000" \
    --gas="auto" \
    --gas-adjustment "1.5" \
    --gas-prices="0.05ulava" \
    --home="$HOME/.lava/" \
    --from=$ACCOUNT_NAME
```
#Yukarıdaki komutu çalıştırmayı bitirdiğinizde code: 0, çıktıda görürseniz, komut başarılı olmuştur.




