---
name: Query an EVM chain via InfStones RPC
description: >-
  Read on-chain state (balances, blocks, transactions, logs) and broadcast signed
  transactions on BNB Chain / Ethereum-compatible networks through the InfStones
  hosted JSON-RPC endpoint.
api: openapi/infstones-bnb-chain-openapi.json
operations:
- eth_blockNumber
- eth_getBalance
- eth_getTransactionReceipt
- eth_getLogs
- eth_sendRawTransaction
---

# Query an EVM chain via InfStones RPC

Use the InfStones hosted node to talk to any EVM chain over standard Ethereum
JSON-RPC. This skill covers reading chain state and broadcasting a signed
transaction. Method names and semantics are the canonical Ethereum JSON-RPC set;
InfStones hosts the nodes.

## Authenticate

InfStones authenticates with a **per-project API key (`project_id`) in the URL
path** — there is no OAuth. Provision a key at https://app.infstones.com/api, then
build the endpoint for your chain/network:

```
https://api.infstones.com/bsc/mainnet/{project_id}
```

Swap `bsc/mainnet` for the chain/network you need (e.g. `eth/mainnet`). See
`authentication/infstones-authentication.yml`.

## Call convention

Every call is an HTTP `POST` with a JSON-RPC 2.0 body
(`conventions/infstones-conventions.yml`). Quantities and hashes are `0x`-prefixed
hex.

```bash
curl https://api.infstones.com/bsc/mainnet/{project_id} \
  -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":0}'
```

## Steps

1. **Get the chain tip** — call `eth_blockNumber` (no params) to confirm
   connectivity and get the current block height.
2. **Read an address balance** — call `eth_getBalance` with
   `["0x<address>", "latest"]`; the result is hex Wei.
3. **Confirm a transaction** — after broadcasting, poll
   `eth_getTransactionReceipt` with `["0x<txhash>"]`; a non-null receipt with
   `status: "0x1"` means success.
4. **Read events** — call `eth_getLogs` with a filter object
   (`{fromBlock, toBlock, address, topics}`) to pull contract event logs. Keep the
   block range bounded to avoid timeouts.
5. **Broadcast** — sign the transaction client-side, then call
   `eth_sendRawTransaction` with `["0x<signedTx>"]`. This is the only
   state-changing call here; it returns the transaction hash.

## Error handling

Errors come back as JSON-RPC 2.0 `error` objects
(`errors/infstones-problem-types.yml`): `-32602` invalid params (check hex
encoding), `-32601` method not found (unsupported on this chain/plan), `-32000`
execution reverted (inspect `error.data`). There is **no idempotency key** —
`eth_sendRawTransaction` is deduplicated by the network via the signed nonce, so
never resend with a fresh nonce assuming the prior one failed without checking the
receipt first.
