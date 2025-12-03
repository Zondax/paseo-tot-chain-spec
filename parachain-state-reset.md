# Parachain State Reset Guide

This document describes the process to reset a parachain's state without deregistering it from the relay chain. This is useful when you need to wipe parachain data (e.g., delete PVCs) while keeping the relay chain running.

## Problem

When you delete a parachain's chain data (PVCs), the parachain resets to genesis (block #0), but the relay chain still has accumulated state about the parachain:

- **DMP Queue**: Downward messages (XCM) sent to the parachain
- **DMP Head**: Hash of the message queue chain
- **Parachain Head**: The last known block header
- **Validation Code**: The WASM runtime registered for the parachain

This state mismatch causes the collators to fail with errors like:

```
Failed to get header for included block
assertion `left == right` failed in enqueue_inbound_downward_messages
Parachain code doesn't match validation code stored in the relay chain state
```

## Prerequisites

- Access to the relay chain with sudo privileges (or proxy to sudo)
- Access to the parachain collator RPC
- The parachain chainspec (for validation code)

## Step-by-Step Process

### Step 1: Delete Parachain Chain Data

Delete only the parachain chain data PVCs and restart the pods. A new PVC will be automatically generated. **Do NOT delete**:
- Relay chain data (embedded relay chain state)
- Keystore data (node keys, session keys)

```bash
# List PVCs to identify the correct ones
kubectl get pvc -n <namespace> | grep <collator-name>

# Delete ONLY the parachain chain-data PVC (NOT relay chain data or keystore)
# The parachain data PVC typically contains "chain-data" in the name
kubectl delete pvc <parachain-chain-data-pvc> -n <namespace>

# Restart the collator pods (new PVCs will be created automatically)
kubectl rollout restart statefulset <collator-statefulset> -n <namespace>
```

Example for Asset Hub collators:
```bash
# Delete parachain data PVCs (keep relay chain and keystore)
kubectl delete pvc chain-data-paseo-tot-collator-assethub-0 -n polkadot-nodes
kubectl delete pvc chain-data-paseo-tot-collator-assethub-bob-0 -n polkadot-nodes

# Restart collators (new PVCs will be created automatically)
kubectl rollout restart statefulset paseo-tot-collator-assethub -n polkadot-nodes
kubectl rollout restart statefulset paseo-tot-collator-assethub-bob -n polkadot-nodes
```

Wait for the collators to start and initialize with fresh genesis state before proceeding.

### Step 2: Get New Genesis Header from Collator

After resetting the parachain data, get the genesis header from the collator:

```bash
kubectl port-forward svc/<collator-service> 9944:9944 -n <namespace> &

curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"chain_getHeader","params":[]}' \
  http://127.0.0.1:9944
```

Example output:
```json
{
  "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "number": "0x0",
  "stateRoot": "0x3467ea312f781e8a9416d6e56eff2628dd9886cf2c015cb56d432ff1b7bef73c",
  "extrinsicsRoot": "0x03170a2e7597b7b7e3d84c05391d139a62b157e78786d8c082f29dcf4c111314",
  "digest": { "logs": [] }
}
```

### Step 3: SCALE-Encode the Genesis Header

Encode the header for use in `paras.forceSetCurrentHead`:

```
0x<parentHash><number_compact><stateRoot><extrinsicsRoot><digest_logs>
```

For genesis (block 0):
- parentHash: 32 zero bytes
- number: `00` (compact encoding of 0)
- stateRoot: 32 bytes from header
- extrinsicsRoot: 32 bytes from header
- digest logs: `00` (empty vector)

Example encoded header:
```
0x0000000000000000000000000000000000000000000000000000000000000000003467ea312f781e8a9416d6e56eff2628dd9886cf2c015cb56d432ff1b7bef73c03170a2e7597b7b7e3d84c05391d139a62b157e78786d8c082f29dcf4c11131400
```

### Step 4: Get DMP Head Expected by Parachain

Query the parachain's expected DMP MQC head:

```bash
curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"state_getStorage","params":["0x45323df7cc47150b3930e2666b0aa313911a5dd3f1155f5b7d0c5aa102a757f9"]}' \
  http://127.0.0.1:9944
```

If it returns `null` or `<none>`, the expected value is all zeros:
```
0x0000000000000000000000000000000000000000000000000000000000000000
```

### Step 5: Get Storage Keys for DMP Queue

On the relay chain, get the storage keys for the parachain's DMP state:

```javascript
// Run in Polkadot.js Apps -> Developer -> JavaScript
const paraId = 1000;
const headKey = api.query.dmp.downwardMessageQueueHeads.key(paraId);
const queueKey = api.query.dmp.downwardMessageQueues.key(paraId);
console.log('Head key:', headKey);
console.log('Queue key:', queueKey);
```

Example keys for para 1000:
- Queue: `0x63f78c98723ddc9073523ef3beefda0ca95dac46c07a40d91506e7637ec4ba57b6ff6f7d467b87a9e8030000`
- Head: `0x63f78c98723ddc9073523ef3beefda0c4d7fefc408aac59dbfe80a72ac8e3ce5b6ff6f7d467b87a9e8030000`

### Step 6: Get Validation Code from Chainspec

```bash
curl -s <chainspec-url> | jq -r '.genesis.raw.top["0x3a636f6465"]' > /tmp/wasm_code.txt
```

### Step 7: Execute Sudo Calls on Relay Chain

Connect to the relay chain via Polkadot.js Apps and execute these calls in order:

#### 7.1 Clear DMP Queue and Set Head

```
sudo.sudo(
  call: system.setStorage([
    [
      "<queue_storage_key>",
      "0x00"
    ],
    [
      "<head_storage_key>",
      "<expected_dmp_head>"
    ]
  ])
)
```

Example:
```
system.setStorage([
  ["0x63f78c98723ddc9073523ef3beefda0ca95dac46c07a40d91506e7637ec4ba57b6ff6f7d467b87a9e8030000", "0x00"],
  ["0x63f78c98723ddc9073523ef3beefda0c4d7fefc408aac59dbfe80a72ac8e3ce5b6ff6f7d467b87a9e8030000", "0x0000000000000000000000000000000000000000000000000000000000000000"]
])
```

#### 7.2 Set Parachain Head

```
sudo.sudo(
  call: paras.forceSetCurrentHead(
    para: 1000,
    newHead: <encoded_genesis_header>
  )
)
```

#### 7.3 Update Validation Code (if chainspec changed)

```
sudo.sudo(
  call: paras.forceSetCurrentCode(
    para: 1000,
    newCode: <wasm_code_from_chainspec>
  )
)
```

### Step 8: Restart Collators

```bash
kubectl rollout restart statefulset <collator-statefulset> -n <namespace>
```

### Step 9: Verify

Check collator logs for successful block production:

```bash
kubectl logs -f <collator-pod> -n <namespace>
```

You should see:
- No more `Failed to get header for included block` errors
- No more DMP assertion failures
- Blocks being produced: `🏆 Imported #1`, `#2`, etc.

## Using Proxy for Sudo

If sudo is behind a proxy, wrap the calls:

```
proxy.proxy(
  real: <sudo_account>,
  forceProxyType: None,
  call: sudo.sudo(
    call: <inner_call>
  )
)
```

## Troubleshooting

### DMP Head Keeps Changing

If new DMP messages keep arriving, you're in a race condition. Options:
1. Execute all sudo calls quickly in succession
2. Consider full deregistration with `registrar.forceRemovePara(paraId)`

### Validation Code Mismatch

If you see `Parachain code doesn't match validation code`:
1. Verify the WASM in chainspec matches what you're registering
2. Use `paras.forceSetCurrentCode` to update the relay chain's copy

### Still Failing After All Steps

If the parachain still doesn't produce blocks, the cleanest solution is full deregistration:

```
sudo.sudo(
  call: registrar.forceRemovePara(1000)
)
```

Wait 2 sessions, then re-register with:
```
sudo.sudo(
  call: registrar.forceRegister(
    who: <account>,
    deposit: 0,
    id: 1000,
    genesisHead: <encoded_header>,
    validationCode: <wasm_code>
  )
)
```

## Reference: Common Storage Keys

| Storage Item | Pallet | Description |
|-------------|--------|-------------|
| `dmp.downwardMessageQueues(paraId)` | DMP | Pending downward messages |
| `dmp.downwardMessageQueueHeads(paraId)` | DMP | MQC head hash |
| `paras.heads(paraId)` | Paras | Current parachain head |
| `paras.currentCodeHash(paraId)` | Paras | Validation code hash |
| `paras.paraLifecycles(paraId)` | Paras | Parachain lifecycle state |

## Reference: Parachain Genesis Storage

| Storage Key | Description |
|-------------|-------------|
| `0x3a636f6465` | `:code` - WASM runtime |
| `0x45323df7cc47150b3930e2666b0aa313911a5dd3f1155f5b7d0c5aa102a757f9` | `parachainSystem.lastDmqMqcHead` |
