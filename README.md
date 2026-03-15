# Cloak402 -- Privacy-First AI Travel Agent

Cloak402 is an autonomous travel booking agent that searches and books flights while preserving user privacy through ephemeral wallets, x402 micropayments, and an AI agent (OpenClaw) that enforces strict disclosure policies.

Link to our whitepaper - [Cloak402](https://drive.google.com/file/d/151_EmVE_ZcyXDl5ipDVTfMEvUIUPwQo-/view?usp=drive_link)

**Built for ETHMumbai on Base Sepolia.**

## How It Works

```
User (Web / Chat)
       |
  OpenClaw Agent (LLM-driven)
       |
  cloak402 skill (SKILL.md)
       |
  CLI Tools (search / book / policy)
       |
  ghostPay Privacy Layer
       |
  Ephemeral Wallet (created -> funded -> used -> destroyed)
       |
  x402 Payment Protocol
       |
  Airline API (Express + x402 middleware)
```

**Core privacy principle:** Search is anonymous (no passenger identity). Booking discloses only name and email. Every API call uses a fresh ephemeral wallet that is destroyed after use.

### Privacy Features

- **Ephemeral wallets** -- a new keypair per API call, destroyed after use
- **Randomized USDC funding** -- 0.03-0.09 USDC band defeats amount clustering
- **Randomized timing delays** -- 1.5-6s jitter reduces timestamp correlation
- **CloakPool** (Phase 1) -- shared pool breaks onchain funding linkability
- **Disclosure policies** -- search never includes passenger identity; booking uses minimal disclosure

## Architecture

```
src/
  airline-api/server.ts        # Mock airline API with x402 payment wall
  app/
    api/agent/run/route.ts     # SSE endpoint for web dashboard
    api/chat/route.ts          # Conversational chat SSE endpoint
    page.tsx                   # Web dashboard UI
    types.ts                   # Shared TypeScript types
  lib/
    agent/
      openclawBridge.ts        # Bridges OpenClaw CLI to SSE events
      runAgent.ts              # Static agent fallback
      sessionStore.ts          # Per-user session state
    privacy/
      ghostPay.ts              # Privacy wrapper: delay -> wallet -> x402 -> destroy
    wallets/
      ephemeral.ts             # Ephemeral wallet factory (direct + pool funding)
    config/
      privacyConfig.ts         # Randomization parameters
    pool/
      cloakPool.ts             # CloakPool smart contract integration

skills/cloak402/
  SKILL.md                     # OpenClaw skill definition
  tools/
    search_flights.ts          # Anonymous flight search CLI
    book_flight.ts             # Booking CLI (minimal identity disclosure)
    get_trip_policy.ts         # Read trip constraints

scripts/
  testOpenClaw.ts              # End-to-end integration tests
  checkBalances.ts             # Check control wallet balances
```

## Prerequisites

- Node.js 20+
- npm
- [OpenClaw CLI](https://docs.openclaw.ai) installed and configured
- A Gemini API key (or OpenRouter key) for the LLM agent
- Base Sepolia testnet ETH and USDC in the control wallet

## Setup

### 1. Clone and install

```bash
git clone <repo-url>
cd EthMumbai
npm install
```

### 2. Configure environment

Create `.env.local`:

```env
BASE_SEPOLIA_RPC=https://sepolia.base.org
FACILITATOR_URL=https://x402.org/facilitator
NETWORK=eip155:84532
USDC_ADDRESS=0x036CbD53842c5426634e7929541eC2318f3dCF7e
PAY_TO_ADDRESS=<your-seller-wallet>
AGENT_CONTROL_WALLET_KEY=0x<your-control-wallet-private-key>
EVM_PRIVATE_KEY=0x<your-evm-private-key>
NEXT_PUBLIC_AIRLINE_API_URL=http://localhost:4021
```

### 3. Configure OpenClaw

```bash
# Install OpenClaw if not already installed
npm i -g openclaw

# Run the setup wizard
openclaw onboard

# Set the model (Gemini via Google)
openclaw config set agents.defaults.model "google/gemini-2.5-flash"

# Set timeouts for blockchain operations
openclaw config set tools.exec.backgroundMs 120000
openclaw config set tools.exec.timeoutSec 300
openclaw config set agents.defaults.timeoutSeconds 300
```

Add your Gemini API key to `~/.openclaw/agents/main/agent/auth-profiles.json`:

```json
{
  "version": 1,
  "profiles": {
    "google:default": {
      "type": "api_key",
      "provider": "google",
      "key": "<your-gemini-api-key>"
    }
  }
}
```

Copy the skill to the OpenClaw workspace:

```bash
cp -r skills/cloak402 ~/.openclaw/workspace/skills/cloak402
```

### 4. Fund the control wallet

The control wallet needs Base Sepolia ETH (for gas) and USDC (for x402 payments). Use the [Base Sepolia faucet](https://www.alchemy.com/faucets/base-sepolia) for ETH and the [Circle USDC faucet](https://faucet.circle.com/) for testnet USDC.

Check balances:

```bash
npx tsx scripts/checkBalances.ts
```

## Running

### Start the airline API

```bash
cd src/airline-api && npm install && npx tsx server.ts
```

This starts the mock airline API on port 4021 with x402 payment middleware.

### Start the web dashboard

```bash
npm run dev
```

Open http://localhost:3000 -- configure a trip and watch the agent search and book with full privacy trace.

### Use the OpenClaw agent directly

```bash
# Search (anonymous)
openclaw agent --local --agent main --timeout 300 \
  --message "Find flights from BOM to BLR under 4500 rupees"

# Book
openclaw agent --local --agent main --timeout 300 \
  --message "Book flight F003 for John Doe, email john@example.com"
```

### Use the chat API

```bash
curl -N http://localhost:3000/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Find cheapest flight from Mumbai to Bangalore"}'
```

## Testing

```bash
# Run all tests (tools + agent + API)
npx tsx scripts/testOpenClaw.ts

# Test only direct tool scripts
npx tsx scripts/testOpenClaw.ts --tools

# Test only the OpenClaw agent
npx tsx scripts/testOpenClaw.ts --agent

# Test only the chat API (requires next dev running)
npx tsx scripts/testOpenClaw.ts --api
```

## How the Privacy Layer Works

Each API call follows this flow inside `ghostPay()`:

1. **Random delay** (1.5-6s) -- decorrelates request timing
2. **Create ephemeral wallet** -- fresh keypair generated in memory
3. **Fund wallet** -- control wallet sends randomized USDC (0.03-0.09) + gas ETH
4. **x402 payment** -- ephemeral wallet pays the API via x402 protocol
5. **Receive data** -- flight results or booking confirmation
6. **Destroy wallet** -- private key overwritten in memory

The seller sees a different wallet address on every call. No two calls are linkable on-chain (with CloakPool, even the funding source is unlinkable).

## Static vs LLM Agent

| | Static Agent (`runAgent.ts`) | OpenClaw Agent |
|---|---|---|
| Decision-making | Hardcoded 3-iteration loop | LLM-driven, policy-aware |
| Privacy enforcement | Code-level separation | Skill-level rules + code |
| Chat support | No | Yes (any message) |
| Multi-channel | Web only | Web + Telegram + WhatsApp |

Toggle with `USE_STATIC_AGENT=true` in your environment to fall back to the static agent.

## Threat Model

See [THREAT_MODEL.md](./THREAT_MODEL.md) for a detailed analysis of what Cloak402 protects, its current limitations, and the upgrade roadmap (CloakPool, disclosure policies, ZK authorization).

## Tech Stack

- **Next.js 16** -- web dashboard + API routes
- **React 19** -- frontend UI
- **Express** -- airline API server
- **viem** -- Ethereum client library
- **x402** -- HTTP 402 payment protocol (`@x402/fetch`, `@x402/evm`)
- **OpenClaw** -- AI agent gateway
- **Base Sepolia** -- L2 testnet (EVM)
- **USDC** -- stablecoin for micropayments
- **Tailwind CSS** -- styling

## License

MIT
