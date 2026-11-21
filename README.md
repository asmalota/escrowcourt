# EscrowCourt

**Escrowed disputes, ruled on-chain — with the money actually held and actually paid out.**

A buyer escrows real GEN into the contract when opening a dispute, so the amount is
custody rather than a number in a description. The named seller opts in; until they do,
the buyer can pull the escrow back. Both sides then file evidence URLs, and an on-chain
jury fetches those pages and rules how the escrow is released.

- **Primary tag:** Dispute Resolution
- **Sub-tags:** Evidence Assessment · Escrow Claims
- **Network:** GenLayer Studionet (chain 61999)
- **Contract:** [`0xA9AA7c4Cb0f4D39563A049Cbe01dFF8F17cD68a7`](https://explorer-studio.genlayer.com/address/0xA9AA7c4Cb0f4D39563A049Cbe01dFF8F17cD68a7)
- **Deploy tx:** `0x482f59d531bd873395b7154ad3195eb125818ae191307b7d9ead47f8bf54705f` — ACCEPTED, MAJORITY_AGREE, `execution_result` SUCCESS
- **Size:** 17,063 bytes · **10 public methods** · **18 direct-mode tests passing**
- **Admin:** none. There is no privileged role anywhere in this contract.

---

## The lifecycle

```
OPEN (payable, GEN held by the contract)
  → seller accepts          → ACTIVE
  → both sides file evidence
  → request_ruling          → RULED   (jury read the pages)
  → settle                  → SETTLED (real GEN transfer, once)

OPEN → refund_if_unaccepted → buyer gets the escrow back if the seller never shows
```

| Method | Who | What it enforces |
|---|---|---|
| `create_dispute(seller, description)` | buyer, **payable** | the escrowed amount is `gl.message.value`; buyer ≠ seller |
| `accept_dispute(id)` | the named seller only | nobody else can opt in |
| `refund_if_unaccepted(id)` | buyer only | the escape hatch while the dispute is still `OPEN` |
| `submit_evidence(id, url, description)` | active parties only | a URL **and** a description are required |
| `request_ruling(id)` | anyone | refuses to run with **zero** evidence on file |
| `settle(id)` | anyone | pays out once; a second settle reverts |

## How the verdict is derived

The jury returns a pair — a winner and a percentage — and the two must agree. That
consistency rule is the whole security property, and it is applied in three places:

```
canon(winner, pct):
    "buyer"  -> ("buyer", 100)     exactly everything to the buyer
    "seller" -> ("seller", 0)      exactly nothing to the buyer
    "split"  -> ("split", pct)     only when 0 < pct < 100
    anything else, or split at 0/100 -> None  (refused)
```

1. **Every validator** runs the jury again from its own fetch of the same evidence and
   requires the same winner. On a split it also refuses a share that drifts by more than
   10 points between readings — a split of 40% and a split of 80% are not the same verdict.
2. **The contract**, after consensus, refuses a payload missing any required key
   (`Malformed verdict payload`) and a payload whose winner contradicts its own percentage
   (`Verdict contradicts its own findings`). A leader saying "split, 100% to the buyer" is
   rejected by the contract itself, not merely by validators — there is a test for it.
3. **Anyone** can check the rule without running consensus:
   `check_verdict(winner, split_percent, leader_winner, leader_percent)` returns the same
   boolean the consensus stops on. Eight live calls below.

Judging rules the jury is given, in the contract's own words: judge only on the fetched
evidence, not on whoever wrote more text; a party that filed no evidence must not win on
assertion alone; use `split` with a fair percentage when fault is shared.

## Money

- **Real custody.** `create_dispute` is `@gl.public.write.payable` and the escrow is
  `gl.message.value`. Verified live: the contract balance went **0 → 0.5 GEN** on creation.
- **Real payout.** `settle` calls `emit_transfer(..., on='finalized')`. `on='finalized'` is
  not optional: without it no message is emitted and the payout silently does nothing.
  Verified live: after finalization the buyer's balance returned to its pre-escrow value
  and the contract balance went back to **0**.
- **Split pays both sides**, and the shares always sum to the escrow exactly.
- **Exactly once.** `settled` is set before the transfer is emitted, so a second `settle`
  reverts rather than double-paying.
- **The buyer is never trapped.** If the seller never accepts, `refund_if_unaccepted`
  returns the whole escrow.

## Test coverage — 18 tests, all passing

```
tests/direct/test_escrow_court.py ........................ 18 passed
```

| Area | Tests | What they defend |
|---|---|---|
| Escrow funding | 3 | unfunded creation refused, escrow is custody not metadata, buyer ≠ seller |
| Acceptance | 3 | only the named seller accepts, evidence blocked until accepted, buyer refunds if never accepted |
| Evidence | 1 | a ruling is refused with zero evidence on file |
| Ruling | 3 | a full flow pays the winner, a split records its share, no double or early settle |
| **Verdict rule** | **5** | `buyer` is exactly 100, `seller` exactly 0, `split` strictly between, unknown winner refused, two independent readings compared |
| **Contract refusals** | **2** | a self-contradictory verdict (`split`, 100%) is refused by the contract; a malformed payload is refused |
| Split bounds | 1 | 40% is recorded as a split; the share follows the rule |

## Verified live on Studionet

Two funded accounts: the buyer (wallet holding the escrow) and the seller.

| # | Call | Result | Money | Tx |
|---|---|---|---|---|
| 1 | `gen_getContractSchema` | **10 methods** | — | — |
| 2 | `create_dispute(seller, …)` escrow 0.5 GEN | **return** | contract **0 → 0.5 GEN** | `0xab0765fe9d2e74d2779989e36bfce36147f297a0304a9e2513ae7ea10344f4d0` |
| 3 | `accept_dispute(1)` by the seller | **return** | — | `0x89320a8123e1d8fe3d8c351104fe09ecafd3472a1e5542ebf051445deea7d3ff` |
| 4 | `submit_evidence` (buyer) | **return** | — | `0xcfd07ac5f8b9b2107960e0f72ae5ce9bccb5a71d4b8dc6fc8471ace4fa083734` |
| 5 | `submit_evidence` (seller) | **return** | — | `0x8e38b68b1d262bf6dcc54d8fa15fd9bf79d726e338b99b1d7ef2938bcf51d424` |
| 6 | `request_ruling(1)` | **return** — winner **buyer**, 100% | — | `0x33561d544f16028d052f904b45a8d86295734bb208626f4a999dfd8ddaa31911` |
| 7 | `settle(1)` | **return**, `settled: true` | contract **0.5 → 0**, buyer restored to 20.0000 | `0xcff356c244d800dd814a947691217b2f27a22e05ed762a73413479ca68646df4` |

The jury's own reason, verbatim, for ruling against the seller:

> "The seller's statement offers no verifiable proof of delivery, such as a transfer log,
> checksum, or receipt from the buyer's endpoint. It also concedes that the link now
> returns 404 and that the buyer cannot access the d…"

That is the substantive judgement the design is meant to produce: not a keyword match, but
a reading of what each side actually put in front of it.

Balance transcript around settlement:

```
before:  buyer 20.0000 | seller 23.5800 | contract 0.0000
escrow:  buyer 19.5000 | seller 23.5800 | contract 0.5000
settled: buyer 20.0000 | seller 23.5800 | contract 0.0000   (transfer finalizes)
```

### `check_verdict` live results — 8/8 as intended

| winner | split | leader winner | leader split | result |
|---|---|---|---|---|
| buyer | 100 | buyer | 100 | **true** |
| seller | 0 | seller | 0 | **true** |
| buyer | 100 | seller | 0 | **false** |
| split | 100 | split | 100 | **false** |
| split | 0 | split | 0 | **false** |
| split | 40 | split | 40 | **true** |
| split | 40 | split | 60 | **false** |
| neither | 50 | neither | 50 | **false** |

## Known platform behaviour

Two things that cost time and are worth writing down:

**`emit_transfer` needs `on='finalized'`.** Without it the transfer is never emitted and
the payout silently does nothing while the transaction still reports success. With it, the
transfer lands when the transaction finalizes — a few tens of seconds later — so a balance
read immediately after `settle` still shows the old figure. Read the balance again.

**Gas estimation starves calls that are expected to revert.** The SDK's
`estimateTransactionGas` throws on such calls and falls back to `200_000n`, which then
kills legitimate calls too. Pinning it (`client.estimateTransactionGas = async () => 3M`)
is what makes a verification script that deliberately includes refusals run at all.

## Repository layout

```
contracts/escrow_court.py            the contract (17,063 bytes, 10 public methods)
tests/direct/test_escrow_court.py    18 tests, incl. the verdict-rule suite
index.html                           the dApp (wallet-pinned genlayer-js client)
evidence_buyer.md                    sample buyer statement for the live run
evidence_seller.md                   sample seller statement for the live run
genlayer.config.yaml                 network config
ruff.toml                            lint config
```

## Running it

```bash
python -m pytest tests/direct/ -v        # 18 tests
```

Deploy (Studionet, chain 61999):

```js
import { createClient, createAccount } from "genlayer-js";
import { studionet } from "genlayer-js/chains";

const client = createClient({ chain: studionet, account: createAccount(PRIVATE_KEY) });
client.estimateTransactionGas = async () => 3_000_000n;
const hash = await client.deployContract({ code, args: [] });
```

The contract address of a Studio deploy is `tx.recipient` on the deploy transaction.

## What this contract deliberately does not do

- It does not authenticate the parties against real-world identities. It authenticates
  that **this** buyer escrowed **this** GEN against **this** named seller.
- It does not decide who is telling the truth in general — only how the escrow is released
  on the evidence both sides actually filed.
- It cannot be appealed. The verdict is final once ruled; a second settle is refused.
