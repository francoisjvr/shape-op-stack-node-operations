# Repair-trie watcher lessons: wrapper exit vs remote truth

Use this when a Shape mainnet `repair-trie` run is launched through SSH/background wrappers and the local watcher exits before the remote repair actually finishes.

## Durable findings

- A local Hermes background process or SSH wrapper can exit with `143` / `-15` while the remote offline repair is still running normally.
- On this stack, the real repair may appear as an anonymous `docker run --rm` container with a random Docker name, while the steady-state `shape-mainnet-op-reth` container remains exited. Do not infer repair completion from the named service container alone.
- Better completion signals than wrapper exit:
  - remote `ps` showing `/usr/local/bin/op-reth db --datadir /data repair-trie --chain /config/genesis-l2.runtime.json --color never`
  - anonymous repair container still present in `docker ps -a`
  - growing `docker logs` output or advancing log-file `mtime`
  - nontrivial CPU on the repair PID
- In the observed live run, progress continued past the stale local snapshot:
  - `40.87%` with `inconsistent_nodes=40`
  - then `48.05%` with `inconsistent_nodes=59`
  - then `60.06%`
  - then `67.91%` with `inconsistent_nodes=63`

## Recommended operator pattern

1. Stop mainnet `op-node`, then `op-reth`.
2. Launch offline `repair-trie`.
3. If the local session may die, install a **VPS-side postwatch script** that:
   - waits until the real `repair-trie` PID/container disappears,
   - starts `shape-mainnet-op-reth`,
   - waits for RPC on `127.0.0.1:18545`,
   - starts `shape-mainnet-op-node-reth`,
   - verifies canonical recovery:
     - local head > `29408468`
     - block `29408469` queryable
     - `current_l1 > 0`
     - `safe_l2` or `unsafe_l2` advancing
4. Treat wrapper termination as ambiguous until remote process state says otherwise.

## Pitfalls

- Do not key the watcher only on `docker ps` for `shape-mainnet-op-reth`; the repair is not that container.
- Do not assume a quiet local watcher means the remote repair is hung.
- Do not restart the main containers just because the wrapper vanished; verify the repair PID/container is really gone first.
