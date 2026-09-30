# LPSignal SDK

Official clients for [LPSignal](https://lpsignal.app): net-of-impermanent-loss APR signals for blue-chip
concentrated-liquidity pools (Uniswap v3/v4, PancakeSwap v3, Aerodrome and Velodrome Slipstream) on Ethereum,
BNB Chain, Base, Arbitrum, Optimism and Polygon. Available for **Node.js** and **Python**.

[中文说明](README.zh.md)

| | Package | Docs |
|---|---|---|
| Node.js ≥ 20 | `npm install lpsignal` | [node/](node) |
| Python ≥ 3.10 | `pip install lpsignal` | [python/](python) |

Both SDKs cover the same things:

- **REST API**: pools ranked by net APR, per-range metrics, hourly history, range backtests, signals and their
  7-day outcomes, the smart-LP leaderboard, your account, webhook and Telegram setup, billing links.
- **A signal stream that never misses a signal.** The server's WebSocket replays the last 24 hours on reconnect.
  The SDK also fetches everything newer than your last signal over REST before every connection, so an outage of
  any length is filled. Signals arrive in id order, once each while the process runs. Save the last id (a file
  store is included) and a restart resumes where it stopped; a crash right after your handler finished can offer
  that one signal again, so make the handler idempotent on `signal.id`.
- **Webhook verification**: checks the `x-lpsignal-signature` HMAC and the timestamp, and returns the parsed body.

## Quick start

```ts
import { LPSignal, SignalStream, FileLastIdStore } from 'lpsignal';

const lps = new LPSignal({ apiKey: process.env.LPSIGNAL_API_KEY });

const { pools } = await lps.pools({ chain: 'base', class: 'volatile', limit: 10 });
for (const p of pools) console.log(p.pair, p.dex, `${(p.best.netApr * 100).toFixed(1)}%`, `±${p.best.rangeBp / 100}%`);

const stream = new SignalStream({
  client: lps,
  store: new FileLastIdStore('./lpsignal-state.json'),
  onSignal: async (signal) => {
    if (signal.kind === 'net_apr') console.log(`open ${signal.pair} ticks ${signal.tickLower}..${signal.tickUpper}`);
  },
});
await stream.start();
```

```python
import asyncio, os
from lpsignal import AsyncLPSignal, FileLastIdStore, SignalStream

async def main():
    lps = AsyncLPSignal(api_key=os.environ["LPSIGNAL_API_KEY"])
    page = await lps.pools(chain="base", pair_class="volatile", limit=10)
    for p in page["pools"]:
        print(p["pair"], p["dex"], f"{p['best']['netApr']:.1%}")

    def on_signal(signal, source):
        if signal["kind"] == "net_apr":
            print("open", signal["pair"], signal["tickLower"], signal["tickUpper"])

    await SignalStream(lps, on_signal, store=FileLastIdStore("lpsignal-state.json")).run()

asyncio.run(main())
```

## Units

- APRs and ratios are fractions: `0.345` is 34.5%.
- `ilApr` and `il7d` are the loss against simply holding the two tokens, so they are **zero or negative**, and
  `netApr = feeApr + ilApr`.
- `fee` is in hundredths of a basis point: `500` is the 0.05% tier.
- `rangeBp` is the half-width of a range in basis points of price: `500` is ±5%, `0` is full range. The exact
  position to open is `tickLower..tickUpper`, already aligned to the pool's tick spacing.
- Ids are strings (they can exceed 2^53). Timestamps are ISO 8601 in UTC.

## Plans

Public endpoints work without a key; opportunity signals then appear once they are 24 hours old. The stream,
webhooks and live opportunities need Basic or Pro; smart-LP signals and wallet positions need Pro. See
[lpsignal.app](https://lpsignal.app) for plans and the full API reference.

## Custom rules

On Basic (3 rules) and Pro (20 rules) you can set your own thresholds; a match reaches only you (stream, webhook,
Telegram) and carries `signal.rule = { id, name }`. Thresholds are fractions, like everything else.

```ts
await lps.createRule({ kind: 'net_apr', name: 'Base, 15%+', minNet7d: 0.15, chains: ['base'] });
await lps.createRule({ kind: 'depeg', name: 'early depeg', minDeviation: 0.003 });
await lps.setDefaultSignals(false); // only rule matches and risk alerts from now on
```

```python
lps.create_rule({"kind": "tvl_outflow", "name": "big exits", "minDrop": 0.2, "windowHours": 6})
```

## Development

```bash
cd node && npm install && npm test                 # unit tests
cd python && pip install -e '.[test]' && pytest    # unit tests
```

Both suites check webhook signatures against [`testdata/webhook-vectors.json`](testdata/webhook-vectors.json),
which was generated with the LPSignal server's own signing function.

End-to-end against a running API (read-only unless `E2E_WRITE=1`, which is for test accounts only):

```bash
cd node && npm run build && LPSIGNAL_BASE_URL=https://api.lpsignal.app LPSIGNAL_API_KEY=lps_... npm run e2e
cd python && LPSIGNAL_BASE_URL=https://api.lpsignal.app LPSIGNAL_API_KEY=lps_... python scripts/e2e.py
```

## Releasing

Both packages publish from GitHub Actions with trusted publishing (OIDC): there are no npm or PyPI tokens. Bump the
version in `node/package.json` or `python/pyproject.toml`, commit, then push a matching tag:

```bash
git tag node-v0.1.1 && git push origin node-v0.1.1        # npm, with provenance
git tag python-v0.1.0 && git push origin python-v0.1.0    # PyPI
```

The `release` environment only accepts those tags, and the job refuses a tag whose version differs from the package.

## License

[MIT](LICENSE)
