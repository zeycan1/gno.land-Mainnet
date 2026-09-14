# gno.land Mainnet (gnoland-1) Full Node Kurulum Rehberi
<img width="1398" height="728" alt="image" src="https://github.com/user-attachments/assets/31688bd2-d7e0-431c-913b-6c63f997185a" />

Bu rehber, gno.land mainnet'ini (chain-id `gnoland-1`, launch: 2026-09-12) sıfırdan bir full node olarak ayağa kaldırmak için hazırlanmıştır. Buradaki her adım, kendi sunucumuzda (ZeycaNode) gerçekten uyguladığımız ve doğrulanmış komutlardır, kağıt üzerinde bir tahmin değildir. Karşılaştığımız ve çözdüğümüz gerçek hatalar (Go versiyon uyumsuzluğu, GNOROOT hatası, config dosya konumu, seed formatı) da dahil edildi, aynı hatalarla uğraşmamanız içindir.

> Mainnet fresh bir zincir, betanet'in hardfork'u değil. Validator seti genesis'te 4 kurucu organizasyonla (Gnocore, OnBloc, Samourai Crew, Berty) kapalı başladı, yeni validator eklenmesi GovDAO onayına bağlıdır. Bu rehber full node kurulumunu ve isteğe bağlı olarak RPC'nizi public yapmayı kapsar.

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
| Resmi RPC | https://rpc.gno.land |
| Web | https://gno.land |
| Seed 1 | `g15rcv5yqef3kvnmueqvkyw8y05sd40jz9p3n5su@seed-1.gno.land:26656` |
| Seed 2 | `g1ck2yeyvvnpl92237gcea0z68jx07a4nnyvuaan@seed-2.gno.land:26656` |
| Resmi Release | https://github.com/gnolang/gno/releases/tag/chain/mainnet |
| Genesis SHA256 | `ea22691003130eae3ba975b7d16460706b5d75ce6c04ae82c0c4faeab7de91f0` |

⚠️ **Seed adresleri için `node_id@host:port` formatı zorunludur.** Sadece `seed-1.gno.land` ya da port'suz bir yazım node'un hiçbir peer'a bağlanamamasına ve "Blockpool has no peers" hatasıyla sonsuza kadar beklemesine sebep olur. Bunu kendi kurulumumuzda deneme yanılmayla bulduk, doğru ID'ler ve portlar yukarıda hazır.

Bonus: ZeycaNode'un kendi çalışan mainnet node'una şu adreslerden read-only erişebilirsiniz (kendi node'unuz senkronize olurken test amaçlı kullanabilirsiniz):
- RPC: https://gnoland-mainnet-rpc.zeycanode.com
- API: https://gnoland-mainnet-api.zeycanode.com

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

Çıktı 1.22'nin altındaysa (örn. eski Ubuntu paketiyle gelen 1.21.x, bizim de başımıza gelen buydu), mainnet binary'si derlenmez veya çalışmaz. Güncelleyin:

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

Kendi bilgilerinizle güncelleyin. `NODE_PORT` özellikle aynı sunucuda başka bir chain/testnet node'u çalıştırıyorsanız önemlidir, çakışmayı önler:

```bash
echo 'export WALLET="wallet-adiniz"' >> $HOME/.bash_profile
echo 'export MONIKER="node-adiniz"' >> $HOME/.bash_profile
echo 'export NODE_PORT="58"' >> $HOME/.bash_profile
source $HOME/.bash_profile
```

`NODE_PORT=58` ile RPC 58657, P2P 58656 olur. Sunucunuzda başka bir chain çalışmıyorsa varsayılan `26` da kullanabilirsiniz (RPC 26657, P2P 26656).

## Adım 4: Kaynağı Çek ve Mainnet Tag'ine Geç

Kaynağı ayrı bir klasöre klonluyoruz (`gno-mainnet`), böylece başka bir chain'in kaynak/build klasörüyle karışmaz:

```bash
cd $HOME
git clone https://github.com/gnolang/gno.git gno-mainnet
cd gno-mainnet && git checkout chain/mainnet
```

## Adım 5: Binary'yi İzole Derle

`make install` sistemdeki önceki `gnoland`/`gnokey` binary'lerini ezer. Bu yüzden mainnet binary'lerini kendine özgü bir isimle (`gnoland-mainnet`, `gnokey-mainnet`) sakla, aynı sunucuda başka bir chain çalışıyorsa da çakışmaz:

```bash
cd $HOME/gno-mainnet
make -C gno.land build
cp gno.land/build/gnoland $HOME/go/bin/gnoland-mainnet
cp gno.land/build/gnokey  $HOME/go/bin/gnokey-mainnet
export PATH="$HOME/go/bin:$PATH"
```

Doğrulayın:

```bash
gnoland-mainnet version
gnokey-mainnet version
```

## Adım 6: Node Yapılandırmasını Oluştur

Veriyi kaynak klasöründen ayrı, kendine özgü bir dizinde (`mainnet-data`) tutuyoruz:

```bash
cd $HOME
gnoland-mainnet config init --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet secrets init --data-dir $HOME/mainnet-data/secrets/
```

⚠️ **Config dosyasının nihai yeri önemli.** `gnoland-mainnet start` komutu config'i `<data-dir>/config/config.toml` yolunda arar, `config init`'in yazdığı ilk yer ise `<data-dir>/config.toml` (bir üst klasör). Genesis dosyasını indirdiğimiz Adım 7'de bu dosyayı doğru klasöre taşıyacağız, aksi halde "unable to load config; config file does not exist" hatası alırsınız.

Ayarları uygulayın:

```bash
gnoland-mainnet config set rpc.laddr tcp://0.0.0.0:${NODE_PORT}657 --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet config set p2p.laddr tcp://0.0.0.0:${NODE_PORT}656 --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet config set proxy_app tcp://127.0.0.1:${NODE_PORT}658 --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet config set moniker "$MONIKER" --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet config set application.prune_strategy syncable --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet config set p2p.external_address "$(wget -qO- eth0.me):${NODE_PORT}656" --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet config set p2p.seeds "g15rcv5yqef3kvnmueqvkyw8y05sd40jz9p3n5su@seed-1.gno.land:26656,g1ck2yeyvvnpl92237gcea0z68jx07a4nnyvuaan@seed-2.gno.land:26656" --config-path $HOME/mainnet-data/config.toml
gnoland-mainnet config set p2p.persistent_peers "" --config-path $HOME/mainnet-data/config.toml
```

## Adım 7: Genesis Dosyasını İndir, Doğrula ve Config'i Taşı

```bash
mkdir -p $HOME/mainnet-data/config
cd $HOME/mainnet-data/config
wget -O genesis.json.gz https://github.com/gnolang/gno/releases/download/chain%2Fmainnet/genesis.json.gz
gunzip genesis.json.gz
sha256sum genesis.json
```

Beklenen çıktı:
```
ea22691003130eae3ba975b7d16460706b5d75ce6c04ae82c0c4faeab7de91f0  genesis.json
```

⚠️ Hash eşleşmiyorsa devam etmeyin, dosyayı tekrar indirin. Genesis dosyası büyük (~56 MB sıkıştırılmış) çünkü 3.26 milyon hesap bakiyesi içeriyor.

Şimdi Adım 6'da oluşturduğumuz config.toml'u genesis ile aynı klasöre taşıyın:

```bash
mv $HOME/mainnet-data/config.toml $HOME/mainnet-data/config/config.toml
ls $HOME/mainnet-data/config/   # config.toml ve genesis.json ikisi de burada olmalı
```

## Adım 8: Cüzdan Oluştur

Yeni cüzdan:
```bash
gnokey-mainnet add $WALLET --home $HOME/mainnet-data
```
⚠️ **KRİTİK:** Mnemonic phrase gösterilecek. Hemen güvenli, offline bir yere kaydedin.

Mevcut bir cüzdanı (örn. testnet'lerde kullandığınız) kurtarmak için:
```bash
gnokey-mainnet add $WALLET --recover --home $HOME/mainnet-data
```

Doğrulayın:
```bash
gnokey-mainnet list --home $HOME/mainnet-data
```

## Adım 9: Systemd Servisi Oluştur

```bash
sudo tee /etc/systemd/system/gno-mainnet.service > /dev/null <<SERVICEEOF
[Unit]
Description=gno.land Mainnet Node
After=network-online.target

[Service]
User=$USER
WorkingDirectory=$HOME
Environment=HOME=$HOME
ExecStart=$HOME/go/bin/gnoland-mainnet start \
  --chainid gnoland-1 \
  --genesis $HOME/mainnet-data/config/genesis.json \
  --data-dir $HOME/mainnet-data/ \
  --skip-genesis-sig-verification
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
SERVICEEOF

sudo systemctl daemon-reload
sudo systemctl enable gno-mainnet
```

Not: `GNOROOT` ortam değişkenini burada tanımlamıyoruz çünkü `gnoland-mainnet` binary'si zaten `$HOME/gno-mainnet` kaynak klasörünün yanında derlendi ve GnoVM standart kütüphaneyi otomatik bulabiliyor. Eğer binary'yi kaynak klasörden farklı bir konumda çalıştırıyorsanız veya "panic: gno was unable to determine GNOROOT" hatası alırsanız, `[Service]` bölümüne şu satırı ekleyin:
```
Environment=GNOROOT=$HOME/gno-mainnet
```

`--skip-genesis-sig-verification` flag'i mainnet için resmi olarak zorunludur (release notlarında belirtilmiştir), olmadan node genesis'i reddeder.

## Adım 10: Firewall

```bash
sudo ufw allow ${NODE_PORT}656/tcp comment "gno-mainnet P2P"
sudo ufw allow ${NODE_PORT}657/tcp comment "gno-mainnet RPC"
```

P2P portu dışa açık olmalı, aksi halde diğer node'lar size bağlanamaz ve peer sayınız düşük kalır. RPC portunu sadece kendi ihtiyacınız için (nginx proxy kurmayacaksanız) dışa açmanız yeterli.

## Adım 11: Başlat ve Senkronizasyonu Doğrula

```bash
sudo systemctl restart gno-mainnet
sudo journalctl -u gno-mainnet -f
```

Not: İlk başlangıçta "InitChainer: standard libraries loaded" mesajından sonra birkaç dakika sessiz kalabilir. Bu normal, 3.26M hesaplık genesis state'i işleniyor. Bu sırada node'u **durdurmayın**, yarıda kesilirse (SIGTERM timeout → SIGKILL) süreç en baştan başlar.

Peer bağlantısını kontrol edin:
```bash
curl -s http://localhost:${NODE_PORT}657/net_info | jq .result.n_peers
```
0'dan büyük bir sayı görmelisiniz (seed'ler doğru girildiyse genelde 10-15 civarı).

Senkronizasyon durumu:
```bash
curl -s http://localhost:${NODE_PORT}657/status | jq .result.sync_info
```
`catching_up: false` olduğunda tam senkronize olmuşsunuz demektir.

---

## Opsiyonel: RPC'nizi Public'e Açmak (nginx + Cloudflare)

Node'unuzu kendi domain'iniz üzerinden HTTPS ile herkese açmak isterseniz (biz de mainnet-rpc/api.zeycanode.com için bunu yaptık):

```bash
sudo tee /etc/nginx/sites-available/gnoland-mainnet-rpc > /dev/null <<NGINXEOF
server {
    listen 80;
    server_name RPC-DOMAIN-ADINIZ;

    location / {
        proxy_pass http://127.0.0.1:58657;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;

        add_header Access-Control-Allow-Origin * always;
        add_header Access-Control-Allow-Methods "GET, POST, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Content-Type" always;
    }
}
NGINXEOF

sudo ln -s /etc/nginx/sites-available/gnoland-mainnet-rpc /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d RPC-DOMAIN-ADINIZ
```

⚠️ Cloudflare kullanıyorsanız ve DNS kaydınız "Proxied" (turuncu bulut) modundaysa, certbot'un HTTP-01 doğrulaması genellikle 522 hatasıyla başarısız olur. Sertifika alma süresince DNS kaydını geçici olarak "DNS only" (gri bulut) yapın, sertifika alındıktan sonra tekrar "Proxied"'a çevirip Cloudflare SSL modunu **Full (strict)** yapın.

⚠️ `sites-enabled` içindeki dosyanın gerçek bir **symlink** olduğundan emin olun, bağımsız bir kopya değil, aksi halde `sites-available`'da yaptığınız gelecekteki değişiklikler hiç uygulanmaz (biz bunu explorer'ımızda yaşadık, saatlerce "neden değişmiyor" diye uğraştık).

---

## Faydalı Komutlar

```bash
# Servis yönetimi
sudo systemctl status gno-mainnet
sudo systemctl restart gno-mainnet

# Canlı loglar
sudo journalctl -u gno-mainnet -f --no-hostname -o cat

# Sync durumu
curl -s http://localhost:${NODE_PORT}657/status | jq .result.sync_info

# Peer sayısı
curl -s http://localhost:${NODE_PORT}657/net_info | jq .result.n_peers

# Node ID
gnoland-mainnet secrets get node_id --data-dir $HOME/mainnet-data

# Validator pubkey (ileride Register için gerekecek)
gnoland-mainnet secrets get validator_key --data-dir $HOME/mainnet-data

# Cüzdan bakiyesi
gnokey-mainnet query -remote "https://rpc.gno.land" auth/accounts/G1-ADRESINIZ --home $HOME/mainnet-data
```

## Sık Karşılaşılan Hatalar: Özet Tablo

| Hata | Sebep | Çözüm |
|---|---|---|
| `unable to determine GNOROOT` | Binary kaynak klasörden farklı yerde çalışıyor | Servis dosyasına `Environment=GNOROOT=$HOME/gno-mainnet` ekleyin |
| `config file ... does not exist` | config.toml `config/` alt klasöründe değil | `<data-dir>/config/config.toml` yoluna taşıyın (Adım 7) |
| `Blockpool has no peers` (sürekli) | Seed'ler `id@host:port` formatında değil ya da port eksik | Adım 6'daki tam seed adreslerini kullanın |
| Genesis hash uyuşmuyor | Bozuk/eski indirme | Dosyayı tekrar indirin, resmi hash ile karşılaştırın |
| `make install` sonrası eski chain binary'si bozuldu | Aynı sunucuda birden fazla chain | Binary'leri `gnoland-mainnet` gibi ayrı isimle derleyin (Adım 5) |
| nginx değişiklikleri hiç uygulanmıyor | `sites-enabled`'daki dosya symlink değil, bağımsız kopya | `sites-enabled`'ı silip gerçek symlink oluşturun |
| Explorer/tarayıcı eski veri gösteriyor | Sunucu `Cache-Control` header'ı göndermiyor, tarayıcı sezgisel önbellekliyor | nginx'e `add_header Cache-Control "no-cache, no-store, must-revalidate" always;` ekleyin |

---

## Not: Validator Kaydı (Register) Adımı

Bu rehber şu an için **full node kurulumuna** kadar olan kısmı kapsıyor. `r/gnops/valopers` üzerinden `Register` çağrısı göndermek için gas fee'ye yetecek kadar `ugnot` gerekiyor. Mainnet'te transferler §126 kuralı gereği genesis'te kilitli olduğundan ve faucet bulunmadığından, elimizde henüz token yok. Ayrıca resmi validator onboarding süreci (KYC + Technical Review) tamamlanmadan `Register` yapmak da önerilmiyor.

Token temin edilip GovDAO süreciyle (Legal Review → Technical Review → onay) mainnet validator seçimimiz netleştiğinde, validator oluşturma ve `Register` adımları bu rehbere ayrı bir bölüm olarak eklenecek.

---

**Bu rehber ZeycaNode tarafından, gno.land Test13→Topaz→Sapphire→Pearl→Mainnet süreçlerinde edinilen operasyonel tecrübeyle hazırlanmıştır.**

