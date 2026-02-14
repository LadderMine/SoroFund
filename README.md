Sorofund is a milestone-based crowdfunding for real-world projects in emerging markets.

## How It Works🔥

```
┌──────────┐     ┌───────────────┐     ┌──────────────┐     ┌─────────────┐
│  Creator  │────▶│ Create Campaign│────▶│ Backers Fund │────▶│  Milestone  │
│  submits  │     │  + Milestones  │     │  (USDC locked │     │  Verified?  │
│  project  │     │  on Soroban    │     │   in escrow)  │     │             │
└──────────┘     └───────────────┘     └──────────────┘     └──────┬──────┘
                                                                    │
                                                          ┌────────┴────────┐
                                                          │                 │
                                                     YES ▼            NO ▼
                                                ┌──────────────┐  ┌──────────────┐
                                                │ Funds released│  │ Funds refunded│
                                                │ to creator    │  │ to backers    │
                                                └──────────────┘  └──────────────┘
```

**The lifecycle of a SoroFund campaign:**

1. **Creator** submits a project with a funding goal, deadline, and a series of milestones (each with a description, target amount, and deliverable).
2. The backend validates the submission and deploys a campaign-specific escrow on the Soroban smart contract, locking the milestone structure on-chain.
3. **Backers** discover the project, review milestones, and pledge funds (USDC on Stellar). Pledged funds go directly into the on-chain escrow — SoroFund never holds your money.
4. If the funding goal is met by the deadline, the campaign is **Active**. If not, all funds are automatically refundable.
5. The creator works on Milestone 1, then submits proof of completion (photos, documents, links) through the platform.
6. **Reviewers** (a panel of 3 assigned at campaign creation — can be community members, domain experts, or platform moderators) evaluate the proof and vote on-chain.
7. If 2-of-3 reviewers approve, the smart contract releases that milestone's funds to the creator. If rejected, the creator can resubmit or the milestone fails.
8. This repeats for each milestone. Unused funds from failed milestones are refundable to backers.

---

## Tech Stack at a Glance

**On-chain (Rust/Soroban):** Campaign escrow contract, milestone verification contract, reviewer registry contract

**Backend (NestJS):** Campaign management, user auth, reviewer assignment, milestone proof uploads, notification engine, admin tools — all backed by PostgreSQL via TypeORM

**Frontend (Next.js):** Campaign discovery, creation wizard, backer pledge flow, reviewer dashboard, creator milestone tracking — styled with Tailwind CSS

**Infra:** Redis (caching + queues), Cloudinary (proof-of-milestone media), Socket.IO (real-time campaign updates), BullMQ (scheduled jobs)

---

## Repo Structure

```
sorofund/
│
├── contracts/
│   ├── campaign/           Soroban escrow — holds funds, enforces milestones, handles releases & refunds
│   ├── reviewer/           On-chain reviewer registry — tracks who can vote on which campaigns  
│   └── governance/         Dispute resolution — handles appeals when milestones are rejected
│
├── apps/
│   ├── api/                NestJS backend
│   │   ├── src/
│   │   │   ├── auth/           JWT auth, OAuth, phone OTP
│   │   │   ├── campaigns/      Campaign CRUD, state machine, Soroban integration
│   │   │   ├── milestones/     Proof submission, reviewer notification, vote tallying
│   │   │   ├── backers/        Pledge management, refund processing
│   │   │   ├── reviewers/      Reviewer onboarding, assignment algorithm, reputation tracking
│   │   │   ├── users/          Profiles, KYC status, wallet linking
│   │   │   ├── notifications/  Email, SMS, push, WebSocket
│   │   │   ├── admin/          Platform moderation, campaign flagging, analytics
│   │   │   └── common/         Guards, interceptors, pipes, shared DTOs
│   │   └── ...
│   │
│   └── web/                Next.js frontend
│       ├── app/
│       │   ├── (public)/       Landing, explore campaigns, campaign detail
│       │   ├── (auth)/         Login, register, verify
│       │   ├── dashboard/      Creator dashboard — my campaigns, milestones, earnings
│       │   ├── fund/           Backer flow — pledge, track, claim refunds
│       │   ├── review/         Reviewer panel — pending reviews, vote history
│       │   └── admin/          Platform admin — moderation, analytics
│       └── ...
│
├── packages/
│   ├── shared/             Types, enums, constants shared between api and web
│   └── stellar-helpers/    Transaction builders, Soroban invocation wrappers
│
├── docker-compose.yml
├── turbo.json
└── README.md
```

---

## Deep Dive: The Smart Contracts

### Campaign Contract

This is the heart of SoroFund. One contract instance manages all campaigns (no per-campaign deployment — that would be wasteful and expensive). Each campaign is stored as structured data keyed by a unique campaign ID.

**What it stores on-chain:**
- Campaign metadata hash (actual metadata lives off-chain in PostgreSQL; the hash ensures integrity)
- Creator address
- Funding goal and deadline (ledger timestamp)
- Milestone count, amounts, and status for each
- Total funds pledged (per-backer tracking)
- Campaign state enum: `Funding` → `Active` → `Completed` | `Failed` | `Cancelled`

**What it enforces:**
- Backers can only pledge during the `Funding` phase, before the deadline
- If the goal isn't met by the deadline, the campaign transitions to `Failed` and all backers can call `refund()` to reclaim their USDC
- Milestone funds are only released when the Reviewer contract confirms a passing vote (2-of-3)
- The creator cannot withdraw funds outside the milestone release flow — there's no backdoor
- If all milestones are completed, the campaign transitions to `Completed`

```rust
#![no_std]
use soroban_sdk::{contract, contractimpl, Address, Env, Symbol, Vec, Map};

#[contract]
pub struct CampaignContract;

#[contractimpl]
impl CampaignContract {
    /// Initialize a new campaign with milestones
    pub fn create_campaign(
        env: Env,
        creator: Address,
        token: Address,                   // USDC contract address
        goal: i128,                        // Total funding goal
        deadline: u64,                     // Funding deadline (ledger timestamp)
        milestone_amounts: Vec<i128>,      // How much to release per milestone
        metadata_hash: Symbol,             // SHA-256 of off-chain metadata
    ) -> Symbol;                           // Returns campaign_id

    /// Backer pledges funds to a campaign
    pub fn pledge(env: Env, backer: Address, campaign_id: Symbol, amount: i128);

    /// Release milestone funds to creator (called by reviewer contract)
    pub fn release_milestone(
        env: Env,
        campaign_id: Symbol,
        milestone_index: u32,
        reviewer_contract: Address,        // Must be the authorized reviewer contract
    );

    /// Backer reclaims funds from a failed/cancelled campaign
    pub fn refund(env: Env, backer: Address, campaign_id: Symbol) -> i128;

    /// Cancel a campaign (creator only, only during Funding phase)
    pub fn cancel(env: Env, creator: Address, campaign_id: Symbol);

    /// Read campaign state
    pub fn get_campaign(env: Env, campaign_id: Symbol) -> CampaignData;

    /// Read a backer's pledge amount
    pub fn get_pledge(env: Env, campaign_id: Symbol, backer: Address) -> i128;
}
```

### Reviewer Contract

Manages the reviewer panel for each campaign and tallies on-chain votes for milestone approvals.

When a campaign is created, the backend assigns 3 reviewers (selected from a pool of verified, reputable reviewers). Their addresses are registered on-chain for that campaign. When a milestone proof is submitted, each reviewer votes `approve` or `reject`. The contract counts votes and, once a 2-of-3 threshold is reached, calls `release_milestone` on the Campaign Contract (for approval) or marks the milestone as `rejected` (triggering a resubmission window or refund).

Reviewer reputation scores are tracked on-chain — reviewers who consistently participate and vote honestly build reputation. Reviewers who go inactive or vote maliciously lose reputation, and the backend rotates them out of the active pool.

### Governance Contract

Handles edge cases and disputes. If a creator believes a milestone was unfairly rejected, they can file an appeal. The governance contract opens a time-limited appeal window where a larger panel (5 reviewers, drawn from high-reputation members) re-evaluates the milestone. The outcome of the appeal is final. This prevents gaming by a single bad reviewer while keeping the system lightweight for the 95% of milestones that don't need dispute resolution.

---

## Deep Dive: The Backend

The NestJS API isn't just a CRUD layer — it's the brain that orchestrates campaign lifecycles, reviewer assignments, and on-chain interactions.

### Campaign State Machine

Every campaign follows a strict state machine. The backend enforces state transitions and triggers side effects (notifications, contract calls, scheduled jobs):

```
                    ┌─────────────────────────────┐
                    │                             │
                    ▼                             │
┌───────┐    ┌──────────┐    ┌────────┐    ┌──────────┐
│ Draft │───▶│ Funding  │───▶│ Active │───▶│ Completed│
└───────┘    └────┬─────┘    └───┬────┘    └──────────┘
                  │              │
                  ▼              ▼
             ┌────────┐    ┌────────┐
             │ Failed │    │ Paused │
             └────────┘    └────────┘
                  ▲
                  │
             ┌──────────┐
             │ Cancelled│
             └──────────┘
```

- **Draft** → Creator is still editing. Not visible to backers.
- **Funding** → Live and accepting pledges. The contract holds funds in escrow.
- **Active** → Goal met. Creator is working on milestones.
- **Completed** → All milestones delivered and funds released.
- **Failed** → Deadline passed without meeting the goal. Refunds available.
- **Cancelled** → Creator cancelled during Funding phase. Auto-refund.
- **Paused** → Admin flagged for review (suspicious activity).

### Reviewer Assignment Algorithm

When a campaign enters `Funding`, the backend auto-assigns 3 reviewers from the active pool. The algorithm optimizes for:

1. **Relevance** — Reviewers who have domain tags matching the campaign category (e.g., agriculture, tech, infrastructure) are preferred.
2. **Availability** — Reviewers currently assigned to fewer than 5 active campaigns get priority.
3. **Reputation** — Higher reputation scores break ties.
4. **Geography** — At least one reviewer should be from the same region as the campaign (local context matters for verifying real-world deliverables).

Reviewers can decline an assignment within 48 hours, triggering a replacement.

### Key API Flows

**Campaign Creation:**
`POST /campaigns` → validate input → save as Draft → creator adds milestones via `POST /campaigns/:id/milestones` → creator publishes via `POST /campaigns/:id/publish` → backend deploys campaign to Soroban → assigns reviewers → campaign enters Funding

**Pledging:**
`POST /campaigns/:id/pledge` → validate backer wallet has sufficient USDC balance → invoke `pledge()` on Campaign Contract → record pledge in database → send real-time update to campaign page via WebSocket → send thank-you notification to backer

**Milestone Submission:**
`POST /milestones/:id/submit` → creator uploads proof (images, docs, links to Cloudinary) → backend notifies assigned reviewers via email + in-app → reviewers vote via `POST /milestones/:id/vote` → backend submits vote to Reviewer Contract → on 2-of-3 approval, backend invokes `release_milestone()` on Campaign Contract → funds transfer to creator → all backers notified

---

## Deep Dive: The Frontend

### The Explore Experience

The landing page isn't a generic hero section. It opens with a live feed of recently funded milestones — real evidence that the platform works. Below that, a filterable grid of active campaigns with progress bars, days remaining, and backer counts. Category filters (Agriculture, Education, Technology, Infrastructure, Health, Creative) and location filters (by country/region) help backers find what matters to them.

### The Campaign Page

Each campaign page is designed to build trust:

- **Hero section** with campaign title, creator profile (with verification badge if KYC'd), location, and a prominent progress bar showing funds raised vs. goal.
- **Milestone timeline** — a visual, step-by-step breakdown of what the creator will deliver and when. Each milestone shows its status (upcoming, in-progress, submitted, approved, rejected) and the amount that will be released upon completion.
- **Proof gallery** — for completed milestones, backers can see the actual proof submitted (photos of equipment purchased, screenshots of work done, receipts, etc.).
- **Backer list** — transparent view of who has backed the project (wallet addresses or display names) and how much.
- **Updates** — creator can post text/photo updates to keep backers informed between milestones.
- **Discussion** — threaded comments where backers and the creator can communicate.

### The Creator Dashboard

A focused workspace for managing campaigns. Campaign cards show real-time funding progress, upcoming milestone deadlines, and action items (submit proof, respond to reviewer feedback). The milestone submission flow is a guided wizard: upload proof → add description → preview → submit. The creator can also track their total earnings, pending releases, and historical campaigns.

### The Reviewer Panel

A clean interface for assigned reviewers. Shows pending reviews with campaign context, milestone details, and submitted proof. Reviewers can view proof media inline, read backer comments, and cast their vote with an optional note explaining their reasoning. The panel also shows the reviewer's reputation score and review history.

---

## Getting Started

> **You need:** Node.js 18+, pnpm 8+, Rust 1.74+ (with wasm32-unknown-unknown), Stellar CLI, PostgreSQL 15+, Redis 7+, Docker (optional)

```bash
# Clone
git clone https://github.com/your-org/sorofund.git && cd sorofund

# Install
pnpm install

# Configure
cp apps/api/.env.example apps/api/.env
cp apps/web/.env.example apps/web/.env
# Fill in your values (database URL, Stellar keys, Cloudinary, etc.)

# Database
docker-compose up -d postgres redis       # or use your local instances
pnpm --filter api migration:run
pnpm --filter api seed

# Smart contracts
cd contracts
rustup target add wasm32-unknown-unknown
cargo build --target wasm32-unknown-unknown --release

stellar keys generate --global deployer --network testnet
stellar keys fund deployer --network testnet

stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/campaign.wasm \
  --source deployer --network testnet --alias campaign

stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/reviewer.wasm \
  --source deployer --network testnet --alias reviewer

stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/governance.wasm \
  --source deployer --network testnet --alias governance

# Copy contract IDs to apps/api/.env

# Run
pnpm dev
# API: http://localhost:4000  |  Web: http://localhost:3000  |  Docs: http://localhost:4000/api/docs
```

<details>
<summary><strong>Full .env reference for the backend</strong></summary>

```env
DATABASE_URL=postgresql://user:password@localhost:5432/sorofund
REDIS_URL=redis://localhost:6379

JWT_SECRET=your-secret
JWT_REFRESH_SECRET=your-refresh-secret

STELLAR_NETWORK=testnet
STELLAR_RPC_URL=https://soroban-testnet.stellar.org
STELLAR_HORIZON_URL=https://horizon-testnet.stellar.org
STELLAR_NETWORK_PASSPHRASE=Test SDF Network ; September 2015

CAMPAIGN_CONTRACT_ID=C...
REVIEWER_CONTRACT_ID=C...
GOVERNANCE_CONTRACT_ID=C...

PLATFORM_STELLAR_PUBLIC_KEY=G...
PLATFORM_STELLAR_SECRET_KEY=S...

CLOUDINARY_URL=cloudinary://...
SENDGRID_API_KEY=SG...
SENDGRID_FROM_EMAIL=hello@sorofund.io
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...
TWILIO_PHONE_NUMBER=+1...
```

</details>

---

## Testing

```bash
# Contracts — unit tests + cross-contract integration tests
cd contracts && cargo test

# Backend — unit + e2e
pnpm --filter api test
pnpm --filter api test:e2e

# Frontend — component + hook tests
pnpm --filter web test

# Everything at once
pnpm test
```

---

## Links

[Stellar Docs](https://developers.stellar.org) · [Soroban](https://stellar.org/soroban) · [Soroban SDK](https://github.com/stellar/rs-soroban-sdk) · [Soroban Example Crowdfund dApp](https://github.com/stellar/soroban-example-dapp) · [Stellar Community Fund](https://communityfund.stellar.org) · [Drips Wave](https://www.drips.network/wave/stellar) · [OpenZeppelin Stellar Contracts](https://github.com/OpenZeppelin/stellar-contracts)
