# SSH Monitoring and Cron Watchdogs

Use this note when operating the live Shape mainnet Dockerized Reth lane over SSH and you want fast visibility without opening Hermes.

## Primary rule

For this stack, **`docker logs` is the primary live log surface**.

Use `journalctl` only if you personally wrapped the Docker Compose stack in a systemd unit. If you launched the containers directly with Docker Compose, `journalctl` is secondary and usually less useful than the container logs themselves.

## One-shot SSH checks

### Container status

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.RunningFor}}' | grep 'shape-mainnet-op-' || true
```

### Recent op-reth logs

```bash
docker logs --tail 120 shape-mainnet-op-reth
```

### Recent op-node logs

```bash
docker logs --tail 120 shape-mainnet-op-node-reth
```

### Follow op-reth live

```bash
docker logs -f --tail 80 shape-mainnet-op-reth
```

### Follow op-node live

```bash
docker logs -f --tail 80 shape-mainnet-op-node-reth
```

### Resource snapshot

```bash
docker stats --no-stream shape-mainnet-op-reth shape-mainnet-op-node-reth
```

## RPC monitoring from SSH

### Local execution head in decimal

```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://127.0.0.1:18545 \
| python3 -c 'import sys,json; print(int(json.load(sys.stdin)["result"],16))'
```

### Public Shape head in decimal

Use a non-empty `User-Agent` for public probes so you do not get fooled by vantage-specific `403` or bare-probe behavior.

```bash
curl -s -H 'Content-Type: application/json' -H 'User-Agent: shape-monitor/1.0' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  https://mainnet.shape.network \
| python3 -c 'import sys,json; print(int(json.load(sys.stdin)["result"],16))'
```

### Local vs public lag

```bash
LOCAL=$(curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://127.0.0.1:18545 \
  | python3 -c 'import sys,json; print(int(json.load(sys.stdin)["result"],16))')
PUBLIC=$(curl -s -H 'Content-Type: application/json' -H 'User-Agent: shape-monitor/1.0' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  https://mainnet.shape.network \
  | python3 -c 'import sys,json; print(int(json.load(sys.stdin)["result"],16))')
echo "local=$LOCAL public=$PUBLIC lag=$((PUBLIC-LOCAL))"
```

### Rollup sync status

```bash
curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"optimism_syncStatus","params":[],"id":1}' \
  http://127.0.0.1:18545 \
| python3 -m json.tool
```

## Live watch loop from SSH

Use this when you want one terminal continuously showing progress:

```bash
watch -n 30 '
LOCAL=$(curl -s -H "Content-Type: application/json" \
  -d "{\"jsonrpc\":\"2.0\",\"method\":\"eth_blockNumber\",\"params\":[],\"id\":1}" \
  http://127.0.0.1:18545 \
  | python3 -c "import sys,json; print(int(json.load(sys.stdin)[\"result\"],16))")
PUBLIC=$(curl -s -H "Content-Type: application/json" -H "User-Agent: shape-monitor/1.0" \
  -d "{\"jsonrpc\":\"2.0\",\"method\":\"eth_blockNumber\",\"params\":[],\"id\":1}" \
  https://mainnet.shape.network \
  | python3 -c "import sys,json; print(int(json.load(sys.stdin)[\"result\"],16))")
echo "local=$LOCAL public=$PUBLIC lag=$((PUBLIC-LOCAL))"
echo
curl -s -H "Content-Type: application/json" \
  -d "{\"jsonrpc\":\"2.0\",\"method\":\"eth_syncing\",\"params\":[],\"id\":1}" \
  http://127.0.0.1:18545
'
```

## journalctl usage

Only use this if you wrapped the stack in systemd.

### Follow your wrapper service

```bash
journalctl -fu <your-shape-mainnet-reth.service>
```

### Check Docker daemon logs

```bash
journalctl -u docker --since '15 min ago'
```

If there is no systemd wrapper for the stack, do **not** invent one mentally — go back to `docker logs`.

## Cron usage

This repo does **not** depend on cron for core node operation.

Cron is fine for lightweight append-only watchdog logging, for example:

```cron
*/5 * * * * /usr/bin/docker ps --format '{{.Names}} {{.Status}}' | /usr/bin/grep 'shape-mainnet-op-' >> /var/log/shape-mainnet-container-status.log 2>&1
*/10 * * * * /root/bin/shape-mainnet-head-sample.sh >> /var/log/shape-mainnet-head.log 2>&1
```

A minimal script for the second cron line:

```bash
#!/usr/bin/env bash
set -euo pipefail

HEAD=$(/usr/bin/curl -s -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://127.0.0.1:18545 \
  | /usr/bin/python3 -c 'import sys,json; print(int(json.load(sys.stdin)["result"],16))')

echo "$(/usr/bin/date -u +%FT%TZ) local_head=$HEAD"
```

But for anything smarter than append-only logging — threshold alerts, restart conditions, or public-vs-local lag alarms — prefer a small explicit watchdog script over giant inline cron commands.

## Operator interpretation rule

Do not judge health from one signal only.

Always combine:
- container state
- recent `docker logs`
- decimal `eth_blockNumber`
- `optimism_syncStatus`
- local-versus-public lag when public probing is available from that vantage
