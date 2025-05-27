# Sepolia ve Beacon Sepolia RPC Node Kurulumu (Geth + Prysm)

Bu rehber, Ubuntu tabanlı bir sunucuda `Sepolia Ethereum (Geth)` ve `Beacon (Prysm)` full node kurulumunu açıklar. Kurulum sonunda RPC ve Beacon endpoint'lerini Aztec gibi uygulamalar için kullanabilirsiniz.

## 🖥️ Donanım Gereksinimleri

- **OS**: Ubuntu 20.04 veya üzeri
- **RAM**: 8-16 GB
- **CPU**: 4-6 çekirdek
- **Disk**: En az 600 GB SSD

---

## 1. Gerekli Paketleri Kurun

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip -y
```

## 2. Docker Kurulumu

```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

sudo apt-get update
sudo apt-get install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu   $(. /etc/os-release && echo "$VERSION_CODENAME") stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

sudo docker run hello-world
```

## 3. Klasör Yapısı ve JWT Oluşturma

```bash
mkdir -p /root/ethereum/execution
mkdir -p /root/ethereum/consensus
openssl rand -hex 32 > /root/ethereum/jwt.hex
```

## 4. `docker-compose.yml` Dosyasını Oluşturun

```bash
cd /root/ethereum
nano docker-compose.yml
```

İçerik:

```yaml
services:
  geth:
    image: ethereum/client-go:stable
    container_name: geth
    network_mode: host
    restart: unless-stopped
    ports:
      - 30303:30303
      - 30303:30303/udp
      - 8545:8545
      - 8551:8551
    volumes:
      - /root/ethereum/execution:/data
      - /root/ethereum/jwt.hex:/data/jwt.hex
    command:
      - --sepolia
      - --http
      - --http.api=eth,net,web3
      - --http.addr=0.0.0.0
      - --authrpc.addr=0.0.0.0
      - --authrpc.jwtsecret=/data/jwt.hex
      - --authrpc.port=8551
      - --syncmode=snap
      - --datadir=/data

  prysm:
    image: gcr.io/prysmaticlabs/prysm/beacon-chain
    container_name: prysm
    network_mode: host
    restart: unless-stopped
    volumes:
      - /root/ethereum/consensus:/data
      - /root/ethereum/jwt.hex:/data/jwt.hex
    depends_on:
      - geth
    ports:
      - 4000:4000
      - 3500:3500
    command:
      - --sepolia
      - --accept-terms-of-use
      - --datadir=/data
      - --execution-endpoint=http://127.0.0.1:8551
      - --jwt-secret=/data/jwt.hex
      - --rpc-host=0.0.0.0
      - --rpc-port=4000
      - --grpc-gateway-host=0.0.0.0
      - --grpc-gateway-port=3500
      - --checkpoint-sync-url=https://checkpoint-sync.sepolia.ethpandaops.io
      - --genesis-beacon-api-url=https://checkpoint-sync.sepolia.ethpandaops.io
```

## 5. Node'u Başlatın

```bash
docker compose up -d
```

## 6. Sync Durumunu Kontrol Edin

Geth için:

```bash
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' http://localhost:8545
```

Prysm için:

```bash
curl http://localhost:3500/eth/v1/node/syncing
```

## 🔁 Notlar

- Geth'in tamamen senkronize olması zaman alabilir.
- Disk doluluğunu izleyin: `df -h`
- Logları izlemek için: `docker compose logs -f`

---

## 🔐 RPC Erişim Noktaları

- **Execution RPC**: `http://<your-ip>:8545`
- **Beacon RPC**: `http://<your-ip>:3500`

Bu endpoint’leri örneğin Aztec gibi zincir üstü uygulamalarda kullanabilirsiniz.