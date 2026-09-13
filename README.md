# gno.land Mainnet (gnoland-1) Full Node Kurulum Rehberi
<img width="1398" height="728" alt="image" src="https://github.com/user-attachments/assets/31688bd2-d7e0-431c-913b-6c63f997185a" />


Bu rehber, gno.land mainnet'ini (chain-id `gnoland-1`, launch: 2026-09-12) sıfırdan bir full node olarak ayağa kaldırmak için hazırlanmıştır. Kendi sunucumda karşılaştığım ve çözdüğüm gerçek sorunlar (Go versiyon uyumsuzluğu, GNOROOT hatası, config dosya konumu, seed formatı) buraya dahil edildi, aynı hatalarla uğraşmamanız içindir.

> Mainnet **fresh bir zincir**, betanet'in hardfork'u değil. Validator seti genesis'te 4 kurucu organizasyonla (Gnocore, OnBloc, Samourai Crew, Berty) kapalı başladı, yeni validator eklenmesi GovDAO onayına bağlıdır. Bu rehber sadece **full node** kurulumunu kapsar.

---

## Donanım Gereksinimleri

| Bileşen | Minimum | Önerilen |
|---|---|---|
| İşletim Sistemi | Ubuntu 22.04+ | Ubuntu 24.04 |
| CPU | 4 çekirdek | 8 çekirdek |
| RAM | 8 GB | 16 GB |
| Disk | 200 GB SSD | 500 GB NVMe SSD |
| Ağ | 100 Mbps | 1 Gbps |

## Ağ Bilgileri

| Tür | Değer |
|---|---|
| Chain ID | `gnoland-1` |
| RPC | https://rpc.gno.land |
| Web | https://gno.land |
| Seed 1 | `g15rcv5yqef3kvnmueqvkyw8y05sd40jz9p3n5su@seed-1.gno.land:26656` |
| Seed 2 | `g1ck2yeyvvnpl92237gcea0z68jx07a4nnyvuaan@seed-2.gno.land:26656` |
| Resmi Release | https://github.com/gnolang/gno/releases/tag/chain/mainnet |
| Genesis SHA256 | `ea22691003130eae3ba975b7d16460706b5d75ce6c04ae82c0c4faeab7de91f0` |

⚠️ **Seed adresleri için `node_id@host:port` formatı zorunludur.** Sadece `seed-1.gno.land` ya da `seed-1.gno.land:26656` yazarsanız node hiçbir peer'a bağlanamaz ve "Blockpool has no peers" hatasıyla sonsuza kadar bekler. Bunu deneme yanılmayla bulduk, ID'ler yukarıda hazır.

---

## Adım 1: Sistem Güncellemesi ve Bağımlılıklar

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git wget htop tmux build-essential jq make lz4 gcc unzip \
  screen cmake perl automake autoconf libtool libssl-dev zstd pv
```

## Adım 2: Go Kurulumu (1.22+ ZORUNLU)

```bash
go version
```

Çıktı 1.22'nin altındaysa (örn. eski Ubuntu paketiyle gelen 1.21.x), mainnet binary'si derlenmez veya çalışmaz. Güncelleyin:

```bash
cd $HOME
VER="1.23.0"
wget "https://go.dev/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"

echo 'export PATH=/usr/local/go/bin:$HOME/go/bin:$PATH' >> ~/.bash_profile
source ~/.bash_profile
go version   # go1.23.0 görmelisiniz
```

## Adım 3: Değişkenleri Ayarla

Kendi bilgilerinizle güncelleyin:

```bash
echo 'export WALLET="wallet-adiniz"' >> $HOME/.bash_profile
echo 'export MONIKER="node-adiniz"' >> $HOME/.bash_profile
echo 'export GNOLAND_PORT="26"' >> $HOME/.bash_profile
source $HOME/.bash_profile
```

`GNOLAND_PORT` aynı sunucuda başka bir chain/testnet node'u çalıştırıyorsanız çakışmayı önler (örn. varsayılan 26 yerine 58 kullanın: RPC 58657, P2P 58656).

## Adım 4: Kaynağı Çek ve Binary Derle

```bash
cd $HOME
git clone https://github.com/gnolang/gno.git
cd gno && git checkout chain/mainnet
make -C gno.land install.gnoland install.gnokey
```

Doğrulayın:

```bash
gnoland version
gnokey version
```

⚠️ Aynı sunucuda başka bir gno chain'i (testnet) çalıştırıyorsanız, `make install` önceki binary'yi ezer. Önce onu farklı bir isimle yedekleyin:
```bash
cp $HOME/go/bin/gnoland $HOME/go/bin/gnoland-eski-chain-adi
```

## Adım 5: Node Yapılandırması

```bash
cd $HOME
gnoland config init --config-path $HOME/gnoland-data/config/config.toml
gnoland secrets init --data-dir $HOME/gnoland-data/secrets/
```

⚠️ **Config dosyasının yeri önemli.** `gnoland start` komutu config'i `<data-dir>/config/config.toml` yolunda arar. Yukarıdaki gibi doğrudan `config/` alt klasörüne yazdırın, aksi halde "unable to load config; config file does not exist" hatası alırsınız.

Ayarları uygulayın:

```bash
gnoland config set rpc.laddr tcp://0.0.0.0:${GNOLAND_PORT}657 --config-path $HOME/gnoland-data/config/config.toml
gnoland config set p2p.laddr tcp://0.0.0.0:${GNOLAND_PORT}656 --config-path $HOME/gnoland-data/config/config.toml
gnoland config set proxy_app tcp://127.0.0.1:${GNOLAND_PORT}658 --config-path $HOME/gnoland-data/config/config.toml
gnoland config set moniker "$MONIKER" --config-path $HOME/gnoland-data/config/config.toml
gnoland config set application.prune_strategy syncable --config-path $HOME/gnoland-data/config/config.toml
gnoland config set p2p.external_address "$(wget -qO- eth0.me):${GNOLAND_PORT}656" --config-path $HOME/gnoland-data/config/config.toml
gnoland config set p2p.seeds "g15rcv5yqef3kvnmueqvkyw8y05sd40jz9p3n5su@seed-1.gno.land:26656,g1ck2yeyvvnpl92237gcea0z68jx07a4nnyvuaan@seed-2.gno.land:26656" --config-path $HOME/gnoland-data/config/config.toml
gnoland config set p2p.persistent_peers "" --config-path $HOME/gnoland-data/config/config.toml
```

## Adım 6: Genesis Dosyasını İndir ve Doğrula

```bash
cd $HOME/gnoland-data/config
wget -O genesis.json.gz https://github.com/gnolang/gno/releases/download/chain%2Fmainnet/genesis.json.gz
gunzip genesis.json.gz
sha256sum genesis.json
```

Beklenen çıktı:
```
ea22691003130eae3ba975b7d16460706b5d75ce6c04ae82c0c4faeab7de91f0  genesis.json
```

⚠️ Hash eşleşmiyorsa **devam etmeyin**, dosyayı tekrar indirin. Genesis dosyası büyük (~56 MB sıkıştırılmış, açıldığında daha büyük) çünkü 3.26 milyon hesap bakiyesi içeriyor.

## Adım 7: Cüzdan Oluştur

Yeni cüzdan:
```bash
gnokey add $WALLET
```
⚠️ **KRİTİK:** Mnemonic phrase gösterilecek. Hemen güvenli, offline bir yere kaydedin.

Mevcut bir cüzdanı kurtarmak için:
```bash
gnokey add $WALLET --recover
```

## Adım 8: Systemd Servisi Oluştur

```bash
sudo tee /etc/systemd/system/gnoland.service > /dev/null <<EOF
[Unit]
Description=gno.land Mainnet Node
After=network-online.target

[Service]
User=$USER
WorkingDirectory=$HOME
Environment=HOME=$HOME
Environment=GNOROOT=$HOME/gno
ExecStart=$(which gnoland) start \\
  --chainid gnoland-1 \\
  --genesis $HOME/gnoland-data/config/genesis.json \\
  --data-dir $HOME/gnoland-data/ \\
  --skip-genesis-sig-verification
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable gnoland
```

⚠️ **`Environment=GNOROOT=$HOME/gno` satırını atlamayın.** Olmadan node "panic: gno was unable to determine GNOROOT" hatasıyla çöker. GnoVM standart kütüphane dosyalarını bu değişkenle bulur.

`--skip-genesis-sig-verification` flag'i mainnet için resmi olarak zorunludur (release notlarında belirtilmiştir), olmadan node genesis'i reddeder.

## Adım 9: Firewall

```bash
sudo ufw allow ${GNOLAND_PORT}656/tcp comment "gnoland P2P"
sudo ufw allow ${GNOLAND_PORT}657/tcp comment "gnoland RPC"
```

P2P portu (656) dışa açık olmalı, aksi halde diğer node'lar size bağlanamaz ve peer sayınız düşük kalır.

## Adım 10: Başlat ve Senkronizasyonu Doğrula

```bash
sudo systemctl restart gnoland
sudo journalctl -u gnoland -f
```

Not: İlk başlangıçta "InitChainer: standard libraries loaded" mesajından sonra birkaç dakika sessiz kalabilir. Bu normal, 3.26M hesaplık genesis state'i işleniyor. Bu sırada node'u **durdurmayın**, yarıda kesilirse (SIGTERM timeout → SIGKILL) süreç en baştan başlar.

Peer bağlantısını kontrol edin:
```bash
curl -s http://localhost:${GNOLAND_PORT}657/net_info | jq .result.n_peers
```
0'dan büyük bir sayı görmelisiniz (seed'ler doğru girildiyse genelde 10-15 civarı).

Senkronizasyon durumu:
```bash
curl -s http://localhost:${GNOLAND_PORT}657/status | jq .result.sync_info
```
`catching_up: false` olduğunda tam senkronize olmuşsunuz demektir. Mainnet henüz yeni olduğu için (launch: 12 Eylül 2026) senkronizasyon süresi kısa, birkaç saat içinde tamamlanır.

---

## Faydalı Komutlar

```bash
# Servis yönetimi
sudo systemctl status gnoland
sudo systemctl restart gnoland

# Canlı loglar
sudo journalctl -u gnoland -f --no-hostname -o cat

# Sync durumu
curl -s http://localhost:${GNOLAND_PORT}657/status | jq .result.sync_info

# Peer sayısı
curl -s http://localhost:${GNOLAND_PORT}657/net_info | jq .result.n_peers

# Node ID
gnoland secrets get node_id --data-dir $HOME/gnoland-data

# Cüzdan bakiyesi
gnokey query -remote "https://rpc.gno.land" auth/accounts/G1-ADRESINIZ
```

## Sık Karşılaşılan Hatalar: Özet Tablo

| Hata | Sebep | Çözüm |
|---|---|---|
| `unable to determine GNOROOT` | Servis dosyasında `GNOROOT` tanımsız | `Environment=GNOROOT=$HOME/gno` ekleyin |
| `config file ... does not exist` | config.toml yanlış klasörde | `<data-dir>/config/config.toml` yoluna taşıyın |
| `Blockpool has no peers` (sürekli) | Seed'ler `id@host:port` formatında değil | Yukarıdaki tam seed adreslerini kullanın |
| Genesis hash uyuşmuyor | Bozuk/eski indirme | Dosyayı tekrar indirin, resmi hash ile karşılaştırın |
| `make install` sonrası eski chain binary'si bozuldu | Aynı sunucuda birden fazla chain | Binary'leri derlemeden önce farklı isimle yedekleyin |

---

## Not: Validator Kaydı (Register) Adımı

Bu rehber şu an için **full node kurulumuna** kadar olan kısmı kapsıyor. `r/gnops/valopers` üzerinden `Register` çağrısı göndermek için gas fee'ye yetecek kadar `ugnot` gerekiyor. Mainnet'te transferler §126 kuralı gereği genesis'te kilitli olduğundan ve faucet bulunmadığından, elimizde henüz token yok.

Token temin edilip GovDAO süreciyle (Legal Review → Technical Review → onay) mainnet validator seçimimiz netleştiğinde, validator oluşturma ve `Register` adımları bu rehbere ayrı bir bölüm olarak eklenecek.

---

**Bu rehber ZeycaNode tarafından, gno.land Test13→Topaz→Sapphire→Pearl→Mainnet süreçlerinde edinilen operasyonel tecrübeyle hazırlanmıştır.**
