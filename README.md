# geth-prysm-node
Step by step guide for setting up a `docker-compose.yml` for running a `Sepolia` Ethereum full node using **Geth** as the `execution client` and **Prysm** as the `consensus client` on an Ubuntu-based system.

## Hardware Requirements
<table>
  <tr>
    <th colspan="3"> OS: Ubuntu 20.04 or later</th>
  </tr>
  <tr>
    <td>RAM</td>
    <td>CPU</td>
    <td>Disk</td>
  </tr>
  <tr>
    <td><code>8-16 GB</code></td>
    <td><code>4-6 cores</code></td>
    <td><code>Initially 550 GB SSD - Can grow upto 1 TB</code></td>
  </tr>
</table>

---

## Step 1. Install Dependecies
**Packages:**
```bash
sudo apt-get update && sudo apt-get upgrade -y
```
```bash
sudo apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev  -y
```
**Docker:**
```bash
sudo apt update -y && sudo apt upgrade -y
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y && sudo apt upgrade -y

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Test Docker
sudo docker run hello-world

sudo systemctl enable docker
sudo systemctl restart docker
```

---

## Step 2. Create Directories
```bash
mkdir -p /root/ethereum/execution
mkdir -p /root/ethereum/consensus
```

---

## Step 3. Generate the JWT secret:
```bash
openssl rand -hex 32 > /root/ethereum/jwt.hex
```
**Verify JWT secrets exist**:
```bash
cat /root/ethereum/jwt.hex
```

---

## Step 4. Configure `docker-compose.yml`
```bash
cd ethereum
```
```bash
nano docker-compose.yml
```
* Replace the following code into your `docker-compose.yml` file:
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
      - 8546:8546
      - 8551:8551
    volumes:
      - /root/ethereum/execution:/data
      - /root/ethereum/jwt.hex:/data/jwt.hex
    command:
      - --sepolia
      - --http
      - --http.api=eth,net,web3,engine
      - --http.addr=0.0.0.0
      - --authrpc.addr=0.0.0.0
      - --authrpc.vhosts=*
      - --authrpc.jwtsecret=/data/jwt.hex
      - --authrpc.port=8551
      - --syncmode=snap
      - --datadir=/data
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

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
      - 3600:3600 # New port for Blob Sink API
    command:
      - --sepolia
      - --accept-terms-of-use
      - --datadir=/data
      - --disable-monitoring
      - --rpc-host=0.0.0.0
      - --execution-endpoint=http://127.0.0.1:8551
      - --jwt-secret=/data/jwt.hex
      - --rpc-port=4000
      - --grpc-gateway-corsdomain=*
      - --grpc-gateway-host=0.0.0.0
      - --grpc-gateway-port=3500
      - --min-sync-peers=3
      - --checkpoint-sync-url=https://checkpoint-sync.sepolia.ethpandaops.io
      - --genesis-beacon-api-url=https://checkpoint-sync.sepolia.ethpandaops.io
      - --enable-blobs=true                        # Enabling Blob Sink
      - --blob-sync-method="direct"                # Direct Blob Sync Method
      - --blob-api-host=0.0.0.0                    # Blob Sink API host
      - --blob-api-port=3600                       # Blob Sink API port
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

---

## Step 5. Port conflicts:
* The setup uses ports 30303, 8545, 8546, 8551, 4000, and 3500. Verify they’re free:
```
netstat -tuln | grep -E '30303|8545|8546|8551|4000|3500'
```
If a port is in use, edit the `docker-compose.yml` to map to a different host port (e.g., `8546:8545` to `8547:8545`).

## Step 6. Run Geth & Prysm Nodes
* Start Geth & Prysm Nodes:
```bash
docker compose up -d
```

* Node Logs
```bash
# All lines of logs:
docker compose logs -f

# Last 100 lines of logs:
docker compose logs -fn 100
```

![image](https://github.com/user-attachments/assets/55eda130-5e7b-429d-9ef0-b48971123df7)

* As you see in the logs, `Prysm` is synced, and `Geth` yet to be synced later.
---

## Step 6. Checking If Nodes Are Synced
➡️**Execution Node (Geth)**
```
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' http://localhost:8545
```
* ✅Response if fully synced:
```json
{"jsonrpc":"2.0","id":1,"result":false}
```
* 🚫Response if still syncing:
```json
{"jsonrpc":"2.0","id":1,"result":{"currentBlock":"0x1a2b3c","highestBlock":"0x1a2b4d","startingBlock":"0x0"}}
```
* You'll see an object with `startingBlock`, `currentBlock`, and `highestBlock`, indicating the sync progress.

➡️**Beacon Node (Prysm)**
```bash
curl http://localhost:3500/eth/v1/node/syncing
```
* ✅Response if fully synced:
```json
{"data":{"head_slot":"12345","sync_distance":"0","is_syncing":false}}
```
If `is_syncing` is `false` and `sync_distance` is `0`, the beacon node is fully synced.

* 🚫Response if still syncing:
```json
{"data":{"head_slot":"12345","sync_distance":"100","is_syncing":true}}
```
* If `is_syncing` is `true`, the node is still syncing, and `sync_distance` indicates how many slots behind it is.

---

## Step 7. VPS Firewall
* Enable Firewall:
```bash
sudo ufw allow 22
sudo ufw allow ssh
sudo ufw enable
```

* Allow incoming traffic on the Geth/Prysm ports:
```bash
sudo ufw allow 8545/tcp    # Geth HTTP RPC
sudo ufw allow 3500/tcp    # Prysm HTTP API
sudo ufw allow 30303/tcp   # Geth P2P
sudo ufw allow 30303/udp   # Geth P2P
```

---

## Step 8. Getting the RPC Endpoints
### Execution Node (Geth)
Geth provides an HTTP RPC endpoint for interacting with the execution layer of Ethereum. Based on `docker-compose.yml` setup, Geth exposes port `8545` for HTTP RPC. The endpoints are:
* Inside the VPS: `http://localhost:8545`
* Outside the VPS: `http://<vps-ip>:8545` (replace `<vps-ip>` with your VPS’s public IP address, e.g., `http://203.0.113.5:8545`).
* **Aztec Sequencer Execution RPC**: `http://<vps-ip>:8545`. Since the Aztec Sequencer uses a bridge network to isolate docker container, you can NOT access Geth via **localhost**.

### Beacon Node (Prysm)
Prysm, as the beacon node, offers an HTTP gateway on port `3500`. the endpoints are:
* Inside the VPS: `http://localhost:3500`
* Outside the VPS: `http://<vps-ip>:3500` (e.g., `http://203.0.113.5:3500`).
* **Aztec Sequencer Consensus Beacon RPC**: `http://<vps-ip>:3500`. Since the Aztec Sequencer uses a bridge network to isolate docker container, you can NOT access Prysm via **localhost**.

---

## Step 9. Monitor System
* Monitor your hardware usage:
```bash
htop
```

* Monitor your Disk usage:
```bash
df -h
```

* Monitor Geth Disk usage:
```bash
docker exec -it geth du -sh /data
```

* Monitor Prysm Disk usage:
```bash
docker exec -it prysm du -sh /data
```
