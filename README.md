# qidx — DEX Settlement Engine

Batch settlement is the core primitive that perp DEXes like dYdX and Drift are built on. This is that primitive, implemented natively on Solana.

A full-stack settlement engine: atomic batch settlement on-chain + a price-time priority order book + a transaction indexer.

**Live on devnet** | Program: [`8omCC2Q9SwwfRJQNkJ9UnFairpzHFkaWSeEd5nXjcooy`](https://explorer.solana.com/address/8omCC2Q9SwwfRJQNkJ9UnFairpzHFkaWSeEd5nXjcooy?cluster=devnet)

**Live APIs**
- Indexer: `https://qidx-production.up.railway.app`
- Matcher: `https://settlement-production-b250.up.railway.app`

**Repos**
- [`mahelikaa/qidx`](https://github.com/mahelikaa/qidx) — indexer
- [`mahelikaa/settlement`](https://github.com/mahelikaa/settlement) — Anchor program + matcher

---

## Architecture

```
packages/
├── indexer/     Node.js REST API — decodes any settle_batch transaction
└── matcher/     TypeScript — price-time priority order book + settlement queue

programs/
└── settlement/  Rust/Anchor — settle_batch on-chain program
```

```
User (maker)             User (taker)
     │  POST /order (sell)     │  POST /order (buy)
     └──────────┬──────────────┘
                │
        ┌───────▼────────┐
        │  Matcher API   │  TypeScript, port 4000
        │  Order Book    │  price-time priority matching
        └───────┬────────┘
                │  settle_batch tx
                ▼
        ┌───────────────┐
        │  Solana devnet │
        │  settle_batch  │  Anchor program (Rust)
        └───────┬────────┘
                │  tx signature
                ▼
        ┌───────────────┐
        │  qidx API     │  Node.js, port 3000
        │  GET /tx/:sig │  decode → structured JSON
        └───────────────┘
```

---

## Why batch settlement

Most DEXes settle one trade per transaction. `settle_batch` is atomic over N trades — either all settle or none do.

| Approach | Trades/tx | CUs total | CUs/trade |
|---|---|---|---|
| One-by-one (anchor-spl) | 1 | 14,644 | 14,644 |
| settle_batch N=1 | 1 | 6,214 | 6,214 |
| settle_batch N=4 | 4 | 19,381 | 4,845 |
| settle_batch N=32 (projected) | 32 | ~55,000 | ~1,700 |

All numbers measured on devnet. **57% CU reduction at N=1** by replacing `anchor-spl::token::transfer` with raw CPI using the SPL Token wire format directly. At N=4, cost per trade drops to 4,845 CUs as batch overhead amortises.

---

## Quick start

### Prerequisites
- Node.js 20+
- Rust + Anchor 1.0
- Solana CLI with a devnet keypair

### Install
```bash
git clone https://github.com/mahelikaa/qidx-mono
cd qidx-mono
npm install
```

### Run the indexer
```bash
# .env: RPC_URL=https://api.devnet.solana.com
cd packages/indexer
node index.js
# → http://localhost:3000
```

### Run the matcher
```bash
# .env: RPC_URL=... ENGINE_KEYPAIR_PATH=~/.config/solana/id.json
cd packages/matcher
npx ts-node matcher.ts
# → http://localhost:4000
```

### Run the end-to-end demo (N=4 batch)
```bash
# requires matcher running on :4000
cd packages/matcher
npx ts-node demo.ts
```

Creates 4 maker/taker pairs on devnet, mints tokens, places 8 crossing orders, waits for the batch to flush, verifies balances.

---

## On-chain program

```rust
pub fn settle_batch(ctx: Context<SettleBatch>, trades: Vec<Trade>) -> Result<()>

pub struct Trade {
    pub base_amount: u64,   // base tokens: maker → taker
    pub quote_amount: u64,  // quote tokens: taker → maker
}
```

**Remaining accounts per trade (4 × N):**
1. `maker_base_account` — maker sells this
2. `taker_base_account` — taker receives this
3. `taker_quote_account` — taker pays this
4. `maker_quote_account` — maker receives this

**Validations:** batch not empty, ≤32 trades, amounts > 0, account count = 4N, token program must be SPL Token.

### Why raw CPI

`anchor-spl::token::transfer` allocates a `CpiContext` on every call. Instead, the 9-byte SPL Token Transfer instruction is built by hand:

```
[0]     u8   discriminator = 3
[1..8]  u64  amount (little-endian)
```

Then `solana_program::invoke` is called directly. Same wire format, zero framework overhead. **Result: 14,644 → 6,214 CUs (-57%) on devnet.**

---

## API

### Indexer — `GET /tx/:signature`

```bash
curl https://qidx-production.up.railway.app/tx/55usB2Dp3A81YAriq1pwL4C5BHPU1MAHojESBNd8B3933Z6p1hxETqjgSKsQehbQpczd9zwUtpBE1aUTs1siEbVQ
```

```json
{
  "signature": "55usB2Dp...",
  "slot": 465788293,
  "timestamp": 1780081837,
  "fee": 5000,
  "compute_units_used": 6214,
  "instructions": [
    {
      "instruction": "settle_batch",
      "program": "8omCC2Q9SwwfRJQNkJ9UnFairpzHFkaWSeEd5nXjcooy",
      "trade_count": 1,
      "trades": [
        {
          "base_amount": "1000000",
          "quote_amount": "500000",
          "maker_base": "5p9jHDeYK...",
          "taker_base": "CoGs7NHW...",
          "taker_quote": "4TSZQPT...",
          "maker_quote": "AbcvADq..."
        }
      ]
    }
  ],
  "token_balance_changes": [
    { "account": "5p9jHDeYK...", "mint": "84he3Lph...", "change": "-1000000" },
    { "account": "CoGs7NHW...", "mint": "84he3Lph...", "change": "1000000" }
  ]
}
```

### Matcher — `POST /order`

```bash
curl -X POST https://settlement-production-b250.up.railway.app/order \
  -H "Content-Type: application/json" \
  -d '{
    "side": "sell",
    "baseMint": "<mint>",
    "quoteMint": "<mint>",
    "baseAmount": "1000000",
    "quoteAmount": "500000",
    "makerBaseAccount": "<ATA>",
    "makerQuoteAccount": "<ATA>"
  }'
```

**`GET /orderbook`** — open bids/asks  
**`GET /trades`** — matched trade history  
**`GET /health`** — engine pubkey, program, cluster

---

## Decentralisation tradeoff

| Component | Decentralised? |
|---|---|
| Settlement (on-chain) | ✅ Yes — trustless, atomic |
| Matching (off-chain) | ❌ No — centralised server |

This is the standard CLOB architecture used by dYdX, Drift, and early Serum. The trust assumption is on the matcher, not the settlement.

---

## Live proof

**N=4 batch** — 4 trades, 1 tx, 19,381 CUs, 16 balance changes:  
[`sP4GKmvSFtUh...`](https://explorer.solana.com/tx/sP4GKmvSFtUh4CmxgV8QRBSGTapeF9SnUDgZttenu3mkiRzEra4GhEEs8QExjkEYT6d4Vk7SCuQpYGfhHFvAjjY?cluster=devnet)

Decoded by qidx:  
[`https://qidx-production.up.railway.app/tx/sP4GKmv...`](https://qidx-production.up.railway.app/tx/sP4GKmvSFtUh4CmxgV8QRBSGTapeF9SnUDgZttenu3mkiRzEra4GhEEs8QExjkEYT6d4Vk7SCuQpYGfhHFvAjjY)

---

Built for Solana Fellowship Q2 2025 | MIT License
