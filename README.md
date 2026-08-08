# SentiSwap

**A natural-language, sentiment-gated DeFi trading agent.**

You send a plain-English instruction to a Telegram bot — *"Buy 2 XRP if it's trending and gas is under 30 gwei"* — and SentiSwap turns it into a structured trade intent, watches Reddit sentiment and Ethereum gas prices until the conditions hold, then executes the swap on-chain against a Uniswap-V2-style router deployed to the Sepolia testnet.

Built as a CS 473 final project. Everything runs against testnet with mock tokens; no real funds are ever at risk.

---

## What it does

```
Telegram message
      │
      ▼
 GPT-4o parser ──────────► parsed_command.json
 (natural language          {action, token, amount,
  → JSON intent)             conditions: {reddit_trending,
      │                                   gas_price_threshold}}
      ▼
 Scheduler loop (every ~10s)
      ├── Reddit monitor  → scan 10 crypto subreddits, VADER sentiment
      │                     filter, count positive mentions of the token
      └── Gas monitor     → Etherscan gas oracle, current gwei
      │
      ▼  both conditions satisfied
 Executor (web3.py)
      ├── buy  → swapETHForExactTokens on the mock router
      └── sell → approve, then swapExactTokensForETH
      │
      ▼
 Telegram notification with the Sepolia Etherscan tx link
```

### The pieces

**1. Natural-language intent parsing** (`core_logic/parser.py`)
A GPT-4o call with a constrained system prompt converts free-form text into a strict JSON object: `action` (buy/sell), `token`, `amount`, and a `conditions` block. The parser strips markdown code fences from the model's response and validates it parses as JSON before handing it downstream. Defaults to `0.001` when no amount is given.

**2. Telegram interface** (`bot/handlers.py`, `scripts/run_bot.py`)
A `python-telegram-bot` long-polling app. Any text message becomes a trade intent; the parsed result is written to `parsed_command.json` for the scheduler to pick up, and the sender's `chat_id` is cached so the executor can push a confirmation back to the same conversation later.

**3. Reddit sentiment signal** (`core_logic/reddit_monitor.py`)
Polls the 10 newest posts from ten crypto subreddits (r/CryptoCurrency, r/CryptoMoonShots, r/XRP, r/defi, and others) via PRAW. Posts whose titles mention the token are scored with VADER; only those with a compound sentiment above `+0.2` count. Matches are kept in a per-token sliding time window (24h by default) with post-ID deduplication, so the "is it trending" question is really *how many positive posts about this token appeared recently*.

**4. Gas price signal** (`core_logic/gas_monitor.py`)
Reads `ProposeGasPrice` from the Etherscan gas oracle. Returns `-1` on failure so a network blip can't be mistaken for cheap gas.

**5. Condition scheduler** (`scripts/scheduler.py`)
The control loop. Clears any stale command on startup, then polls every 10 seconds: refresh the signals, check them against the intent's thresholds, and fire the trade the first time both line up. Sends a Telegram notification and exits after execution.

**6. On-chain execution** (`core_logic/executor.py`)
Signs and submits transactions with `web3.py` on Sepolia (chain ID `11155111`). Buys call `swapETHForExactTokens` with the ETH value computed from the router's fixed rate and a 5-minute deadline. Sells first send an ERC-20 `approve`, wait for that receipt to confirm, then call `swapExactTokensForETH` with a slippage floor.

**7. Mock DEX contracts** (`contracts/`)
Rather than depend on live Sepolia liquidity, the exchange itself is deployed as part of the project:

| Contract | Role |
| --- | --- |
| `XRP.sol` | Minimal OpenZeppelin ERC-20 ("Test XRP") standing in for the traded token |
| `WETH9.sol` | Trimmed WETH implementation — deposit, withdraw, approve, transfer |
| `UniswapV2Router02.sol` | Simplified router with `swapETHForExactTokens` / `swapExactTokensForETH` at a fixed 1 ETH = 1000 XRP rate, path validation, deadline checks, and ETH refunds |
| `UniswapV2Factory.sol` | Pair-creation scaffolding |

Deployed addresses (Sepolia) are hardcoded in `core_logic/executor.py`.

**8. Helper scripts** (`debugging/`)
`SendToRouter.py` seeds the router with test XRP so it has inventory to sell; `RouterBalance.py` reads that balance back.

**9. Result snapshot** (`core_logic/index.html`)
A static page documenting one full end-to-end run: the five Reddit posts that tripped the trending condition, the gas price at the time (0.62 gwei), and the resulting confirmed Sepolia transaction.

---

## Setup

Requires Python 3.10+.

```bash
git clone <this-repo>
cd sentiswap
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
```

Create a `.env` in the project root:

```ini
# LLM parsing
OPENAI_API_KEY=

# Telegram bot
TELEGRAM_API_KEY=

# Reddit (script-type app credentials)
REDDIT_CLIENT_ID=
REDDIT_SECRET=
REDDIT_USER_AGENT=

# Ethereum / Sepolia
RPC_URL=
PRIVATE_KEY=
PUBLIC_ADDRESS=
ETHERSCAN_API_KEY=

# Optional — Twitter monitor (not wired into the trade decision)
TWITTER_BEARER_TOKEN=
```

Use a throwaway testnet wallet. The private key signs transactions directly.

## Running it

Both processes read and write `parsed_command.json` and `chat_id.txt` by relative path, so **run them from the repository root**, in two terminals:

```bash
# terminal 1 — Telegram listener
python scripts/run_bot.py

# terminal 2 — condition scheduler / executor
python scripts/scheduler.py
```

Then message the bot:

```
/start
Buy 2 XRP if it's trending on Reddit and gas is below 30 gwei
```

The scheduler logs each poll and messages you back with an Etherscan link once the trade lands.

---

## Notes and known limitations

This is a course project, and a few rough edges are worth naming rather than hiding:

- **Only `XRP` executes.** The parser accepts any ticker and the monitors will track it, but `execute_trade` rejects anything but the mock XRP token, since that's the only one with a deployed pair.
- **The trending threshold is effectively off.** `scheduler.py` checks `post_count >= 0`, which is always true when the `reddit_trending` condition is present. The counting and sentiment machinery is real; the comparison was loosened to make live demos reproducible.
- **`check_conditions` is called twice per iteration**, so the Reddit scan runs twice as often as it needs to.
- **Fixed exchange rate.** The mock router prices at a constant 1 ETH = 1000 XRP with no pool math, so there is no price impact and slippage protection is nominal.
- **The Twitter monitor is dead code.** `twitter_monitor.py` works and is imported, but its tweet counts never feed the trade decision — the Reddit signal replaced it after Twitter's API access changed.
- **`parsed_command.json` and `chat_id.txt` are runtime state** that ended up committed to the repo. They're regenerated on each run and belong in `.gitignore`.
- **Single-command, single-shot.** One intent at a time, and the scheduler exits after the first trade.

## Stack

Python · OpenAI GPT-4o · python-telegram-bot · PRAW · vaderSentiment · web3.py · Solidity · Ethereum Sepolia · Etherscan API

## License

MIT — see [LICENSE](LICENSE).
