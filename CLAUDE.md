# contracts — agent notes

Keep this current when the decisions below change.

---

## PriceTimePriority is the default matching mode. SizePriority jams.

**Every new pair must be created with `ExchangeOrderbook.MatchingMode.PriceTimePriority`.**
Changed 2026-08-06; before that every `addPair` call site in `script/` hardcoded
`SizePriority`, so the vulnerable mode was the de-facto default on every chain.

### Why

`ExchangeOrderbook._insertId` branches on the mode:

```solidity
if (mode == MatchingMode.PriceTimePriority) {
    _insertFifo(self, price, id);   // appends via self.tail[price] — O(1)
    return;                          // ← returns BEFORE the traversal
}
...
while (head != 0) { ... }            // SizePriority: walks the queue — O(depth)
```

SizePriority keeps one price level's resting orders sorted by deposit size, and there is
no index into that list, so every insert walks it. `_deleteOrder` walks it too. Nothing
counts or caps those traversals — `maxMatches` / `TooManyMatches` bound the **matching**
loop only, which is a different loop.

Measured, `test/exchange/orderbook/MatchingModeQueueCost.t.sol`:

| mode | 1 order at the price | 300 orders at the price |
|---|---|---|
| SizePriority | 154,197 gas | **347,589 gas** |
| PriceTimePriority | 151,207 gas | **151,250 gas** |

SizePriority grows ~645 gas per resting order, without limit. PTP is flat — 43 gas of
drift across 300 orders.

### The failure is silent, which is the actual problem

Past roughly 850 orders at one price the insert exceeds a wallet-realistic gas cap and
the transaction dies **out-of-gas, returning ZERO bytes**. Not `TooManyMatches`, not
`OrderSizeTooSmall` — nothing for viem to decode and nothing for the UI to explain. See
`test/exchange/orderbook/PoC_UnboundedOrderQueue.t.sol`, which pins this with a 700k cap.

The chain is not the constraint: RISE's block gas limit is 1.5B. The **wallet's** cap is.

A deep single-price queue is not exotic — it is what a market maker quoting one level,
or any incentive that rewards resting at the touch, produces naturally.

### What this does NOT fix

- **Mode is immutable.** `Orderbook.matchingMode` is set in `initialize` and there is only
  `getMatchingMode()`. **A pair created with SizePriority is vulnerable forever.** The RISE
  SMKB/SMKQ pair (`0x19eB4a46…`) is mode 0 and stays that way.
- **`_deleteOrder` is still O(depth) in SizePriority**, so cancelling on an existing
  SizePriority pair degrades the same way.
- SizePriority is still reachable and still tested — this changes the default, not the
  capability. If a venue ever wants size-priority matching it needs a real index into the
  queue first, not the current linear list.

### Related, unfixed

`test/exchange/orderbook/PoC_DustEatsMatchBudget.t.sol` — dust orders consuming the match
budget. Not addressed here.

---

## `maxMatches` bounds matching, and is now readable

`maxMatches` is 20 (set in `MatchingEngine.initialize`). Passing `n` above it **reverts the
whole transaction** with `TooManyMatches(n)` — it does not clamp. Verified against the live
RISE engine: `n=21` reverts with nothing executed; `n=20` fills exactly 20 levels of a
25-level book and leaves 5 resting.

It was `private` with no getter, so a client had to hard-code 20 and hope nobody called
`setMaxMatches` — which takes any `uint32`, emits no event, and is the only thing that
changes it. It is now `public`. **That getter is not deployed on RISE**; the engine is
deployed directly rather than behind a proxy, so it ships with the next generation.

Clamping instead of reverting would be friendlier — a partial fill plus a resting
remainder is what the swap card already models — but it is a behaviour change and has not
been made.

## MatchingEngine has 166 bytes of headroom

24,410 against EIP-170's 24,576. That margin is the tightest constraint on this contract:
measure any addition before writing it. `forge build --sizes` runs in CI
(`.github/workflows/contracts.yml`) for this reason.

## foundry.toml is tracked, and its settings are not arbitrary

solc **0.8.24**, optimizer on, **runs = 200**. Recovered by matching
`forge inspect MatchingEngine deployedBytecode` against the live contract — 52 of 24,347
bytes differ, being the library link address and the metadata hash. It was gitignored and
uncommitted until `ba70353`, which meant a fresh clone could not build at all: with no
`foundry.toml`, forge resolves the project root to the repo root and no `forge-std` import
resolves.

`contracts/` is deliberately **not** in the pnpm workspace and has its own npm tree —
`remappings.txt` maps `@lukso/` into `contracts/node_modules`, so `npm install` must run
before `forge build` or `src/**/TransferHelper.sol` cannot resolve `ILSP7DigitalAsset`.
