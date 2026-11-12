---
title: "Bitcoin Signatures, Disentangled: ECDSA, Schnorr, Taproot, and SIGHASH"
slug: "bitcoin-signature-schemes"
date: 2026-11-12T17:30:00+08:00
draft: false
author: "Jackie"
tags: ["bitcoin", "ecdsa", "schnorr", "taproot", "sighash", "cryptography", "signatures", "cryptocurrency", "blockchain"]
categories: ["share-everyday"]
summary: "Bitcoin signatures split cleanly into three layers: the algorithm (ECDSA vs Schnorr), the locking script (P2PK, P2PKH, P2SH, SegWit, Taproot), and the SIGHASH commitment. Here is a layer-by-layer comparison of security, fees, and privacy, plus three approaches to multisig and a practical checklist."
showtoc: true
cover:
  image: "images/bitcoin-signature-schemes-flow.svg"
  alt: "Bitcoin signature stack: algorithm (ECDSA vs Schnorr), locking script (P2PK → P2PKH → P2SH → P2WPKH/P2WSH → P2TR), and SIGHASH (ALL / NONE / SINGLE + ANYONECANPAY; DEFAULT)"
  relative: false
images:
  - "images/bitcoin-signature-schemes-og.png"
---

Most explanations of "how Bitcoin signatures work" fuse three unrelated questions into one: **which mathematical algorithm makes the signature**, **which conditions must be met to spend**, and **which parts of the transaction the signature vouches for**. P2TR is not a signature algorithm; MuSig2 is not a script type. They belong to different layers. Once you pull them apart, the picture is much cleaner:

| Layer | Question it answers | Options |
| --- | --- | --- |
| Signature algorithm | How is the signature computed? | ECDSA (secp256k1) vs Schnorr (BIP340) |
| Locking script | What conditions authorize spending? | P2PK → P2PKH → P2SH → P2WPKH/P2WSH → P2TR |
| SIGHASH | Which parts of the transaction are committed to? | ALL / NONE / SINGLE + ANYONECANPAY; DEFAULT |

This division covers the usual arrangements on Bitcoin, but the layers are not perfectly orthogonal: `SIGHASH_DEFAULT`, for example, exists only under Taproot, and specific scripts accept only specific algorithms. What follows walks down each layer and ends with selection guidance.

## Layer one: the signature algorithm

### ECDSA: the only on-chain algorithm from 2009 to 2021

Bitcoin arrived just as the Schnorr patent was expiring (the commonly cited date is February 2008; counting 20 years from filing gives February 2010), while ECDSA already had standardized implementations and support in OpenSSL. So until Taproot activated, ECDSA over `secp256k1` was the only signature algorithm Bitcoin had on chain. Signatures use DER encoding and typically run 70–72 bytes, before the trailing one-byte SIGHASH flag:

```text
s = k⁻¹ · (H(m) + r·x) mod n
```

Three consequences follow from that equation:

- **The nonce `k` is the whole game.** Sign two different messages with the same private key and the same `k`, and the private key falls out directly. A `SecureRandom` flaw in Android in 2013 leaked private keys from a batch of mobile wallets for exactly this reason. The eventual mitigation is [RFC 6979](https://datatracker.ietf.org/doc/html/rfc6979): derive `k` deterministically from the private key and the message, and stop depending on the quality of a random number generator.
- **Malleability.** ECDSA's algebra lets anyone replace `s` with `n−s` and the signature still verifies; lax parsing of non-DER encodings and a mutable `scriptSig` offered further room for tweaking. On non-SegWit transactions, a third party could therefore change the txid. When Mt. Gox suspended withdrawals in 2014 it blamed transaction malleability, though the actual losses cannot be pinned on malleability alone. [BIP66](https://github.com/bitcoin/bips/blob/master/bip-0066.mediawiki) enforced strict DER at consensus level, low-S and similar rules arrived mainly through standard transaction policy, and [BIP62](https://github.com/bitcoin/bips/blob/master/bip-0062.mediawiki) proposed a fuller fix but never advanced and is now Closed. SegWit moved signatures out of txid computation, which eliminates txid malleability from third-party signature edits in **SegWit inputs** — but not all transaction malleability.
- **No Schnorr-style linear aggregation.** This does not mean ECDSA can do neither batch verification nor multisignatures; it means you cannot get Schnorr's clean, native key and signature aggregation. Threshold ECDSA generally requires considerably more elaborate protocols.

### Schnorr ([BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)): live with Taproot

Activated in November 2021 (block 709,632) as part of the [Taproot](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki) soft fork. The core relation looks like this (omitting the conditional negation that x-only encoding requires):

```text
s = k + e·x mod n,   e = tagged_hash("BIP0340/challenge", R || P || m)
```

Signatures are exactly 64 bytes (32 for `r`, 32 for `s`) with strict encoding; public keys are 32 bytes in x-only form, with verification using the even-`y` point. Verification checks `s·G = R + e·P`, which involves no modular inversion as ECDSA verification does, and there is no equivalent signature obtained by swapping `s` for `n−s`.

| | ECDSA | Schnorr (BIP340) |
| --- | --- | --- |
| Signature size | ~70–72 B (DER, excluding the SIGHASH byte) | Fixed 64 B |
| Public key | 33 B (compressed) | 32 B (x-only) |
| Malleability | An equivalent high-S form exists; standard transaction rules can constrain it | Avoided by strict encoding and signature construction |
| Single verification | Verification relation contains a modular inversion | No ECDSA-style inversion; real-world speed depends on the implementation |
| Batch verification | No native efficient batch check like Schnorr's | Supported with random coefficients |
| Signature / key aggregation | No equally clean native protocol; more complex multiparty schemes required | [MuSig2 (BIP327)](https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki), n-of-n |

**Linearity is the point — but you cannot simply add verification equations together.** Batch verification needs random coefficients so that invalid signatures cannot cancel one another out, and multi-scalar multiplication to make it fast. Across parties, protocols such as MuSig2 aggregate keys so the output looks exactly like an ordinary single-signer BIP340 signature. ECDSA is not mathematically barred from batch verification or threshold signing; it just lacks this clean native path.

## Layer two: the locking script

The algorithm decides how to sign; the script decides what has to be true for a signature to count. The evolutionary arc, roughly: **put the conditions behind a hash, then hide the optional conditions inside a [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree).**

- **P2PK** (2009): the public key sits directly in the output, common in early transactions; most of Satoshi's early mining rewards are generally believed to use this form. The key is public on chain while unspent, which is why this type often gets singled out in quantum discussions. That said, under the classical security model a published public key is not the same as a leaked private key.
- **P2PKH** (2009): only `hash160(pubkey)` is stored; the public key and signature are revealed at spend time. The output script is 25 bytes, a compressed-key input costs about 148 vB, and mainnet addresses begin with `1`.
- **P2MS** (bare multisig, [BIP11](https://github.com/bitcoin/bips/blob/master/bip-0011.mediawiki)): m-of-n is written straight into the output and all n public keys are exposed. `OP_CHECKMULTISIG` also has the historic off-by-one quirk of popping one extra element, so an extra placeholder is required.
- **P2SH** ([BIP16](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki), 2012): the output commits only to a 20-byte script hash; multisig, timelocks, and other complex conditions move off chain until spend time, when the full script is revealed. Mainnet addresses begin with `3`. Nested P2SH-P2WPKH was derived from this in 2017 for wallet compatibility (~91 vB per input) and now serves mainly as a transition path.
- **P2WPKH / P2WSH** (SegWit, [BIP141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki) / [BIP143](https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki), 2017-08, block 481,824): signatures move into the witness, where each byte costs 1 weight unit against 4 for non-witness data, bringing a P2WPKH input down to about 68 vB. Because the txid no longer covers the witness, txid malleability from third-party signature edits disappears — a key foundation for the [Lightning Network](https://lightning.network/). BIP143 fixes two older problems as a side effect: the signed message now includes the amount of the input being spent, so a malicious host cannot misstate the value and still produce a valid signature; and reusable hashes bring the signature-hash computation down from O(n²) to O(n).
- **P2TR** ([BIP341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki) / [BIP342](https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki), 2021-11): the output script is `OP_1 <32-byte output key Q>`, 34 bytes in total, and mainnet addresses begin with `bc1p`:

```text
Q = P + t·G,   t = tagged_hash("TapTweak", P || merkle_root)
```

When there is no script tree, `merkle_root` is omitted rather than hashed in as zero. The two spending paths:

- **Key path**: a single Schnorr signature. With `SIGHASH_DEFAULT` that is 64 bytes and a typical input of about 57.5 vB — the smallest among common standard single-sig inputs. While unspent, a P2TR output exposes neither an internal key nor a script tree; a key-path spend reveals neither either. So as far as the output script and witness structure go, a single-sig spend and a key-path spend with a hidden script tree are indistinguishable.
- **Script path**: reveals the executed script branch and a control block (leaf version and output-key parity bit, internal key, Merkle path). Tapscript replaces `OP_CHECKMULTISIG` with `OP_CHECKSIGADD`, accumulating successful signatures, which removes the placeholder quirk and eases batch validation; the spend does not reveal the other script branches.

### Output script and typical input sizes

| Type | Output script size | Input (typical) | Algorithm | Notes |
| --- | --- | --- | --- | --- |
| P2PK | 35 B (compressed key) / 67 B (uncompressed key) | ~114 vB | ECDSA | Public key exposed indefinitely |
| P2PKH | 25 B | ~148 vB | ECDSA | Assumes a compressed key |
| P2SH-P2WPKH | 23 B | ~91 vB | ECDSA | Nested transition form |
| P2WPKH | 22 B | ~68 vB | ECDSA | Common single-sig type |
| P2WSH 2-of-3 | 34 B | ~104.5–105 vB (varies slightly with DER length) | ECDSA ×2 | Witness script fully revealed |
| **P2TR key path** | 34 B | **~57.5 vB** | **Schnorr** | Uses DEFAULT; script tree stays hidden |

Output sizes above cover the `scriptPubKey` only and exclude the 8-byte amount field. Inputs are given in vB, with ECDSA rows assuming a low-R (71-byte DER) signature; real sizes shift with DER length, the SIGHASH flag, and script and witness structure.

## Layer three: SIGHASH

The same algorithm with a different SIGHASH flag signs a completely different "contract" — the flag decides which inputs and which outputs the signature commits to:

| Flag | Value | Commits to inputs | Commits to outputs | Typical use and caveats |
| --- | --- | --- | --- | --- |
| ALL | 0x01 | All | All | The default for legacy / SegWit v0; by far the most common |
| NONE | 0x02 | All | None | Outputs can be replaced; only for deliberate protocol designs |
| SINGLE | 0x03 | All | The output at the same index | CoinJoin-style joint transactions; must handle a missing output at that index |
| +ANYONECANPAY | Combined with the low flag, adding 0x80 | Only this input | Determined by the low flag | Crowdfunding, adding inputs, and similar |
| DEFAULT | 0x00 | All | All | Taproot only; same scope as ALL, saves one byte |

The table simplifies commitment at the level of prevouts and outputs. More precisely: under legacy and BIP143, NONE and SINGLE zero out the `nSequence` of other inputs in the hash, so those fields are not committed at all. Taproot (BIP341) does the opposite and commits to `nSequence` on every input even under NONE and SINGLE (unless ANYONECANPAY is set). That is one of BIP341's changes from BIP143.

Three historical footguns worth remembering:

- **The SINGLE out-of-range bug.** In legacy, when the input index has no matching output, `SignatureHash` returns the 32-byte representation of the integer 1 instead of a normal transaction digest (internally, `0x01` followed by 31 zero bytes in little-endian). This does not make arbitrary signatures valid, but a signature produced against that constant commits to nothing about the actual transaction and may be reusable in other transactions meeting the same condition. BIP143's v0 witness path avoids the degenerate branch, and Taproot (BIP341) simply declares `SIGHASH_SINGLE` invalid when the corresponding output is missing.
- **No amount commitment.** The legacy signed message omits the input amount — the root of hardware-wallet amount-spoofing attacks. BIP143 fixes this by including the input's amount.
- **Taproot's signature message** uses the `TapSighash` tagged hash instead of the double SHA256 of legacy / BIP143, and commits to the amount and `scriptPubKey` of the output being spent, among other context. The script path additionally commits to the tapleaf hash, key version, and `OP_CODESEPARATOR` position. Which fields are included still depends on the SIGHASH flag.

**In practice**: use ALL for legacy / SegWit v0 and DEFAULT for Taproot. Reach for NONE, SINGLE, or ANYONECANPAY only when a protocol explicitly requires it and you understand exactly what you are authorizing.

## Multisig: three implementations side by side

Multisig is where signature choices diverge most sharply:

| | P2WSH m-of-n | Taproot script path m-of-n | **MuSig2 (BIP327) key path** |
| --- | --- | --- | --- |
| On-chain form | Output is a script hash; spending reveals n public keys and the threshold | Output is a tweaked key; spending reveals the executed branch | A single aggregate key, itself tweakable for Taproot |
| Signature count | m ECDSA signatures | m Schnorr signatures | **1 aggregate Schnorr signature** |
| Typical 2-of-3 input | ~104.5–105 vB | ~108–116.5 vB (including the control block) | **Not directly supported; only 3-of-3 key path at ~57.5 vB** |
| Revealed at spend | Full script plus every public key | Executed branch plus control block | **No extra script information** |
| Threshold | Any m-of-n | Any (single script or per branch) | n-of-n only |
| Visible while unspent | Only that it is a P2WSH, not the threshold or keys | Identical to any other P2TR output | **Identical to any other P2TR output** |

The ~108–116.5 vB Taproot figure corresponds to the common shapes — a single 2-of-3 `OP_CHECKSIGADD` leaf, or three 2-of-2 leaves — assuming `SIGHASH_DEFAULT` and the relevant tree depth; depth, script, and SIGHASH all move the number.

A few points matter more than the table:

- **Rogue-key attacks.** Let the victim's key be `P₁`; an attacker can pick `P₂ = x₂·G − P₁` so that naive addition yields `P₁ + P₂ = x₂·G`, giving them sole control of the aggregate key. BIP327 first computes `L` over the list of public keys, then derives coefficients `a_i = int(tagged_hash("KeyAgg coefficient", L || P_i)) mod n`; by the MuSig2* optimization, the second distinct key and any duplicates of it take coefficient 1, and the specification defines how duplicate keys are handled.
- MuSig1 required three rounds of interaction; MuSig2 cuts it to two. Each signer's public nonce consists of two curve points, and implementations must store session state safely and never reuse a `secnonce`. BIP327 provides an interoperability standard, but before adopting it you still need to confirm end-to-end support in your wallets, coordinator, and hardware firmware.
- MuSig2 is n-of-n only. For genuine t-of-n thresholds, use [FROST](https://eprint.iacr.org/2020/852), write the threshold as an `OP_CHECKSIGADD` script under Taproot, or split signer combinations into script branches.
- A 2-of-3 Taproot script path is not necessarily cheaper than P2WSH; under the assumptions above it costs roughly 4–12 vB more. Its real payoff is that the routine cooperative path can use the key path, the emergency branches stay hidden in normal operation, and the output looks like any other P2TR while unspent.
- **The privacy boundary.** When n parties spend via a MuSig2 key path, the output script and witness structure are indistinguishable from an ordinary single-sig P2TR spend. That is not the same as anonymity: amounts, address reuse, and wallet fingerprints still leak metadata.

## How to choose

1. **Self-custody**: use a P2TR key path when your toolchain supports it; fall back to P2WPKH if your wallet or the recipient does not.
2. **Shared control by every party** (say, a team cold wallet with n-of-n): MuSig2, which has the smallest on-chain footprint and looks like an ordinary single-sig P2TR spend.
3. **Conditional spending** (inheritance, timeouts, two-factor, escape hatches): a Taproot script tree, with the most common cooperative path on the key path and everything else in the tree.
4. **Compatibility first** (exchange whitelists, older hardware wallets): P2WPKH / P2WSH multisig remains the safe choice; older environments may still need P2PKH.

[PSBT (BIP174)](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki) and [output descriptors](https://github.com/bitcoin/bitcoin/blob/master/doc/descriptors.md) let these designs work across devices and vendors — provided every link in the chain supports the same scripts and signing protocols. Confirm the full toolchain before you commit to a design.

## The failure mode never changed

However the algorithm layer evolves, **a leaked or reused nonce can expose the corresponding private key**:

- ECDSA uses RFC 6979 deterministic nonces; BIP340 derives nonces per its own specification; MuSig2 cannot simply copy purely deterministic session nonces and must follow BIP327's rules for high-quality randomness and single-use `secnonce` values.
- Hardware wallets with anti-klepto support let the host inject extra entropy and prove via [DLEQ proofs](https://github.com/bitcoin/bips/blob/master/bip-0374.mediawiki) and similar mechanisms that nonces were generated as agreed, reducing the risk of private-key exfiltration through nonce grinding. This is not something every wallet offers by default.
- In MuSig2, one party reusing its own `secnonce` may expose **that party's** key, not automatically everyone else's. Every endpoint must still audit how nonces are generated, stored, used, and wiped, so that it does not become the weak link in the authorization chain.

Looking ahead: [BIP118 (ANYPREVOUT)](https://github.com/bitcoin/bips/blob/master/bip-0118.mediawiki) would let a signature avoid committing to a specific UTXO, paving the way for eltoo-style Lightning updates; cross-input signature aggregation (combining signatures across inputs into one) could cut the cost of transactions like CoinJoin. The first builds on Schnorr / Taproot's extensible signing structure, the second exploits Schnorr's linearity more directly. Neither is a consensus rule today: BIP118 remains a Draft, and cross-input aggregation is not deployed.

## A quick checklist

**Algorithm layer**

1. Does ECDSA use RFC 6979 or an equally sound nonce scheme? Do Schnorr and MuSig2 follow BIP340 and BIP327 respectively?
2. Is nonce discipline auditable? Does the hardware wallet support anti-klepto? Is every MuSig2 `secnonce` used exactly once?

**Script layer**

3. Is the public key exposed long-term? P2PK and P2TR outputs publish a key outright; P2PKH / P2WPKH reveal it on spend; address reuse further tightens transaction linkage.
4. Do you need to hide spending conditions? A P2TR key path hides the internal key and script tree, but the output key `Q` is always on chain. Script privacy and public-key exposure are not the same concern.

**Multiparty and SIGHASH**

5. Is the multisig really n-of-n or actually t-of-n? MuSig2 directly supports only the former. Are the interaction rounds and implementation audits clearly understood?
6. Does SIGHASH need a non-default mode? Legacy / SegWit v0 default to ALL, Taproot to DEFAULT. Without a clear requirement, stay away from NONE / SINGLE / ANYONECANPAY.
7. Does the multiparty contract verify the Taproot internal key, aggregation coefficients, Merkle root, and tweak? Naive key addition or trusting a coordinator invites rogue-key and substitution risks.

---

Bitcoin's signature landscape evolves by accumulation, not replacement: new schemes lean on Schnorr, the script layer moves toward minimum disclosure per spending path, and SIGHASH keeps patching and extending its commitment scope. The real divergence is in multiparty coordination protocols — where both the fee and privacy leverage and the new attack surface live.
