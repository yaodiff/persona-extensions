# Binance

Read your Binance account in Persona and, after a confirmation on your phone, place or cancel single spot, USDⓈ-M and COIN-M futures orders within guards you set.

## What it does

Read tools (also available to coding agents): `binance_overview`, `binance_balances`, `binance_positions`, `binance_open_orders`, `binance_order`, `binance_trades`, `binance_income`, `binance_transfers` (deposit and withdrawal records only), `binance_price`, `binance_limits`, `binance_status`.

Writes (assistant only, every one confirmed on the phone through `confirm_always`): `binance_spot_order`, `binance_spot_cancel`, `binance_usdm_order`, `binance_usdm_cancel`, `binance_coinm_order`, `binance_coinm_cancel`. Orders are `LIMIT` or `MARKET`; futures orders support `reduce_only` (one-way mode) and `position_side` (hedge mode).

Not offered, and not reachable: withdrawals, any transfer, loans, leverage or margin-mode changes, conditional orders (stop, OCO, algo), batch orders or cancel-all. The client only calls the endpoints in [endpoints.go](endpoints.go); `endpoints_test.go` fails when that table, or any Binance path written in the source, could move money out, borrow, or change leverage or margin.

Inbox items (with a fallback tier, and pushed when urgent): `liq_risk`, `position_change` (a position changed without an order from Persona), `order_filled`, `order_rejected`, `price_move`, `deposit`, `withdrawal`, `key_problem`.

## Guards

Every order is checked when the card is built (cached prices and exchange rules, no network) and again right before it is sent (a price read then):

| Setting | Default |
| --- | --- |
| `trading_enabled` | off |
| `markets` | `spot` |
| `symbol_whitelist` | empty (nothing may be traded) |
| `max_order_usdt` | 200 |
| `max_daily_usdt` | 1000 (reset at 00:00 in `timezone`; reduce-only orders and cancels do not count) |
| `max_orders_per_day` | 20 |
| `max_leverage` | 5 (the extension never changes leverage; opening above it is refused) |
| `limit_deviation_pct` | 5 |
| `market_orders` | on |
| `market_slippage_pct` | 1 (a market order is refused when the price moved more than this since the card) |

Each card yields at most one order: the order's `newClientOrderId` is derived from the hub's request id, and a repeated request returns the first result. A rejected order gives its allowance back. A failed send whose outcome is open (lost connection, timeout, any 5xx) is never sent again: the order is looked up by its client id; only Binance answering "no such order" twice gives the allowance back, otherwise the order is marked unknown, its allowance stays used and you get a push to check it. Open orders carry their amount into the next day's allowance, and a market order counts its worst case (price now plus the slippage guard).

## Set up

1. Create an Ed25519 key pair on the hub machine: `openssl genpkey -algorithm ed25519 -out binance.pem` and `openssl pkey -in binance.pem -pubout -out binance.pub.pem`.
2. In Binance API Management create a self-generated API key with the public key. Enable **Reading** and the trading you want (**Spot & Margin Trading** and/or **Futures**). Leave withdrawals, internal transfer, universal transfer, margin loans, options and portfolio margin off. Restrict access to trusted IPs: the public IP Binance sees from the hub (or from the proxy below).
3. In **Settings › Extensions › 币安**, enter the API key and the private key (PEM). Saving checks the key with Binance first and refuses one that can withdraw or transfer, has no IP whitelist, or cannot read or trade.
4. Optionally set `proxy_url` to an HTTP proxy on the hub machine that supports CONNECT, as `http://host.persona.internal:<port>`. In the sandbox the extension reaches it through the hub's egress proxy, then tunnels to Binance through it.
5. Turn on `trading_enabled` and add the symbols you allow.

Keys and the private key go only through the hub's encrypted secret flow; they are never logged or sent anywhere but Binance (the key as the `X-MBX-APIKEY` header, signatures in the query).

## Verify

```sh
go -C extensions/binance test -race ./...
scripts/package.sh binance
```

The tests use a synthetic Binance server and a key made for each run; they need no account.
