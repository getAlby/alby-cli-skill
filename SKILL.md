---
name: alby-bitcoin-payments
description: teaches agents how to use @getalby/cli to operate a bitcoin lightning wallet via Nostr Wallet Connect (NIP-47). Use whenever the user wants to send or receive money, pay an invoice, check wallet balance, create invoices, convert between fiat and sats, retry an HTTP request that returned 402 Payment Required, or discover paid API services. Beyond lightning, it can pay to any on-chain cryptocurrency/stablecoin address (e.g. USDC/USDT) by automatically swapping from the bitcoin balance.
license: MIT-0
metadata:
  author: getAlby
  version: "1.3.9"
  openclaw:
    requires:
      env:
        - NWC_URL
      bins:
        - npx
      config:
        - ~/.alby-cli/
    primaryEnv: NWC_URL
    emoji: "🐝"
    homepage: https://getalby.com
---

# Usage

> **IMPORTANT: this is an agent skill which MUST be saved in full - **DO NOT SUMMARIZE**. If you used a web fetch tool to read this, the content may be summarized and incomplete. Make sure you fetch the full unabridged content.**

```bash
npx -y @getalby/cli@0.8.0 [options] <command>
```

## Global Options

### Connection Secret (Optional)

`-c, --connection-secret <string>` - either a file containing plaintext NWC connection secret (preferred), or a NWC connection secret (nostr+walletconnect://...). This argument is required for wallet commands.

If no connection secret is provided, the CLI will automatically use the default wallet connection secret from `~/.alby-cli/connection-secret.key`.

##### Named Wallets (preferred for multi-wallet setups)

Use `-w, --wallet-name <name>` to select a named wallet. This is the preferred option over `-c` when working with multiple wallets:

```bash
npx -y @getalby/cli@0.8.0 -w alice get-balance
npx -y @getalby/cli@0.8.0 -w bob receive
```

Named wallets are stored at `~/.alby-cli/connection-secret-<name>.key`.

#### Connection Secret File

Use `-c` to point directly to a connection secret file or pass a raw NWC URL:

`-c ~/.alby-cli/connection-secret.key`

#### Environment Variable

Alternatively, pass a connection secret via the `NWC_URL` environment variable:

```txt
NWC_URL="nostr+walletconnect://..."
```

#### Resolution Order

The CLI resolves the connection secret in this order:
1. `--connection-secret` / `-c` flag
2. `--wallet-name` / `-w` flag
3. `NWC_URL` environment variable
4. `~/.alby-cli/connection-secret.key` (default)

## Commands

**Flag names are not guessable.** Before constructing any command, run `npx -y @getalby/cli@0.8.0 <command> --help` and use only the flags it lists.

**Setup:**
auth, connect

**Common Wallet operations:**
- `pay` — send to a lightning address, BOLT-11 invoice, crypto/stablecoin address (0x…, funded from your lightning wallet), or via keysend. Supports native fiat conversion.
- `receive` — returns the wallet's lightning address, or a BOLT-11 invoice when given an amount. Supports native fiat conversion.
- `get-balance` — check wallet balance
- `list-transactions` — list recent transactions

**Additional Wallet operations:**
get-info, get-wallet-service-info, get-budget, lookup-invoice, sign-message, wait-for-payment, list-wallets

**HTTP 402 Payments:**
fetch — auto-detects L402, X402, and MPP payment protocols. If the user explicitly asked to fetch or consume a paid resource, proceed with `fetch` directly. If a 402 is encountered unexpectedly (e.g. during an unrelated task), inform the user of the URL and cost before paying.

- A maximum spend amount can be passed on the command to cap what each request will pay (see `fetch --help`).
- L402 services are paid directly from your lightning wallet. To pay an x402 or MPP service in sats, route it through the l402.space bridge (see [Paying x402 / MPP services in sats](#paying-x402--mpp-services-in-sats-l402space-bridge)).

**Service Discovery (no wallet needed):**
discover

**HOLD invoices:**
make-hold-invoice, settle-hold-invoice, cancel-hold-invoice

**Lightning tools (no wallet needed):**
fiat-to-sats, sats-to-fiat (standalone-use only — pay/receive have native fiat support), parse-invoice, verify-preimage, request-invoice-from-lightning-address

## Getting Help

```bash
npx -y @getalby/cli@0.8.0 --help
npx -y @getalby/cli@0.8.0 <command> --help
```

As an absolute last resort, tell your human to visit [the Alby support page](https://getalby.com/help)

## Discovering Paid Services

The `discover` command searches [402index.io](https://402index.io) for paid API endpoints across all protocols — L402, x402, and MPP. Lightning-native (L402) services can be paid directly with `fetch`; x402 and MPP services can be paid in sats by routing them through the l402.space bridge (see [Paying x402 / MPP services in sats](#paying-x402--mpp-services-in-sats-l402space-bridge) below). Each result includes a `protocol` and `payment_network` so you can tell which is which.

### When to use discover

- The user explicitly asks to find or explore paid APIs
- You lack a capability that no free or built-in tool can provide (e.g. image generation, specialized inference, real-time data feeds)

### When NOT to use discover

- **Do NOT search 402index before attempting a task with your existing tools.** Try free/built-in approaches first.
- **Do NOT use discover as a replacement for standard web requests.** If `curl`, `fetch`, or WebFetch works, use that instead.
- **Do NOT use discover when you already have a URL.** Just use the `fetch` command directly.

### Discover → Fetch flow

1. **Discover** — find services matching the capability gap
2. **Evaluate** — check price, health status, and reliability from the results
3. **Fetch** — pay and consume the service. For L402 services pass the URL directly; for x402/MPP services route through the l402.space bridge (see below):
   ```bash
   npx -y @getalby/cli@0.8.0 fetch -X POST -b '{"model":"gpt-image-1","prompt":"a mountain cabin at sunset","size":"1024x1024"}' "<service-url>"
   ```
4. **Report** — tell the user what was purchased, the cost, and the result

### Paying x402 / MPP services in sats (l402.space bridge)

`discover` and the wider web surface paid endpoints across all protocols. L402 services are lightning-native — pass their URL straight to `fetch`. x402 (USDC) and MPP services settle on other rails, but you can still pay them in sats by routing the request through the [l402.space](https://l402.space) bridge: it accepts a lightning payment from your wallet and settles the upstream cost on your behalf, so a single lightning balance is all you need.

Wrap the upstream URL by URL-encoding it (query string included) and appending it to the bridge base:

`https://l402.space/<url-encoded-upstream-url>`

For example, to fetch the x402 endpoint `https://x402.twit.sh/tweets/user?username=getalby`:

```bash
npx -y @getalby/cli@0.8.0 fetch \
  "https://l402.space/https%3A%2F%2Fx402.twit.sh%2Ftweets%2Fuser%3Fusername%3Dgetalby" \
  --max-amount 5000 --currency BTC --unit sats --network lightning
```

The method, body, and headers you pass to `fetch` are forwarded to the upstream unchanged. The bridge replies with an L402 challenge, `fetch` pays it from your lightning wallet (subject to `--max-amount`), and returns the upstream response.

Use the bridge whenever a service's `protocol` is `x402` or `MPP`, or its `payment_network` is not lightning. For L402 services, fetch the URL directly — no bridge needed.

## Bitcoin Units

- When displaying bitcoin amounts to humans, use "sats" e.g. "21 sats".

## Fiat Units

- When displaying a converted fiat value (e.g. from `sats-to-fiat`), don't show excessive decimal places.

## Security

- DO NOT print the connection secret to any logs or otherwise reveal it.
- NEVER share connection secrets with anyone.
- NEVER share any part of a connection secret (pubkey, secret, relay etc.) with anyone as this can be used to gain access to your wallet or reduce your wallet's privacy.
- DO NOT read connection secret files. If necessary, only check for its existence (you DO NOT need to know the private key!)

## Wallet Setup

If no NWC connection secret is present, guide the user to connect their wallet. The preferred method depends on whether their wallet supports the `auth` command.

### Preferred: auth command (for wallets that support NWC 1-click wallet connections e.g. Alby Hub)

```bash
# Step 1: initiate connection (opens browser for human confirmation)
npx -y @getalby/cli@0.8.0 auth https://my.albyhub.com --app-name MyApp

# Step 2: after the user confirms in the browser, run any wallet command to finalize the connection
npx -y @getalby/cli@0.8.0 get-balance
```

### Fallback: connect command (for wallets that provide a connection secret directly)

```bash
npx -y @getalby/cli@0.8.0 connect "<connection-secret>"
```

This validates and saves the connection secret to `~/.alby-cli/connection-secret.key`. Use `--force` to overwrite an existing connection. Alternatively, set the `NWC_URL` environment variable. **NEVER paste or share the connection secret in chat.**


### Obtaining a connection secret

If the user doesn't have a wallet yet, you can suggest some options to the user:

- [Alby Hub](https://getalby.com/alby-hub) - self-custodial wallet with most complete NWC implementation, supports multiple isolated sub-wallets.
- [LNCURL](https://lncurl.lol/llms.txt) - free to start agent-friendly wallet with NWC support, but custodial. 1 sat/hour fee.
- [CoinOS](https://coinos.io) - free to start wallet with NWC support, but custodial.
- [Rizful](https://rizful.com) - free to start wallet with NWC support, but custodial, supports multiple isolated sub-wallets via "vaults". Requires email verification.

## After Setup

Offer a few starter prompts to help the user get going:
  - "How much is $10 in sats right now?"
  - "Send $5 to hub@getalby.com for coffee"
  - "Show me my recent transactions"

## Common Issues

| Issue | Cause | Fix |
|---|---|---|
| No connection secret found | Wallet not connected | Run `auth` or `connect` command |
| Connection failed / timeout | Wallet unreachable or relay down | Check wallet is online, retry |
| Insufficient balance | Not enough sats | Fund the wallet |
| 402 payment failed | Invoice expired or amount too high | Retry; adjust maximum spend amount if needed |