---
title: "How Proof of Reserves (PoR) Actually Works"
slug: "proof-of-reserves"
date: 2026-11-02T00:30:00+08:00
draft: false
author: "Jackie"
tags: ["proof-of-reserves", "cryptography", "exchange-security", "blockchain", "cryptocurrency"]
categories: ["share-everyday"]
summary: "A technical breakdown of Proof of Reserves: Merkle liability commitments, signed reserve addresses, what each party can verify, and the trust assumptions that remain."
showtoc: true
cover:
  image: "images/proof-of-reserves-flow.svg"
  alt: "Proof of Reserves structure: assets (signed addresses, unencumbered) and liabilities (Merkle root, zk sum proof)"
  relative: false
images:
  - "images/proof-of-reserves-og.png"
---

Most exchanges' **Proof of Reserves** page is essentially a list of wallet addresses plus a green checkmark. That is the easy half. The real question is: can the custodian prove that the assets it controls are at least the sum of all user balances, without publishing everyone's balance?

A complete PoR consists of two independent parts:

1. **Proof of assets**: the custodian actually controls these addresses, and the assets are free — not borrowed, pledged, or lent out.
2. **Proof of liabilities**: the published total liabilities equal the sum of all user balances, and every user can verify that their own balance is included.

Only when both hold does `assets ≥ liabilities` mean anything.

## Freezing the liability snapshot

At a fixed block height `h` (or timestamp), freeze the account database. Every account that constitutes a user claim must be included: spot, margin, futures PnL, earn products, pending withdrawals, internal accounts. Negative balances must be netted, and derivative liabilities need an explicit mark-to-market rule. This step involves almost no cryptography, yet it is the easiest one to get wrong: omit a single account type and the entire proof is false.

## Committing each balance into a leaf

For account `i`:

```text
leaf_i = H("leaf" || user_id_i || salt_i || balance_i)
```

- `salt_i` is unique to each user and kept private. Without it, anyone could brute-force the hashes of known balances and enumerate accounts.
- `H` is a collision-resistant hash, such as SHA-256 or Keccak-256.
- Domain separation (`"leaf"` vs `"node"`) prevents second-preimage attacks in which an internal node is passed off as a leaf.

Adjacent leaves are hashed pairwise, layer by layer, and finally the Merkle root `R` is published. The tree construction must be unambiguous: a fixed left/right order and an explicit rule for odd nodes (promote them, or hash them with a zero leaf). Ambiguity enables duplicate-leaf attacks and tree-shape attacks.

A tree with `n` accounts has depth `ceil(log2 n)`; for a million accounts, an inclusion proof is only about 20 hashes.

## Publishing the root and distributing proofs

The operator signs `(R, h, total_liabilities)`. Each user receives `(balance, salt, user_id, sibling path)`, recomputes their own leaf, folds upward along the path, and compares the result with the signed root. If any leaf changes, every path passing through it changes.

This proves **inclusion**: **my balance is in this committed set**. By itself, it proves neither that the set contains all users nor that the published total is the true sum. Those are two separate claims.

## Proving the assets

The asset side is a separate procedure:

- Publish the addresses holding the reserves.
- Sign a challenge message with each address's private key, so that a copied address list cannot be passed off as ownership.
- Read balances at the same block height `h` used for the liability snapshot.

The sum of assets must cover liabilities, but **controlling an address** is not the same as the assets being **unencumbered**. Coins that have been lent out, pledged as collateral, staked, or deposited with a third-party custodian may still appear in on-chain addresses controlled by the exchange — yet they are not free reserves. Off-chain fiat and receivables typically fall outside the scope of most PoRs.

## What each party can actually verify

| Party | What they can verify |
| --- | --- |
| User | Leaf hash, Merkle path, signed root → their own balance is included |
| Auditor | Root, address signatures, database reconciliation, total liabilities, completeness of account types |
| Public | Root signature, timestamp, address control, asset and liability totals |

Users cannot verify the total; the public cannot verify that **no account was omitted**. This gap is structural.

## Where PoR fails

- **Borrowed reserves / window dressing**: borrow coins before the snapshot and return them afterwards. A point-in-time proof is not incompatible with insolvency.
- **Incomplete liabilities**: hidden accounts, undisclosed negative balances, derivative exposure, pending withdrawals, internal market-making accounts.
- **Encumbered assets**: pledged or lent-out assets still show up on-chain.
- **Timing mismatch**: assets read at height `h`, liabilities at another time.
- **Weak commitment construction**: no salt → enumerable; no domain separation → forgeable proofs; ambiguous tree shape → duplicate leaves.
- **Reliance on the auditor**: an attestation is only as credible as its scope and independence.
- **Stale proofs**: a root from six months ago says nothing about today.

## Stronger schemes

**zk-PoR / proof of solvency.** Instead of publishing every leaf, balances are committed with a hiding commitment (e.g., Pedersen), followed by a zero-knowledge proof that all committed balances sum to the published total and that every balance is non-negative (a range proof). This hides individual balances, in some designs removes the need to issue a Merkle path to every user, and cryptographically binds the total instead of trusting the operator's own arithmetic. But the completeness problem remains: unless you can inspect the database, no proof can show that some account was not excluded.

**Continuous proofs.** Updating signed roots more frequently, publishing reserve addresses on-chain, and randomly challenging users can shrink the window for window dressing — but cannot eliminate it.

## Binance's PoR in practice

After FTX collapsed in November 2022, Binance was among the first to launch a public PoR page. The initial version covered only BTC, committing user balances with a plain Merkle tree. In February 2023, Binance upgraded the system to a **zk-SNARK version and open-sourced it** — a real-world deployment of the zk-PoR scheme described above:

- **Liability side**: each user balance is a leaf; after logging in, users can obtain their own Merkle path and recompute up to the published root — a standard inclusion proof.
- **Total side**: a zk-SNARK circuit proves two things — the sum of all leaves equals the published total liabilities, and no negative balance exists. This cryptographically locks down the "total" and "no negative balances," rather than relying on the operator's own arithmetic.
- **Asset side**: a set of reserve wallet addresses is published, balances are read on-chain, and a reserve ratio is reported per asset, claimed to be no less than 100% for each.
- **Frequency**: snapshots are updated monthly, with coverage expanded from the initial BTC to dozens of assets.

But it also happens to land squarely on the failure points listed above:

- **No independent audit**: Binance initially hired Mazars to issue an attestation report; after Mazars suspended all work for crypto clients in December 2022, PoR became pure self-attestation.
- **Limited scope**: it covers only users' on-platform assets, not the company's own loans, intra-group liabilities, or fiat solvency — `assets ≥ liabilities` holding does not imply that "the company is solvent."
- **Point-in-time snapshots**: within the 30 days between monthly snapshots, window dressing cannot be ruled out.

Binance's practice illustrates PoR's real boundary: Merkle inclusion proofs and the zk-SNARK total proof are the reliable cryptographic parts, while account completeness, whether assets are unencumbered, and the company's overall liabilities remain outside what cryptography covers.

## The 5-min checklist

1. Is there a signed root with a clear timestamp and block height?
2. With the salt and path you were given, does your leaf recompute to that root?
3. Are the reserve addresses signed, and were balances read at the same height?
4. Does `total assets ≥ total liabilities` hold, and are borrowed or pledged assets excluded?
5. Does the audit report specify which account types and asset types are in scope?
6. Is the proof recent enough, with a public update schedule?

---

PoR is a **cryptographic inclusion proof** plus a **signature-based control proof**. The cryptography is the reliable part; completeness and encumbrance still depend on process, audit scope, and trust.
