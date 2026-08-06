# Verifying the Solen founder

This directory publishes a **cryptographic attestation** that binds the named,
legally-incorporated founder of Solen to the on-chain founder key. It lets any
third party (an exchange, investor, or auditor) independently confirm — with
their own tools, without trusting Solen's software — that the same person named
on the corporate filings controls the founder account.

Files:

| File | What it is |
|------|------------|
| `founder-attestation.txt` | The exact statement that is signed (sign the raw UTF-8 bytes, unchanged). |
| `founder-attestation.sig` | Ed25519 signature of `founder-attestation.txt`, as hex. |

Solen signs with **standard RFC 8032 Ed25519** (`ed25519-dalek`, `verify_strict`),
so the signature verifies with any conventional Ed25519 library — libsodium,
PyNaCl, Node's `crypto`, etc.

---

## What this proves (and what it doesn't)

**Proves:** the holder of the founder account's Ed25519 key signed a statement
claiming the legal identity *Porter Stovall / Solen Inc. (WY Filing ID
2026-002015126)*, and that key is the on-chain auth key of the founder account.

**Does not prove, on its own:** that Porter Stovall *is* who the LinkedIn/legal
docs say — that comes from KYC (e.g. a CertiK KYC badge) and the corporate
registry. This attestation is the *key-control* link in the chain; identity and
entity verification are the other two links. It is **not** a security audit and
says nothing about the safety of the protocol code.

---

## Verify the signature (any machine, no Solen tooling)

```bash
# Requires: pip install pynacl
python3 - <<'PY'
from nacl.signing import VerifyKey
from nacl.exceptions import BadSignatureError

msg = open("founder-attestation.txt", "rb").read()          # exact bytes
sig = bytes.fromhex(open("founder-attestation.sig").read().strip())
pk  = bytes.fromhex("<FILL: FOUNDER_PUBLIC_KEY_HEX>")        # from the statement

try:
    VerifyKey(pk).verify(msg, sig)
    print("OK — signature is valid for the stated founder public key")
except BadSignatureError:
    print("FAIL — signature does not match")
PY
```

## Confirm the key is the on-chain founder account's auth key

Verifying the signature only proves the signer holds *that* key. To tie it to
the live chain, confirm the same public key authenticates the founder account:

```bash
solen-cli --network mainnet account <FOUNDER_ACCOUNT_ID>
# check the account's Ed25519 auth key equals <FOUNDER_PUBLIC_KEY_HEX>
```

(Or query the account over the public RPC at https://rpc.solenchain.io and read
its auth method.) If the on-chain auth key matches the key that signed the
statement, the loop is closed: named founder → key → founder account.

---

## For the founder: how to produce the signature

> **Run this OFFLINE, on a machine you trust.** Your 32-byte seed is full
> control of the account and its funds — never paste it into any online service.

1. Fill the two `<FILL: ...>` fields in `founder-attestation.txt` with the
   founder account id and its Ed25519 public key. Do not change anything else —
   the signature covers the file byte-for-byte. (Update `Issued`/`Nonce` if you
   sign on a later date.)
2. Sign the exact bytes:

```bash
# Requires: pip install pynacl   (run offline)
python3 - <<'PY'
import getpass
from nacl.signing import SigningKey

seed = bytes.fromhex(getpass.getpass("Paste 32-byte hex seed (input hidden): ").strip())
sk   = SigningKey(seed)
print("public key hex:", sk.verify_key.encode().hex())   # paste into the statement
msg  = open("founder-attestation.txt", "rb").read()
open("founder-attestation.sig", "w").write(sk.sign(msg).signature.hex() + "\n")
print("wrote founder-attestation.sig")
PY
```

3. Confirm the printed public key matches the one in the statement, then commit
   both `founder-attestation.txt` and `founder-attestation.sig`.

**PQ note:** if the founder account has been upgraded to post-quantum auth
(ML-DSA-65) *only*, its on-chain auth key is no longer Ed25519 — in that case
attest with the ML-DSA key using Solen's tooling instead, since generic
libraries can't verify ML-DSA. A **hybrid** (Ed25519 + ML-DSA) account still
lists the Ed25519 key, so the flow above still works.
