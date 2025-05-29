<p align="center">
  <img src="https://github.com/user-attachments/assets/5c76256c-4aad-457d-a43f-8de77238dd22" width="450"/>
</p>



### Selamlar, bugün Aztec için Sepolia node kurup gerekli endpointleri oluşturacağız.

* Bu kurulum için en az 1.5 TB disk alanı öneriyorum. (1TB disk, indirme ve dizine çıkarma işlemi sırasında yetmeyecektir) 
* Tüm servisler çalışır durumdayken yaklaşık 221 GB boş alan kalmaktadır. (32GB swap kullanılmıştır)
* Eğer ciddi şekilde endpointler ile işiniz olacaksa daha sonra çıkabilecek potansiyel problemleri göz önünde bulundurarak snapshot'ı yedekte tutmanızı öneriyorum.



### Hazırsanız başlayalım

### Sistem Güncellemesi ve Gerekli Paketler

* Curl, lz4, aria2 ve docker.io gibi temel bağımlılıkları yükler.

```
sudo apt update && sudo apt install -y curl lz4 aria2 docker.io
```

### Docker Compose Kurulumu

* Docker container'ları yönetmek için docker-compose binary'sini indirir ve çalıştırılabilir hale getirir.



```
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```


### Dizinleri Oluştur ve JWT Anahtarı Üret

* Node için gerekli klasör yapısını kurar ve consensus client ile execution client arasında iletişim için jwt oluşturur.

```
mkdir -p /root/sepolia-node/{execution,beacon/data,jwt}
openssl rand -hex 32 > /root/sepolia-node/jwt/jwt.hex
```

### Snapshot

* Senkronizasyonu hızlandırmak için snapshot'ı indirip açar.
* Bu işlem dosya büyüklüğünden dolayı epey bir sürecek.
* Snaphot'ı gerekli dizine açma kısmında çalışmıyor gibi görünebilir, sabırla bekleyin. (Btop ile kontrol edebilirsiniz)


```
aria2c -x 16 -s 16 -d /root -o snapshot.tar.lz4 \
  https://snapshots.publicnode.com/ethereum-sepolia-reth-8434125.tar.lz4
lz4 -d /root/snapshot.tar.lz4 -c | tar -xf - -C /root/sepolia-node/execution
```

### Reth Konfigürasyon Dosyasını Oluştur

* Pruning yapılandırması yaparak disk kullanımını kontrol eder.

```
cat <<EOF > /root/sepolia-node/execution/reth.toml
[prune]
block_interval = 0

[prune.segments]
sender_recovery = { before = 0 }
transaction_lookup = { before = 0 }
receipts = { before = 0 }
account_history = { before = 0 }
storage_history = { before = 0 }
EOF
```

### Docker Compose YML Dosyası

* Reth ve Lighthouse container'larını tanımlar, port yönlendirmelerini ve veri klasörlerini belirtir.
* NOT: Aşağıdaki yapılandırma dış dünyaya açık şekilde (0.0.0.0) tasarlanmıştır.

```
cat <<EOF > /root/sepolia-node/docker-compose.yml
version: "3.8"

services:
  reth:
    image: ghcr.io/paradigmxyz/reth:v1.3.12
    container_name: reth
    restart: unless-stopped
    command: >
      node
      --chain sepolia
      --http
      --http.addr 0.0.0.0
      --ws
      --authrpc.jwtsecret /jwt/jwt.hex
      --authrpc.addr 0.0.0.0
      --authrpc.port 8551
      --datadir /data
      --config /data/reth.toml
    ports:
      - "8545:8545"
      - "8551:8551"
    volumes:
      - ./execution:/data
      - ./jwt:/jwt

  lighthouse:
    image: sigp/lighthouse:latest
    container_name: lighthouse
    restart: unless-stopped
    depends_on:
      - reth
    command: >
      lighthouse bn
      --network sepolia
      --checkpoint-sync-url https://checkpoint-sync.sepolia.ethpandaops.io
      --execution-endpoint http://reth:8551
      --execution-jwt /jwt/jwt.hex
      --http
      --http-address 0.0.0.0
      --metrics
      --datadir /data
    ports:
      - "5052:5052"
    volumes:
      - ./beacon/data:/data
      - ./jwt:/jwt
EOF
```

### Node Servislerini Başlat

* Docker container'larını başlatır ve arka planda çalıştırır.

```
cd /root/sepolia-node
docker-compose up -d
```

### Loglara Bakma

* Container loglarını izlemek için aşağıdaki komutları kullanabilirsiniz.

```
docker logs -f reth
```
```
docker logs -f lighthouse
```

* Beacon kısa sürede kullanılabilir halde olacaktır. Reth'nin senkronize olması biraz bekletebilir.

### Sync Durumunu Kontrol Etme

* Execution (Reth) sync durumu

```
curl -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}'
```
* "result":false ise işlem tamamdır.

* Beacon (Lighthouse) sync durumu

```
curl http://localhost:5052/eth/v1/node/syncing
```
* "is_syncing": false ise senkron tamamdır.


### Aztec Rpc Ekleme

* Aztec'i başlatırken endpoint’leri bu şekilde ekleyebilirsiniz.

```
--l1-rpc-urls http://localhost:8545 \
--l1-consensus-host-urls http://localhost:5052
```



EK BİLGİ: Bu yapılandırma dış dünyaya açık olacak şekilde düzenlenmiştir. Reth ve Lighthouse servisleri 0.0.0.0 IP adresi üzerinden tüm arayüzlerden erişilebilir durumdadır. Bu sayede aynı sunucuda çalışan harici uygulamalar (örneğin Aztec node gibi) bu servislerle iletişim kurabilir.

Ancak bu durumda RPC uç noktalarına yetkisiz erişimi önlemek için mutlaka bir ters proxy (örneğin NGINX) ile IP filtreleme yapmanız önerilir. (İsteğe bağlı olarak NGINX yapılandırması repoya eklenebilir.)

Ayrıca, sistem düzeyinde güvenliği artırmak için iptables veya ufw gibi güvenlik duvarı araçlarıyla sadece belirli IP'lere erişim izni verilmesi tavsiye edilir.

### Reponun ilk halini oluşturan [Codeesura](https://github.com/codeesura/) 'ya teşekkürler.

### Problem yaşamanız durumunda [Telegram](https://t.me/tigernode) adresimi ziyaret edebilirsiniz.
