<p align="center">
  <img src="https://github.com/user-attachments/assets/5c76256c-4aad-457d-a43f-8de77238dd22" width="450"/>
</p>

### Hello! Today we’ll set up a Sepolia node for Aztec and configure the necessary endpoints.

* I recommend at least 1.5 TB of disk space for this setup. (1TB won't be enough during snapshot extraction.)
* With all services running, approximately 221 GB of free space remains. (32GB swap is used.)
* If you plan to work extensively with endpoints, I suggest keeping the snapshot as a backup to avoid potential issues later.

### Let's get started

### System Update and Required Packages

* Installs essential dependencies such as curl, lz4, aria2, and docker.io.

```
sudo apt update && sudo apt install -y curl lz4 aria2 docker.io
```

### Install Docker Compose

* Downloads and makes docker-compose binary executable to manage Docker containers.

```
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Create Directories and Generate JWT

* Sets up necessary folder structure for the node and generates a jwt token for communication between consensus and execution clients.

```
mkdir -p /root/sepolia-node/{execution,beacon/data,jwt}
openssl rand -hex 32 > /root/sepolia-node/jwt/jwt.hex
```

### Snapshot

* Downloads and extracts snapshot to speed up synchronization.
* This process will take a while due to file size.
* It may seem unresponsive during extraction—be patient. (You can monitor with `btop`.)

```
aria2c -x 16 -s 16 -d /root -o snapshot.tar.lz4 \
  https://snapshots.publicnode.com/ethereum-sepolia-reth-8855864.tar.lz4
lz4 -d /root/snapshot.tar.lz4 -c | tar -xf - -C /root/sepolia-node/execution
```

### Create Reth Configuration File

* Configures pruning to manage disk usage.

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

### Docker Compose YAML File

* Defines Reth and Lighthouse containers, port mappings, and data directories.
* NOTE: The configuration below is exposed to the public (0.0.0.0).

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

### Start Node Services

* Starts the Docker containers in detached mode.

```
cd /root/sepolia-node
docker-compose up -d
```

### View Logs

* Use the following commands to monitor container logs.

```
docker logs -f reth
```

```
docker logs -f lighthouse
```

* Beacon will be available shortly. Reth may take longer to fully sync.

### Check Sync Status

* Execution (Reth) sync status:

```
curl -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}'
```

* If `"result": false`, the sync is complete.

* Beacon (Lighthouse) sync status:

```
curl http://localhost:5052/eth/v1/node/syncing
```

* If `"is_syncing": false`, the sync is complete.

### Adding Aztec RPC

* You can use the following flags when launching Aztec:

```
--l1-rpc-urls http://localhost:8545 \
--l1-consensus-host-urls http://localhost:5052
```

Note: This setup is exposed to the public (0.0.0.0). Reth and Lighthouse services are accessible from all interfaces. This allows external applications running on the same server (like an Aztec node) to communicate with them.

To prevent unauthorized access to RPC endpoints, it is strongly recommended to use a reverse proxy (e.g., NGINX) with IP filtering. (Optionally, an NGINX configuration can be added to this repo.)

Additionally, for enhanced system-level security, it is advised to restrict access to specific IPs using firewall tools such as `iptables` or `ufw`.


### Thanks to [Codeesura](https://github.com/codeesura/) for creating the initial version of this repo.

### If you face any issues, feel free to reach out via [Telegram](https://t.me/tigernode).




