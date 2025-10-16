# Fusaka (Pectra) Update Guide for Geth and Prysm Ethereum Node

This guide provides step-by-step instructions to update your Ethereum node running Geth and Prysm for the Fusaka (Pectra) hard fork.

We were running this node for **Aztec sequencer node**, proceed to avoid getting slashed.

## Stop node
```
cd ethereum
```
```
docker compose down -v
```

## Edit docker-compose.yml
```
nano docker-compose.yml
```

## Replace the following code
* I added descriptions of changes made to the previous `docker-compose.yml` file.
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
      - --http.api=eth,net,web3
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
    image: gcr.io/offchainlabs/prysm/beacon-chain:stable  # Updated repo/tag for v6.1.2+
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
      - --subscribe-all-data-subnets  # Added this flag to subscription for supernode
      - --verbosity=debug  # Added this temp for troubleshooting
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

## Restart node
```
docker compose up -d
```



## Check if nodes are synced
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


## Execute the following to signal the proposal
This part is for Aztec sequencer nodes:
```
curl -X POST http://localhost:8880 \
  -H 'Content-Type: application/json' \
  -d '{
    "jsonrpc":"2.0",
    "method":"nodeAdmin_setConfig",
    "params":[{"governanceProposerPayload":"0x9D8869D17Af6B899AFf1d93F23f863FF41ddc4fa"}],
    "id":1
  }'
```
