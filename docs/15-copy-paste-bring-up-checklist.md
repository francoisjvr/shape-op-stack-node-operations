# Copy/Paste Bring-Up Checklist

This is the **boring operator checklist** for a first parallel Shape mainnet `op-reth` bring-up.

It is deliberately opinionated.
If you are unsure, follow it in order.

## Assumptions

You already have:
- a validated Reth datadir at `/root/shape-mainnet-op-reth-data`
- config files staged in `/root/.shape-mainnet-op-reth-config`
- a `.env` file created from `examples/.env.example`
- an existing geth fallback you do **not** want to disturb

## 0. Go to the repo

```bash
cd /home/moltbot/.hermes/hermes-agent/shape-op-stack-node-operations
```

## 1. Sanity-check the files you are about to rely on

```bash
ls -lah examples
ls -lah .env
ls -lah /root/.shape-mainnet-op-reth-config
ls -lah /root/shape-mainnet-op-reth-data | head
```

You want to see:
- the compose template
- your `.env`
- config artifacts like `reth.uploaded.toml`, `rollup.runtime.json`, `genesis-l2.runtime.json`, `jwt.hex`
- a real-looking Reth datadir, not an empty placeholder

## 2. Check the important paths exist

```bash
test -d /root/shape-mainnet-op-reth-data && echo "reth data dir ok"
test -d /root/shape-mainnet-op-node-reth-data && echo "op-node data dir ok"
test -d /root/.shape-mainnet-op-reth-config && echo "config dir ok"
test -f /root/.shape-mainnet-op-reth-config/jwt.hex && echo "jwt ok"
```

## 3. Check that your isolated ports are free

```bash
ss -ltnup | egrep ':(18545|18546|18551|19222|19545|17300|31303)\b' || echo "isolated ports look free"
```

If these ports are already busy, stop and understand why before starting anything.

## 4. Render the compose config before starting

```bash
docker compose -f examples/docker-compose.recommended.yml --env-file .env config >/tmp/shape-reth-compose.rendered.yml
sed -n '1,220p' /tmp/shape-reth-compose.rendered.yml
```

You are checking for obvious stupidity:
- wrong paths
- placeholder env values left in place
- wrong ports
- empty bootnodes

## 5. Start only `op-reth` first

```bash
docker compose -f examples/docker-compose.recommended.yml --env-file .env up -d op-reth
docker logs --tail 120 shape-mainnet-op-reth
```

If that log command stays blank even though the container is up, your `op-reth` command is missing explicit stdout logging. Add `--log.stdout.format=log-fmt` and `--log.stdout.filter=info`, then recreate the container.

You want:
- container starts cleanly
- no instant path/config crash
- no obvious port-collision failure

## 6. Check `op-reth` RPC before adding `op-node`

```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://127.0.0.1:18545

curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' \
  http://127.0.0.1:18545
```

Do not expect perfection yet.
Just confirm it is alive enough to answer RPC.

## 7. Start `op-node`

```bash
docker compose -f examples/docker-compose.recommended.yml --env-file .env up -d op-node
docker logs --tail 160 shape-mainnet-op-node-reth
```

You want:
- no obvious JWT mismatch
- no instant rollup-config crash
- no obvious engine-wiring failure

## 8. Take a first health sample

```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://127.0.0.1:18545

curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  https://mainnet.shape.network

curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' \
  http://127.0.0.1:18545

curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"optimism_syncStatus","params":[],"id":1}' \
  http://127.0.0.1:19545
```

## 9. Watch the logs without hallucinating success

```bash
docker logs --since 10m shape-mainnet-op-reth | tail -n 120
docker logs --since 10m shape-mainnet-op-node-reth | tail -n 120
```

`op-reth` should now emit normal startup and block-processing lines into Docker logs. If `op-node` is noisy but `op-reth` is still silent, treat that as a config bug, not as proof the node is idle.

Remember:
- startup is not success
- `unsafe_l2` movement alone is not success
- zero EL peers is not the main failure signal on current Shape mainnet

## 10. Take repeated samples

Take at least 3 samples over time.

```bash
for i in 1 2 3; do
  echo "--- sample $i ---"
  date -u
  echo "local reth head"
  curl -s -H 'Content-Type: application/json' \
    -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
    http://127.0.0.1:18545
  echo
  echo "public shape head"
  curl -s -H 'Content-Type: application/json' \
    -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
    https://mainnet.shape.network
  echo
  echo "syncing"
  curl -s -H 'Content-Type: application/json' \
    -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' \
    http://127.0.0.1:18545
  echo
  sleep 180
done
```

## 10.5 If it catches up, run the post-sync sanity pass

Once local head reaches public head, do one more boring check instead of declaring victory instantly.

```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}' \
  http://127.0.0.1:18545

curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}' \
  https://mainnet.shape.network

curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_gasPrice","params":[],"id":1}' \
  http://127.0.0.1:18545

curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_gasPrice","params":[],"id":1}' \
  https://mainnet.shape.network

curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_getBlockByNumber","params":["latest",false],"id":1}' \
  http://127.0.0.1:18545

curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_getBlockByNumber","params":["latest",false],"id":1}' \
  https://mainnet.shape.network

docker inspect -f '{{.Name}} restart={{.RestartCount}} status={{.State.Status}}' \
  shape-mainnet-op-reth shape-mainnet-op-node-reth

docker logs --since 20m shape-mainnet-op-reth 2>&1 | tail -n 80
docker logs --since 20m shape-mainnet-op-node-reth 2>&1 | tail -n 80
```

What you want:
- local and public `eth_chainId` both say chain `360`
- local and public `eth_gasPrice` are in family, ideally identical at sample time
- latest local block hash matches public latest block hash
- restart count stays `0`
- `op-node` still processes unsafe blocks cleanly

What should not scare you by itself:
- `net_peerCount = 0`
- `safe_l2` and `finalized_l2` lagging behind `unsafe_l2`
- occasional `op-node` peer ping warnings while the execution head still matches public Shape

## 11. If it looks bad, stop the Reth lane cleanly

```bash
docker compose -f examples/docker-compose.recommended.yml --env-file .env down
```

That is the point of the parallel lane: it should be easy to stop without wrecking the geth fallback.

## 12. Quick failure triage

### Port collision smell
- container exits immediately
- logs mention bind/listen errors

Check:
```bash
ss -ltnup | egrep ':(18545|18546|18551|19222|19545|17300|31303)\b'
```

### Bad config smell
- instant startup crash
- logs mention missing files, bad TOML/JSON, or invalid flags

Check:
```bash
ls -lah /root/.shape-mainnet-op-reth-config
sed -n '1,200p' .env
```

### Fake movement smell
- `unsafe_l2` moves
- execution head stays flat

Check:
- repeated `eth_blockNumber` samples
- repeated `optimism_syncStatus`
- recent `op-reth` and `op-node` logs

## Bottom line

If this checklist feels slow, good.
Slow and boring beats deleting the fallback lane because a container looked optimistic for 90 seconds.
