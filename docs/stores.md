# In-Memory Stores and Test Reset Contract

`src/stores.ts` is the single in-process state module for StableRoute Backend.
Route handlers import typed stores and accessors from this module instead of
creating ad hoc mutable state. The stores are process-local: restarting the Node
process resets every collection and scalar to its default value.

The source comments intentionally describe which pieces mirror on-chain state so
future database or ledger adapters can preserve the same model.

## Store Inventory

| Export | Type | Holds | Mirrors or supports |
| --- | --- | --- | --- |
| `pairRegistry` | `Set<string>` | Registered pair keys formatted as `SOURCE::DEST` | On-chain `DataKey::Pair(source, dest)` membership |
| `pairMeta` | `Map<string, PairMeta>` | Per-pair fee, min amount, max amount, and liquidity metadata keyed by `pairKey(source, dest)` | On-chain pair fee/min/max/liquidity entries |
| `apiKeyStore` | `Map<string, ApiKeyRecord>` | Generated API key labels and creation timestamps | Runtime API-key management |
| `webhookStore` | `Map<string, WebhookRecord>` | Webhook URL, subscribed event names, and creation timestamp by webhook id | Runtime webhook registration |
| `eventLog` | `AppEvent[]` | Bounded event ring buffer with id, timestamp, type, and payload | Operator/API event history |
| `rateBuckets` | `Map<string, number[]>` | Per-IP sliding-window request timestamps | In-memory rate limiting |
| `config` | `Record<string, number>` | Mutable runtime settings such as `rateLimitPerWindow`, `rateLimitWindowMs`, `bulkMaxItems`, and `eventLogCap` | `GET/PATCH /api/v1/config` |
| `paused` | `boolean` | Service-level pause state | Pause-guard middleware |

## Pair Keys

Use `pairKey(source, dest)` whenever code needs to address a pair. It encodes the
pair as:

```ts
pairKey("USDC", "EURC"); // "USDC::EURC"
```

Keeping the delimiter centralized prevents handlers and tests from drifting on
how pair state is stored in `pairRegistry` and `pairMeta`.

## Default Pair Metadata

`defaultMeta()` returns a fresh `PairMeta` object with all numeric fields zeroed:

```ts
{
  feeBps: 0,
  minAmount: "0",
  maxAmount: "0",
  liquidity: "0",
}
```

Use it for newly registered pairs so every pair starts from the same baseline and
later updates can patch individual fields without assuming missing keys.

## Event Log Lifecycle

`recordEvent(type, payload)` appends an `AppEvent` with a generated id and current
timestamp. `EVENT_LOG_CAP` is `10_000`; when the buffer grows beyond that cap,
the oldest event is removed with `shift()`.

The log is intentionally in memory. A restart clears it, so callers must not rely
on it as durable audit storage.

## Runtime Config Lifecycle

`config` starts from `defaultConfig()` and is mutated in place so imports keep a
stable object reference. The default keys are:

- `rateLimitPerWindow`
- `rateLimitWindowMs`
- `bulkMaxItems`
- `eventLogCap`

`resetStores()` deletes current keys and reassigns factory defaults, which is
important for tests that mutate config through API handlers.

## Pause State

`paused` is exported as a read binding and updated through `setPaused(value)`.
Callers should use `setPaused` instead of attempting to reassign `paused` from a
consumer module.

## `resetStores()` Test Contract

`resetStores()` is a test-isolation helper. It clears every mutable collection,
restores config defaults, and sets `paused` back to `false`.

Use it in test hooks whenever a test touches route handlers, stores, rate limits,
webhooks, API keys, config, event logs, or pause state:

```ts
import { resetStores } from "../stores";

beforeEach(() => {
  resetStores();
});
```

Omitting the reset can create cross-test bleed: a pair registered in one test,
a consumed rate-limit bucket, a paused service flag, or a patched config value can
change the behavior of later tests.

`resetStores()` is never exposed over HTTP. It exists only as a module export for
the test harness and local contributors.

## Restart Semantics

All state in `src/stores.ts` is process-local and non-durable. A process restart,
serverless cold start, or test module reload returns the stores to their initial
empty/default state. Durable persistence should be added behind explicit adapters
rather than by reaching around this module.