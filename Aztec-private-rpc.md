# 🪩 Sepolia RPC + Prysm Beacon Setup (2025 Updated Guide)

### A complete, modern setup for a Sepolia Execution (Geth) and Consensus (Prysm) node — ideal for Aztec Sequencer or private RPC usage.

## ⚙️ System Requirements

## Minimum (RPC only):

### 4 vCPU

### 8 GB RAM

### 1 TB+ SSD

## Recommended (RPC + Aztec):

### 6+ vCPU

### 16 GB+ RAM

### 1 TB+ SSD

### Ubuntu 22 / 24 LTS

## 🧩 Step 0 — Become Root
```bash 
sudo -i
```

## ⚙️ Step 1 — Install Dependencies
```bash 
apt-get update && apt-get upgrade -y
apt-get install -y curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip net-tools ufw
```

## 🐳 Step 2 — Install Docker (Official)
```bash 
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do apt-get remove -y $pkg >/dev/null 2>&1 || true; done
apt-get install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" > /etc/apt/sources.list.d/docker.list
apt-get update && apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable docker && systemctl restart docker
docker run --rm hello-world
```

## 🏗 Step 3 — Prepare Folders & JWT Secret
```bash 
mkdir -p /root/ethereum/execution
mkdir -p /root/ethereum/consensus
openssl rand -hex 32 > /root/ethereum/jwt.hex
cat /root/ethereum/jwt.hex
```

## 🧾 Step 4 — Configure Docker Compose
```bash 
cd /root/ethereum
```

Create and open the file:
```bash 
nano docker-compose.yml
```

Paste this:
```bash 
services:
  geth:
    image: ethereum/client-go:v1.16.4
    container_name: geth
    network_mode: host
    restart: unless-stopped
    ports:
      - 30303:30303
      - 30303:30303/udp
      - 8545:8545
      - 8546:8546
      - 8551:8551
    volumes:
      - /root/ethereum/execution:/data
      - /root/ethereum/jwt.hex:/data/jwt.hex
    command:
      - --sepolia
      - --http
      - --http.addr=0.0.0.0
      - --http.api=eth,net,web3
      - --http.vhosts=*
      - --ws
      - --ws.addr=0.0.0.0
      - --ws.port=8546
      - --authrpc.addr=0.0.0.0
      - --authrpc.port=8551
      - --authrpc.vhosts=*
      - --authrpc.jwtsecret=/data/jwt.hex
      - --syncmode=snap
      - --cache=4096
      - --maxpeers=100
      - --datadir=/data
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"

  prysm:
    image: gcr.io/prysmaticlabs/prysm/beacon-chain:v6.1.2
    container_name: prysm
    network_mode: host
    restart: unless-stopped
    depends_on:
      - geth
    ports:
      - 4000:4000
      - 3500:3500
    volumes:
      - /root/ethereum/consensus:/data
      - /root/ethereum/jwt.hex:/data/jwt.hex
    command:
      - --sepolia
      - --accept-terms-of-use
      - --datadir=/data
      - --disable-monitoring
      - --rpc-host=0.0.0.0
      - --rpc-port=4000
      - --grpc-gateway-host=0.0.0.0
      - --grpc-gateway-port=3500
      - --grpc-gateway-corsdomain=*
      - --min-sync-peers=3
      - --execution-endpoint=http://127.0.0.1:8551
      - --jwt-secret=/data/jwt.hex
      - --checkpoint-sync-url=https://checkpoint-sync.sepolia.beaconcha.in
      - --genesis-beacon-api-url=https://checkpoint-sync.sepolia.beaconcha.in
      - --suggested-fee-recipient=0x0000000000000000000000000000000000000000
      - --subscribe-all-data-subnets
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"
```

## 🚀 Step 5 — Start Your Node
```bash 
docker compose up -d
```

## 🔍 Step 6 — Check Logs
```bash 
docker compose logs -f
```
Press Ctrl + C to exit log view.

## 🧱 Step 7 — Check Node Health

Execution (Geth):
```bash 
curl -s -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' \
  http://localhost:8545 | jq
```

Beacon (Prysm):
```bash 
curl -s http://localhost:3500/eth/v1/node/syncing | jq
```

## 🔐 Step 8 — Open Firewall
```bash 
ufw allow 22/tcp
ufw allow ssh
ufw allow 8545/tcp
ufw allow 3500/tcp
ufw allow 30303/tcp
ufw allow 30303/udp
ufw enable
```

## 🪄 Step 9 — Mirror Switchers (if checkpoint fails)

Switch to ChainSafe
```bash 
sed -i 's#checkpoint-sync.sepolia.beaconcha.in#beaconstate-sepolia.chainsafe.io#g' /root/ethereum/docker-compose.yml && docker compose up -d prysm
```

Switch to EthStaker
```bash 
sed -i 's#checkpoint-sync.sepolia.beaconcha.in#sepolia-checkpoints.ethstaker.cc#g' /root/ethereum/docker-compose.yml && docker compose up -d prysm
```

## ⏱ Step 10 — Verify Sync Completion

✅ Execution fully synced
```bash 
curl -s -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' \
  http://localhost:8545
# → {"result":false}
```

✅ Beacon fully synced
```bash 
curl -s http://localhost:3500/eth/v1/node/syncing
# → "is_syncing": false
```

## 🧼 Step 11 — Maintenance (Optional)

Auto-clean /tmp daily
```bash 
echo "0 3 * * * root rm -rf /tmp/* /tmp/.[!.]* >/dev/null 2>&1" > /etc/cron.d/tmp-clean && chmod 644 /etc/cron.d/tmp-clean
```

Auto-prune Docker weekly
```bash 
echo "@weekly root docker system prune -af --volumes >/dev/null 2>&1" > /etc/cron.d/docker-prune && chmod 644 /etc/cron.d/docker-prune
```

## 🌐 RPC Endpoints
### Layer	Inside VPS	External Access
### Execution (Geth)	http://localhost:8545	
### Beacon (Prysm)	http://localhost:3500	
### Replace localhost with ur vps IP adress

## ✅ Done!
You now have a fully functional private Sepolia RPC ready for Aztec or custom use.
