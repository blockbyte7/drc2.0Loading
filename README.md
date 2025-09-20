# Doginal Locking (Dogecoin, CLTV-in-P2SH)

Prevent movement of an inscribed UTXO (“Doginal”) until a specified **block height** on Dogecoin.

Doginal Locking uses an **absolute timelock** enforced by `OP_CHECKLOCKTIMEVERIFY` (BIP‑65) inside a **P2SH** output. This design matches Dogecoin’s current consensus/policy and does **not** require Taproot or CSV. It is the Dogecoin-native way to achieve the user-visible behavior of “frozen until height H, then spendable by the owner.”

---

## Capability differences vs. Bitcoin “Ordinal Lockers”

- **Bitcoin Ordinal Lockers**: Taproot script path + a NUMS internal key, and often **CSV (BIP‑112)** for *relative* timelocks.
- **Dogecoin today**: No Taproot on mainnet; **CSV is not active**. Use **CLTV (BIP‑65)** with **absolute block heights** inside **P2SH**.

**Implication:** Users get the same outcome (funds can’t move until height H), but the address is legacy **P2SH**, and the redeem script is revealed on spend.

---

## Threat model & invariants

- A locked Doginal **cannot be spent** until the chain height reaches **`lock_height`**.
- After maturity, the output is spendable **only** by a valid signature under the owner’s public key.
- There is **no alternative key path**; the spender must reveal the redeemScript and satisfy it.

---

## Script construction

**Redeem script (P2SH redeemScript):**
```
<lock_height: ScriptNum> OP_CHECKLOCKTIMEVERIFY OP_DROP <pubkey> OP_CHECKSIG
```

**Semantics**
- `<lock_height>` is an **absolute block height**. (Avoid timestamps.)
- `OP_CHECKLOCKTIMEVERIFY` requires the spending transaction to set **`nLockTime >= lock_height`** *and* to include at least one input with **`nSequence < 0xFFFFFFFF`** (non‑final). Otherwise the spend is invalid / non-standard.
- `OP_DROP` removes the lock value from the stack.
- `<pubkey>` is the owner’s **compressed** public key (33 bytes).
- `OP_CHECKSIG` ensures only the corresponding private key can authorize the post‑lock spend.

**Funding output (scriptPubKey):**
```
OP_HASH160 <HASH160(redeemScript)> OP_EQUAL
```
(standard **P2SH** output).

> **Privacy note:** P2SH reveals the full redeemScript at spend time (the pubkey and the exact lock height become public).

---

## Implementation reference (Node.js / `bitcore-lib-doge`)

### 1) Build the redeem script

```js
const dogecore = require('bitcore-lib-doge');
const { Script, Opcode, crypto } = dogecore;

function buildCltvRedeemScript(lockHeight, pubkeyHex) {
  if (!Number.isInteger(lockHeight) || lockHeight <= 0) {
    throw new Error('lockHeight must be a positive integer (block height)');
  }
  // Use minimal ScriptNum encoding for CLTV operand
  const bn = new crypto.BN(lockHeight);
  const lockBuf = bn.toScriptNumBuffer(); // minimal encoding

  // pubkeyHex should be a 33-byte compressed key (66 hex chars)
  const pubkeyBuf = Buffer.from(pubkeyHex, 'hex');
  if (pubkeyBuf.length !== 33) {
    throw new Error('pubkeyHex must be a 33-byte compressed public key');
  }

  return new Script()
    .add(lockBuf)                       // <lock_height>
    .add(Opcode.OP_CHECKLOCKTIMEVERIFY) // BIP-65 absolute timelock
    .add(Opcode.OP_DROP)                // clean stack
    .add(pubkeyBuf)                     // owner pubkey
    .add(Opcode.OP_CHECKSIG);           // enforce ownership
}
```

### 2) Derive the P2SH address from the redeem script

```js
function p2shAddressFromRedeemScript(redeemScript, network = dogecore.Networks.livenet) {
   const p2shScript = Script.buildScriptHashOut(redeemScript);
   return new Address(p2shScript, network).toString();
}
```

### 3) Fund the P2SH output with the Doginal-carrying UTXO

- Choose an amount comfortably **above your node/pool dust threshold** (varies by policy; many setups default near ~0.01 DOGE).  
- Send the Doginal to the **P2SH address** derived above.  
- Confirm the funding transaction is final and propagated.

---

## Spending (unlock) after maturity

At or after `lock_height`, create a spending transaction from the P2SH output to the user’s chosen destination.

**Requirements:**
- Set **`nLockTime = lock_height`** (or any value ≥ that).
- Ensure **at least one input** in the transaction has **`nSequence < 0xFFFFFFFF`** (e.g., `0xFFFFFFFE`). If all inputs are final (`0xFFFFFFFF`), CLTV is ignored.
- The input spending the P2SH output must push:
  ```
  <sig> <redeemScript>
  ```
  where `<sig>` is a DER-encoded ECDSA signature with a standard sighash flag (commonly `SIGHASH_ALL`).

**Sketch with `bitcore-lib-doge`:**
```js
const { Transaction } = dogecore;

function buildUnlockTx(utxo, redeemScript, destAddress, amountSatoshis, feeSatoshis, privKey, lockHeight) {
  const tx = new Transaction()
    .from({
      txId: utxo.txid,
      outputIndex: utxo.vout,
      script: redeemScript,     // redeemScript for P2SH input
      satoshis: amountSatoshis, // value of the locked output
    })
    .to(destAddress, amountSatoshis - feeSatoshis);

  // Set absolute locktime
  tx.nLockTime = lockHeight;

  // Ensure the spending input is non-final so CLTV is enforced
  tx.inputs.forEach((input) => {
    // Many bitcore versions expose sequenceNumber on inputs:
    input.sequenceNumber = 0xFFFFFFFE;
  });

  // Sign with the owner key corresponding to <pubkey> in the redeemScript
  tx.sign(privKey);

  return tx;
}
```

> **Note:** Libraries differ on how they expose `nLockTime` and input sequence. If helpers like `lockUntilBlockHeight()` exist, you may use them; otherwise set `tx.nLockTime` and `input.sequenceNumber` directly before signing.

---

## Duration → height conversion (UI layer)

Expose friendly presets and convert to an absolute height. Dogecoin’s target is ~1 block/minute; actual times vary. Show users the **estimated** unlock date/time and the **exact** block height.

```js
const BLOCKS_PER_MIN = 1;            // target rate (approximate)
const BLOCKS_PER_DAY = 60 * 24;

const presetDurations = {
  '30min': 30 * BLOCKS_PER_MIN,
  '1month': 30 * BLOCKS_PER_DAY,
  '3months': 90 * BLOCKS_PER_DAY,
  '6months': 180 * BLOCKS_PER_DAY,
};

function toLockHeight(currentHeight, choiceOrHeight) {
  if (Number.isInteger(choiceOrHeight) && choiceOrHeight > 0) {
    return choiceOrHeight; // explicit height provided
  }
  const blocks = presetDurations[choiceOrHeight];
  if (!blocks) throw new Error('Invalid duration preset');
  return currentHeight + blocks;
}
```

---

## RPC checklist (Dogecoin Core)

Below is a neutral RPC flow that you can adapt to your stack (indexer, backend service, etc.).

**Locking (funding):**
1. Build `redeemScript` with `buildCltvRedeemScript(lock_height, pubkeyHex)`.
2. Derive P2SH address with `p2shAddressFromRedeemScript`.
3. Create and broadcast a funding tx paying the P2SH address (value > dust threshold).

**Unlocking (spend after maturity):**
1. Wait until chain height ≥ `lock_height`.
2. Create a raw tx spending the P2SH output to the destination address.
3. Set `nLockTime = lock_height` (or higher).
4. Ensure at least one input has `nSequence < 0xFFFFFFFF`.
5. Provide `<sig> <redeemScript>` in the input’s unlocking script (your library’s signer should do this after you attach the redeemScript and sign).
6. Broadcast; the network will accept once the height condition is met.

## Compatibility matrix (2025‑09)

| Feature                          | Bitcoin | Dogecoin |
|----------------------------------|:-------:|:--------:|
| Taproot (script/key paths)       |   Yes   |    No    |
| CSV (BIP‑112) relative timelocks |   Yes   |   N/A*   |
| CLTV (BIP‑65) absolute timelock  |   Yes   |   Yes    |
| P2SH                             |   Yes   |   Yes    |

\* CSV not active on Dogecoin mainnet as of this writing; CLTV is the recommended primitive for absolute locks.

---

## Test recipe (quick sanity check)

1. Choose `lock_height = current_height + 20` (about ~20 minutes).
2. Build `redeemScript` and P2SH address; fund it with a clearly non-dust amount.
3. Immediately attempt to spend with `nLockTime = lock_height` and a non-final input — the mempool should **reject** because the chain hasn’t reached `lock_height` yet.
4. After the chain reaches `lock_height`, broadcast the same spend — it should be **accepted** and mined.

## License / disclaimer

This document is informational and reflects Dogecoin’s typical mainnet behavior. Always validate against the exact Dogecoin Core version and mempool policy used by your infrastructure.
