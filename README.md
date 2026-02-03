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
| **Sürüm**   | Ubuntu 24.04 LTS |

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

- `validatör-name-girin` yerine validatöre vermek istediğiniz ismi girin.

```bash
REPUBLIC_HOME="$HOME/.republicd"
republicd init validatör-name-girin --chain-id raitestnet_77701-1 --home "$REPUBLIC_HOME"
```

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
curl -s https://rpc.republicai.io/net_info | jq -r '.result.peers[] | .node_info.id + "@" + .remote_ip + ":" + (.node_info.listen_addr | split(":") | last)' | paste -sd "," -
PEERS=$(curl -s https://rpc.republicai.io/net_info | jq -r '.result.peers[] | .node_info.id + "@" + .remote_ip + ":" + (.node_info.listen_addr | split(":") | last)' | paste -sd "," -)
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" ~/.republicd/config/config.toml
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

## 12. Bakiye Kontrol:

- Bakiye geldikten sonra validatörü başlatmalısınız. Bakiyeyi aşağıdaki komut ile kontrol edebilirsiniz.

```bash
republicd query bank balances $(republicd keys show mykey -a)
```

---

## 13. Validator Oluşturma

- "moniker": "BURAYA-VALIDATOR-ISMINIZI-YAZIN" kısmını kendi validator isminizle değiştirin!

```bash
PUBKEY=$(jq -r '.pub_key.value' $HOME/.republicd/config/priv_validator_key.json)
cat > validator.json << EOF
{
  "pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"$PUBKEY"},
  "amount": "1000000000000000000000arai",
  "moniker": "BURAYA-VALIDATOR-ISMINIZI-YAZIN",
  "identity": "",
  "website": "",
  "security": "",
  "details": "",
  "commission-rate": "0.1",
  "commission-max-rate": "0.2",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
EOF

republicd tx staking create-validator validator.json \
--chain-id raitestnet_77701-1 \
--gas auto \
--gas-adjustment 1.5 \
--fees 350000000000000arai \
--from mykey \
-y
```

---

## 14. Validator Kontrol

```bash
republicd query staking validator $(republicd keys show mykey --bech val -a)
```

---

## ✅ Explorer Kontrol:

- https://explorer.republicai.io/validators adresine gidin.
- Cüzdan adresinizi aratıp üzerine tıklayın.
- Yanında “Active” ibaresi yer alıyorsa, validatörünüz başarıyla çalışıyor demektir.

![1 (2)](https://github.com/user-attachments/assets/10a1f6a0-11b2-457a-b0f8-1ee68dfe4571)

