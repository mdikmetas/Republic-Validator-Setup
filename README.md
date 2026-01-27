# Republic Testnet Node & Validator Kurulum Rehberi

Bu rehber, **Republic AI Testnet** node’unu sıfırdan kurmak ve validator olmak isteyenler için hazırlanmıştır.
Aşağıdaki adımları sırasıyla uygulamanız yeterlidir.

---

## Sistem Gereksinimleri

| Bileşen  | Gereksinim       |
| -------- | ---------------- |
| **CPU**  | En az 4 çekirdek |
| **RAM**  | Minimum 16 GB    |
| **Disk** | En az 500 GB SSD |
| **Sürüm**   | Ubuntu 22.04 LTS |

---

## 1. Sunucuya Bağlanma

```bash
ssh root@SUNUCU_IP
```

---

## 2. Sistem Güncelleme ve Gerekli Paketler

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl jq git build-essential
```

---

## 3. republicd Binary İndirme

```bash
VERSION="v0.1.0"
curl -L "https://media.githubusercontent.com/media/RepublicAI/networks/main/testnet/releases/${VERSION}/republicd-linux-amd64" -o /usr/local/bin/republicd
chmod +x /usr/local/bin/republicd
```

Kurulum kontrolü:

```bash
republicd version
```

---

## 4. Node Başlatma (Init)

```bash
REPUBLIC_HOME="$HOME/.republicd"
republicd init validatör-name --chain-id raitestnet_77701-1 --home "$REPUBLIC_HOME"
```

- `validatör-name` yerine istediğiniz moniker validatör yazın.

---

## 5. Genesis Dosyasını İndirme

```bash
curl -s https://raw.githubusercontent.com/RepublicAI/networks/main/testnet/genesis.json > "$REPUBLIC_HOME/config/genesis.json"
```

---

## 6. State Sync Ayarlama (Hızlı Senkronizasyon)

```bash
SNAP_RPC="https://statesync.republicai.io"
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height)
BLOCK_HEIGHT=$((LATEST_HEIGHT - 1000))
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)

sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" "$REPUBLIC_HOME/config/config.toml"
```

---

## 7. Peer Ekleme

```bash
PEERS="e281dc6e4ebf5e32fb7e6c4a111c06f02a1d4d62@3.92.139.74:26656,cfb2cb90a241f7e1c076a43954f0ee6d42794d04@54.173.6.183:26656,dc254b98cebd6383ed8cf2e766557e3d240100a9@54.227.57.160:26656"
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" "$REPUBLIC_HOME/config/config.toml"
```

---

## 8. Systemd Servis Kurulumu

```bash
sudo tee /etc/systemd/system/republicd.service > /dev/null <<EOF
[Unit]
Description=Republic Protocol Node
After=network-online.target

[Service]
User=root
WorkingDirectory=/root
ExecStart=/usr/local/bin/republicd start --home /root/.republicd --chain-id raitestnet_77701-1
Restart=always
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

Servisi aktif et:

```bash
sudo systemctl daemon-reload
sudo systemctl enable republicd
sudo systemctl start republicd
```

Logları izlemek için:

```bash
journalctl -u republicd -f
```

Örnek Çıktı:

<img width="1657" height="232" alt="image" src="https://github.com/user-attachments/assets/40f16c29-01d5-4882-9894-1a9173e133d5" />

---

## 9. Senkronizasyon Kontrolü

```bash
republicd status | jq '.sync_info'
```

Aşağıdaki çıktıyı görmelisiniz:

<img width="768" height="198" alt="image" src="https://github.com/user-attachments/assets/6e219da7-b583-48a3-b75f-0b4b19ce987f" />

- Bu değer `false` olduğunda node tamamen senkron olmuştur. Aşağıdaki örnek çıktıyı kontrol edin.

---

## 10. Key Oluşturma

```bash
republicd keys add mykey
```

- Seed (mnemonic) kelimelerini mutlaka güvenli bir yere kaydedin.
- Adressi kopyala ve token talep etmek için bir yere kaydedin.

---

## 11. Faucet (Test Token Alma)

✅ Test token almak için Discord sunucusuna katılın: [https://discord.gg/kNERz4xC](https://discord.gg/kNERz4xC)

- Faucet için sadece **adresinizi** paylaşın.
- **Seed (mnemonic) asla paylaşılmaz.**

---

## 12. Validator Oluşturma

```bash
republicd tx staking create-validator \
  --amount=1000000000000000000000arai \
  --pubkey=$(republicd comet show-validator) \
  --moniker="validatör-name" \
  --chain-id=raitestnet_77701-1 \
  --commission-rate="0.10" \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1" \
  --gas=auto \
  --gas-adjustment=1.5 \
  --gas-prices="250000000arai" \
  --from=mykey
```

- `validatör-name` kısmını kendi validator adınız ile değiştirin.
- `mykey` kısmı oluşturduğunuz keyi girin.

---

## 13. Validator Kontrol

```bash
republicd query staking validator $(republicd keys show mykey --bech val -a)
```

---

## ✅ Ek Notlar

* Node tamamen senkron olmadan validator oluşturmayın.
* Seed kelimelerinizi kimseyle paylaşmayın ve güvenli bir yerde saklayın.

