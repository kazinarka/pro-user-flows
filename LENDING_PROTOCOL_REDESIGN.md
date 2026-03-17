# Lending Protocol — Simplified UX Redesign

> **Goal:** Make the lending protocol accessible to users who have never interacted with DeFi or blockchain.  
> **Principle:** Hide blockchain complexity. Speak in dollars, not tokens. Guide, don't overwhelm.

---

## Table of Contents

1. [Current vs. Proposed — Summary of Changes](#1-current-vs-proposed--summary-of-changes)
2. [Simplified Concepts — How We Explain Things](#2-simplified-concepts--how-we-explain-things)
3. [Full User Flow Diagram](#3-full-user-flow-diagram)
4. [Page-by-Page Design Specs](#4-page-by-page-design-specs)
  - 4.1 [Lending Hub (Entry Point)](#41-lending-hub-entry-point)
  - 4.2 [Earn Page (Deposit/Withdraw)](#42-earn-page-depositwithdraw)
  - 4.3 [Borrow Page (Borrow/Repay)](#43-borrow-page-borrowrepay)
  - 4.4 [Collateral Page (Add/Remove)](#44-collateral-page-addremove)
  - 4.5 [Liquidation Page (Advanced)](#45-liquidation-page-advanced)
  - 4.6 [Transaction History](#46-transaction-history)
5. [Approval Abstraction Strategy](#5-approval-abstraction-strategy)
6. [Error Message Rewriting](#6-error-message-rewriting)
7. [Tooltips & Education Layer](#7-tooltips--education-layer)
8. [Mobile Considerations](#8-mobile-considerations)

---

## 1. Current vs. Proposed — Summary of Changes

### Current Problems
| Problem | Where | Impact |
|---------|-------|--------|
| 7 separate modal windows for 7 actions | `LendingProtocol.vue` | User must understand entire DeFi model upfront |
| "Approve WBTC" / "Approve USDC" as separate buttons | Every modal | Users don't know what "approve" means |
| "Shares to burn", "exchange rate USDC/share" | `WithDrawModal`, `DepositModal` | Meaningless to non-crypto users |
| "Health Factor: 1.47 → 1.23" | `BorrowModal`, `RepayModal`, `RemoveCollateral` | Users don't know what Health Factor is |
| Collateral card requires click-to-flip animation | `LendingProtocol.vue` | Hidden actions behind non-obvious interaction |
| Liquidation exposed as primary feature | Same-level modal as Deposit | Intimidating; only relevant to advanced arbitrageurs |
| Raw BigInt values and 8/18 decimal formatting | Everywhere | `0.00000312 WBTC` is confusing |
| Pool selector (WBTC_USDC / WXRP_USDC / WETH_USDC) | Top dropdown | Technical pool naming |
| Error codes: PAUSED, INSUF_SHARES, INSUF_CASH, INSUF_COLL | Watch handlers | Cryptic to users |

### Proposed Simplifications
| Change | Details |
|--------|---------|
| **Replace 7 modals with 3 focused pages** | Earn, Borrow, Manage Collateral — each a full view, not a modal |
| **Auto-approve in background** | Merge "Approve" + "Action" into a single button with a stepper progress |
| **Remove "shares" from UI** | Show only dollar values and token amounts |
| **Replace "Health Factor" with visual safety meter** | Traffic-light gauge: "Safe" / "Caution" / "Danger" with plain-English explanation |
| **Rename pools to asset names** | "Bitcoin Pool", "XRP Pool", "Ethereum Pool" |
| **All amounts shown in $ first** | Token amounts secondary, in smaller text |
| **Hide Liquidation behind "Advanced" toggle** | Separate page, not visible by default |
| **Guided first-time experience** | Inline explainer cards for first visit |
| **Human-readable errors** | Full sentence errors with suggested actions |

---

## 2. Simplified Concepts — How We Explain Things

### Vocabulary Translation Table

| DeFi Term (Current) | User-Friendly Term (Proposed) | Tooltip / Explanation |
|---------------------|-------------------------------|----------------------|
| Deposit | **Earn** / "Start Earning" | "Lend your USDC to earn interest" |
| Withdraw | **Cash Out** / "Stop Earning" | "Withdraw your USDC plus earnings" |
| Borrow | **Borrow** | "Borrow USDC using your crypto as a guarantee" |
| Repay | **Pay Back** | "Return borrowed USDC to free your guarantee" |
| Collateral | **Guarantee** | "Crypto you lock to secure your loan" |
| Add Collateral | **Add Guarantee** | "Lock more crypto to keep your loan safe" |
| Remove Collateral | **Unlock Guarantee** | "Release some of your locked crypto" |
| Health Factor | **Loan Safety** | "How safe your loan is. Green = safe, Red = risky" |
| Liquidation | **Auto-Repayment** | "If your loan becomes unsafe, the system may sell your guarantee to repay it" |
| APY | **Annual Earnings Rate** | "How much you'll earn in a year, in %" |
| Shares | *(hidden)* | Not shown — only $ amounts displayed |
| Exchange Rate | *(hidden)* | Not shown to user |
| Utilization | *(hidden from main view)* | Only in "Advanced Stats" expandable |
| Supply Cap | "Earning limit reached" | "This pool is temporarily full" |
| ERC20 Approve | *(hidden)* | Handled automatically as step 1 of 2 |

---

## 3. Full User Flow Diagram

### Entry Point
```
My Wallet → Lending Tab (Polygon only)
    │
    └── Lending Hub
         │
         ├── [1] YOUR OVERVIEW CARD
         │    │ Total Earning: $1,240.00
         │    │ Total Borrowed: $500.00
         │    │ Guarantee Locked: $2,100.00 (0.021 BTC)
         │    │ Loan Safety: ████████░░ Safe
         │    └── "View History" link
         │
         ├── [2] POOL SELECTOR
         │    │ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
         │    │ │ ₿ Bitcoin    │ │ ◆ Ethereum  │ │ ✕ XRP       │
         │    │ │ Pool         │ │ Pool        │ │ Pool        │
         │    │ │ Earn: 4.2%   │ │ Earn: 3.8%  │ │ Earn: 5.1%  │
         │    │ │ Borrow: 6.1% │ │ Borrow: 5.5%│ │ Borrow: 7.0%│
         │    │ └──────────────┘ └─────────────┘ └─────────────┘
         │    └── Selected pool highlighted with gradient border
         │
         └── [3] ACTION CARDS (3 cards, horizontal)
              │
              ├── 💰 EARN Card
              │    │ "Lend USDC, earn interest"
              │    │ Your earnings: $1,240.00
              │    │ Annual rate: 4.2%
              │    │ [Start Earning] [Cash Out]
              │    └── → Opens Earn Page
              │
              ├── 🏦 BORROW Card
              │    │ "Borrow USDC against your crypto"
              │    │ Your loan: $500.00
              │    │ Interest rate: 6.1%
              │    │ [Borrow More] [Pay Back]
              │    └── → Opens Borrow Page
              │
              └── 🔒 GUARANTEE Card
                   │ "Manage your locked crypto"
                   │ Locked: 0.021 BTC ($2,100)
                   │ Loan Safety: ████████░░ Safe
                   │ [Add More] [Unlock Some]
                   └── → Opens Collateral Page
```

### Earn Flow (Deposit / Withdraw)
```
Earn Page
    │
    ├── Tab: "Start Earning" (Deposit)
    │    │
    │    │ ┌──────────────────────────────────────┐
    │    │ │  How much USDC do you want to lend?  │
    │    │ │                                      │
    │    │ │  [$_______] USDC         [Use Max]   │
    │    │ │                                      │
    │    │ │  Available: $5,430.00 USDC            │
    │    │ │                                      │
    │    │ │  ── What you'll earn ──               │
    │    │ │  Annual rate:        4.2%             │
    │    │ │  Estimated daily:    ~$0.57            │
    │    │ │  Estimated monthly:  ~$17.50           │
    │    │ │                                      │
    │    │ │  ⓘ You can cash out anytime.          │
    │    │ │    No lock-up period.                 │
    │    │ │                                      │
    │    │ │  [  Start Earning  ]  ← single button │
    │    │ └──────────────────────────────────────┘
    │    │
    │    │ Behind the scenes (hidden from user):
    │    │   Step 1/2: Approving USDC... (progress bar)
    │    │   Step 2/2: Depositing... (progress bar)
    │    │   ✅ Done! You're now earning 4.2% on $1,000
    │    │
    │    └── Success toast → back to Lending Hub
    │
    └── Tab: "Cash Out" (Withdraw)
         │
         │ ┌──────────────────────────────────────┐
         │ │  How much do you want to withdraw?   │
         │ │                                      │
         │ │  [$_______] USDC         [Use Max]   │
         │ │                                      │
         │ │  Your earnings balance: $1,240.00     │
         │ │  Available to withdraw: $1,200.00     │
         │ │  (some funds may be lent out)         │
         │ │                                      │
         │ │  ── Summary ──                        │
         │ │  You will receive:  $1,000.00 USDC    │
         │ │  Network fee:       ~$0.02            │
         │ │                                      │
         │ │  [  Cash Out  ]                       │
         │ └──────────────────────────────────────┘
         │
         │ Error states (human-readable):
         │   "Pool is temporarily paused. Try again later."
         │   "You're trying to withdraw more than you deposited."
         │   "Not enough liquidity right now. Max: $X,XXX.XX"
         │
         └── Success toast → back to Lending Hub
```

### Borrow Flow (Borrow / Repay)
```
Borrow Page
    │
    ├── Tab: "Borrow"
    │    │
    │    │ ┌──────────────────────────────────────┐
    │    │ │  How much USDC do you want to borrow?│
    │    │ │                                      │
    │    │ │  [$_______] USDC         [Use Max]   │
    │    │ │                                      │
    │    │ │  ── Your borrowing power ──           │
    │    │ │  Guarantee value:   $2,100.00         │
    │    │ │  Already borrowed:  $500.00            │
    │    │ │  Still available:   $900.00            │
    │    │ │                                      │
    │    │ │  ── After this loan ──                │
    │    │ │  Total owed:        $1,400.00          │
    │    │ │  Interest rate:     6.1% / year        │
    │    │ │  Daily interest:    ~$0.23              │
    │    │ │                                      │
    │    │ │  Loan Safety:                         │
    │    │ │  ████████░░ Safe  →  ██████░░░░ Caution│
    │    │ │                                      │
    │    │ │  ⚠️ If your safety drops below the   │
    │    │ │  red zone, your guarantee may be sold │
    │    │ │  automatically to repay the loan.     │
    │    │ │                                      │
    │    │ │  [  Borrow USDC  ]                    │
    │    │ └──────────────────────────────────────┘
    │    │
    │    └── Success toast → back to Lending Hub
    │
    └── Tab: "Pay Back"
         │
         │ ┌──────────────────────────────────────┐
         │ │  How much do you want to pay back?   │
         │ │                                      │
         │ │  [$_______] USDC                     │
         │ │                                      │
         │ │  Quick amounts:                      │
         │ │  [ 25% ]  [ 50% ]  [ Pay All ]       │
         │ │                                      │
         │ │  Outstanding loan: $1,400.00           │
         │ │  Wallet balance:   $5,430.00 USDC     │
         │ │                                      │
         │ │  ── After paying back ──              │
         │ │  Remaining loan:   $900.00             │
         │ │  Interest saved:   ~$0.08/day          │
         │ │                                      │
         │ │  Loan Safety:                         │
         │ │  ██████░░░░ Caution → ████████░░ Safe │
         │ │                                      │
         │ │  [  Pay Back  ]                       │
         │ └──────────────────────────────────────┘
         │
         │ Behind the scenes (hidden):
         │   Step 1/2: Approving USDC...
         │   Step 2/2: Paying back...
         │   ✅ Done! Remaining loan: $900.00
         │
         └── Success toast → back to Lending Hub
```

### Collateral (Guarantee) Flow
```
Collateral Page
    │
    ├── Tab: "Add Guarantee"
    │    │
    │    │ ┌──────────────────────────────────────┐
    │    │ │  Lock more [BTC] to secure your loan │
    │    │ │                                      │
    │    │ │  [$_______] BTC          [Use Max]   │
    │    │ │                                      │
    │    │ │  Wallet balance: 0.05 BTC ($5,000)   │
    │    │ │  Currently locked: 0.021 BTC ($2,100) │
    │    │ │                                      │
    │    │ │  ── After adding ──                   │
    │    │ │  Total locked:     0.031 BTC ($3,100) │
    │    │ │  Borrowing power:  +$500              │
    │    │ │                                      │
    │    │ │  Loan Safety:                         │
    │    │ │  ██████░░░░ Caution → ██████████ Safe │
    │    │ │                                      │
    │    │ │  [  Lock Guarantee  ]                 │
    │    │ └──────────────────────────────────────┘
    │    │
    │    │ Behind the scenes (hidden):
    │    │   Step 1/2: Approving BTC...
    │    │   Step 2/2: Locking guarantee...
    │    │   ✅ Done! Your loan is now safer.
    │    │
    │    └── Success toast → back to Lending Hub
    │
    └── Tab: "Unlock Guarantee"
         │
         │ ┌──────────────────────────────────────┐
         │ │  Unlock some of your guaranteed [BTC]│
         │ │                                      │
         │ │  [$_______] BTC          [Safe Max]  │
         │ │                                      │
         │ │  Currently locked:  0.031 BTC ($3,100)│
         │ │  Outstanding loan:  $1,400.00          │
         │ │  Safe to unlock:    0.010 BTC ($1,000) │
         │ │                                      │
         │ │  ── After unlocking ──                │
         │ │  Remaining locked: 0.021 BTC ($2,100) │
         │ │                                      │
         │ │  Loan Safety:                         │
         │ │  ██████████ Safe → ██████░░░░ Caution │
         │ │                                      │
         │ │  ⚠️ Unlocking too much may put your  │
         │ │  loan at risk of auto-repayment.     │
         │ │                                      │
         │ │  [  Unlock  ]                         │
         │ └──────────────────────────────────────┘
         │
         │ Error states:
         │   "You can't unlock this much — your loan would become unsafe."
         │   "Max you can safely unlock: 0.010 BTC ($1,000)"
         │
         └── Success toast → back to Lending Hub
```

### Liquidation (Advanced Only)
```
Lending Hub → scroll to bottom → "Advanced: Liquidation Opportunities"
    │ ⓘ "Earn a bonus by helping repay risky loans"
    │ (collapsed by default — click to expand)
    │
    └── Liquidation Page
         │
         ├── Explained header:
         │    "When someone's loan becomes unsafe (safety below 1.0),
         │     you can repay part of their loan and receive their
         │     guarantee crypto at a discount (bonus: X.X%)."
         │
         ├── Summary bar:
         │    Your USDC balance: $5,430.00
         │    Liquidation bonus: 6.5%
         │    Max repayable per position: 40%
         │
         ├── Risky Positions Table
         │    ┌──────────┬──────────┬─────────────┬────────┬────────┐
         │    │ Borrower │ Owes     │ Guarantee   │ Safety │ Action │
         │    ├──────────┼──────────┼─────────────┼────────┼────────┤
         │    │ 0xab...12│ $2,400   │ 0.03 BTC    │ 0.92🔴│ [View] │
         │    │ 0xcd...34│ $1,800   │ 0.02 BTC    │ 0.85🔴│ [View] │
         │    └──────────┴──────────┴─────────────┴────────┴────────┘
         │    [← Previous]              [Next →]
         │
         └── Position Detail (on click "View")
              │
              │ ┌──────────────────────────────────────┐
              │ │  Liquidation Opportunity              │
              │ │                                      │
              │ │  Borrower: 0xab...12                 │
              │ │  Loan safety: 0.92 🔴 Danger         │
              │ │  Total debt: $2,400.00                │
              │ │  Guarantee: 0.03 BTC ($3,000)        │
              │ │  Max you can repay: $960 (40%)       │
              │ │                                      │
              │ │  How much to repay?                  │
              │ │  [$_______] USDC        [Use Max]    │
              │ │                                      │
              │ │  ── You will receive ──              │
              │ │  BTC amount:    0.0102 BTC           │
              │ │  BTC value:     $1,020.00             │
              │ │  Your profit:   $60.00 (6.5%)        │
              │ │                                      │
              │ │  [  Repay & Collect  ]                │
              │ └──────────────────────────────────────┘
              │
              │ Behind the scenes:
              │   Step 1/2: Approving USDC...
              │   Step 2/2: Liquidating...
              │   ✅ Done! You received 0.0102 BTC
              │
              └── Success toast → back to list
```

---

## 4. Page-by-Page Design Specs

### 4.1 Lending Hub (Entry Point)

**Replaces:** Current `LendingProtocol.vue` with its 4 info cards + 7 modals  
**Location:** `MyWalletTabs` → "Lending" segment → full-page view (not a modal)

#### Layout (Desktop)
```
┌─────────────────────────────────────────────────────────────────┐
│                     YOUR LENDING OVERVIEW                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐   │
│  │ Earning   │ │ Borrowed │ │ Locked   │ │ Loan Safety      │   │
│  │ $1,240    │ │ $500     │ │ $2,100   │ │ ████████░░ Safe  │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│  SELECT A POOL                                                   │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐   │
│  │ ₿ Bitcoin Pool   │ │ ◆ Ethereum Pool │ │ ✕ XRP Pool      │   │
│  │ Earn 4.2%        │ │ Earn 3.8%       │ │ Earn 5.1%       │   │
│  │ Borrow 6.1%      │ │ Borrow 5.5%     │ │ Borrow 7.0%     │   │
│  │ ─────────────── │ │ ─────────────── │ │ ─────────────── │   │
│  │ Total in pool:   │ │ Total in pool:  │ │ Total in pool:  │   │
│  │ $2.4M            │ │ $1.8M           │ │ $890K           │   │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐       │
│  │  💰 EARN       │  │  🏦 BORROW    │  │  🔒 GUARANTEE │       │
│  │               │  │               │  │               │       │
│  │ Your: $1,240  │  │ Your: $500    │  │ Locked: $2.1K │       │
│  │ Rate: 4.2%    │  │ Rate: 6.1%    │  │ Safety: Safe  │       │
│  │               │  │               │  │               │       │
│  │ [Earn] [Out]  │  │ [Borrow][Pay] │  │ [Add] [Free]  │       │
│  └───────────────┘  └───────────────┘  └───────────────┘       │
├─────────────────────────────────────────────────────────────────┤
│  📊 Market Statistics (expandable, collapsed by default)        │
│  ► Total Supply: $2.4M | Total Borrowed: $1.2M | Util: 50%     │
├─────────────────────────────────────────────────────────────────┤
│  ⚡ Advanced: Liquidation Opportunities (collapsed)              │
│  ► "Earn a bonus by repaying risky loans"                       │
└─────────────────────────────────────────────────────────────────┘
```

#### Key Design Decisions
- **Pool selector** uses human names (Bitcoin Pool, not WBTC_USDC)
- **Market statistics** hidden by default — expandable for power users
- **Liquidation** fully collapsed — only visible when user explicitly opens it
- **All dollar values primary**, token amounts secondary in lighter text
- **Loan Safety meter** uses gradient color bar (green→yellow→red) instead of raw number

---

### 4.2 Earn Page (Deposit/Withdraw)

**Replaces:** `DepositModal.vue` + `WithDrawModal.vue`  
**Type:** Full page with two tabs, not a modal

#### "Start Earning" Tab — What Changes From Current
| Current (DepositModal) | Proposed |
|------------------------|----------|
| Shows "shares you'll receive" | Hidden — only shows dollar amounts |
| Shows "exchange rate USDC/share" | Hidden |
| Two separate buttons: "Approve USDC" + "Deposit" | Single "Start Earning" button with auto-stepper |
| Error: "SUPPLY_CAP" | "This pool is temporarily full. Try again later or try a smaller amount." |
| Error: "ZERO_SHARES" | "Amount too small. Please enter at least $1.00" |
| APY shown as single number | APY + estimated daily + monthly earnings shown |

#### "Cash Out" Tab — What Changes From Current
| Current (WithDrawModal) | Proposed |
|-------------------------|----------|
| Shows "shares to burn" | Hidden |
| "Your Deposits" as single number | "Your earnings balance" with clear available amount |
| Error: "INSUF_SHARES" | "You're trying to withdraw more than you deposited." |
| Error: "INSUF_CASH" | "Not enough funds available right now. You can withdraw up to $X,XXX. Try again later for the full amount." |
| No explanation of liquidity | Small note: "Some of your funds are currently lent out. Max available: $X" |

---

### 4.3 Borrow Page (Borrow/Repay)

**Replaces:** `BorrowModal.vue` + `RepayModal.vue`  
**Type:** Full page with two tabs

#### "Borrow" Tab — What Changes
| Current (BorrowModal) | Proposed |
|----------------------|----------|
| "Borrow Limit" / "Already Borrowed" / "Available to Borrow" | "Your borrowing power" section with visual bar |
| "Health Factor: 1.47 → 1.23" | Visual Safety Meter with colors + plain text |
| "Daily Interest: ~$0.23" (buried) | Prominent: "This will cost you ~$0.23/day" |
| Warning: "Liquidation occurs at HF < 1.0" (tiny text) | Full-width yellow warning banner with simple language |
| Error: "INSUF_COLL" | "You need more guarantee locked first. Go to Guarantee → Add More." |

#### "Pay Back" Tab — What Changes
| Current (RepayModal) | Proposed |
|---------------------|----------|
| Two buttons: "Approve" + "Repay" | Single "Pay Back" button with auto-stepper |
| Shows interest savings as tiny text | Prominent: "You'll save $X.XX per day in interest" |
| HF shown as numbers | Visual Safety Meter showing improvement (arrow from yellow to green) |

---

### 4.4 Collateral Page (Add/Remove)

**Replaces:** `CollateralModal.vue` + `RemoveCollateral.vue`  
**Type:** Full page with two tabs

#### "Add Guarantee" Tab — What Changes
| Current (CollateralModal) | Proposed |
|--------------------------|----------|
| "Amount of tokens to collateral" placeholder | "How much [BTC] do you want to lock?" |
| "New Borrow Limit" | "Borrowing power: +$500" (focus on what it gives them) |
| Two buttons: "Approve" + "Add Collateral" | Single "Lock Guarantee" with auto-stepper |
| No explanation of why | Inline card: "Why lock crypto? It secures your loan and lets you borrow more." |

#### "Unlock Guarantee" Tab — What Changes
| Current (RemoveCollateral) | Proposed |
|---------------------------|----------|
| "Max" button (risky — could take position to danger) | "Safe Max" button (only unlocks what's safe) |
| Error: "UNHEALTHY — Position would become unhealthy" | "Unlocking this much would make your loan unsafe. Safe max: 0.010 BTC ($1,000)" |
| Status shown as "✓ Safe" / "✗ Unsafe" | Full Safety Meter visualization |

---

### 4.5 Liquidation Page (Advanced)

**Replaces:** `LiquidationModal.vue` → `LiquidationList.vue` + `LiquidationCurrent.vue`  
**Type:** Collapsed section at bottom of Lending Hub, expands to full sub-page

#### What Changes
| Current | Proposed |
|---------|----------|
| Same-level modal as Deposit/Withdraw | Hidden behind "Advanced" toggle, collapsed by default |
| No explanation of what liquidation is | Full explainer paragraph at top |
| Table columns: Address, Debt, Collateral, HF | Address, Owes, Guarantee, Safety (color-coded) |
| "Approve USDC" + "Liquidate" buttons | Single "Repay & Collect" with auto-stepper |
| Profit shown in raw bigint format | Clear "$60.00 profit (6.5% bonus)" |
| Error: "BAD_ADDR" | "This wallet address is invalid. Please go back and try another." |
| Error: "HEALTHY" | "Good news — this loan is now safe and no longer needs liquidation." |
| No guidance on what the user is actually doing | Step-by-step: "You repay their loan → You get their crypto at a discount" |

---

## 5. Approval Abstraction Strategy

### Current UX (Confusing)
```
User clicks "Deposit" modal →
  Sees two buttons: [Approve USDC] and [Deposit] (greyed out)
  Clicks [Approve USDC] →
    MetaMask popup (what is this? why?) →
    Waits for confirmation →
  Now [Deposit] becomes active →
  Clicks [Deposit] →
    Another MetaMask popup →
    Done
```

### Proposed UX (Simplified)
```
User clicks [Start Earning] →
  Button changes to progress stepper:
  ┌──────────────────────────────────────┐
  │  ● Preparing...  ○ Confirming...     │
  │  ████████░░░░░░░░░░░░░░░░░░  Step 1/2│
  └──────────────────────────────────────┘
  MetaMask popup appears (1 of 2) → user confirms →
  ┌──────────────────────────────────────┐
  │  ✓ Prepared     ● Depositing...     │
  │  ████████████████░░░░░░░░░░  Step 2/2│
  └──────────────────────────────────────┘
  MetaMask popup appears (2 of 2) → user confirms →
  ┌──────────────────────────────────────┐
  │  ✅ Done! You're now earning 4.2%    │
  │     on $1,000 USDC                   │
  └──────────────────────────────────────┘
```

### Implementation Notes
- Wrap `approve()` + `action()` calls in a single `async` function
- Show a 2-step progress indicator during the sequence
- If approval fails, show: "Transaction cancelled. No funds were moved."
- If deposit fails after approval, show: "The deposit didn't go through, but your funds are safe. Try again."
- Skip the approval step if allowance is already sufficient (check on page load)

---

## 6. Error Message Rewriting

### Full Error Code → Human Message Mapping

| Error Code | Current Message | Proposed Message |
|-----------|----------------|-----------------|
| `PAUSED` | "The lending pool is currently paused." | "This pool is temporarily unavailable. Please try again in a few minutes." |
| `ZERO_SHARES` | "Deposit amount is too small to receive shares" | "Amount too small. Please enter at least $1.00." |
| `SUPPLY_CAP` | "Deposit amount is too large, max(X USDC)" | "This pool is full right now. Maximum you can add: $X,XXX.XX" |
| `INSUF_SHARES` | "Not enough shares to withdraw. Max X USDC" | "You're trying to withdraw more than you deposited. Your balance: $X,XXX.XX" |
| `INSUF_CASH` | "Protocol doesn't have enough liquidity right now. Max(X USDC)" | "Not enough funds available for withdrawal right now. You can withdraw up to $X,XXX.XX. The rest will become available as other borrowers repay." |
| `INSUF_COLL` | "Not enough collateral to remove." | "You can't unlock this much — your loan would become unsafe. Safe max: X.XX BTC ($X,XXX)" |
| `UNHEALTHY` | "Position would become unhealthy." | "Unlocking this much would put your loan at risk. The most you can safely unlock is X.XX BTC ($X,XXX)" |
| `BAD_ADDR` | "Invalid wallet address." | "Something went wrong with this position. Please go back and try another." |
| `NO_DEBT` | "This position is healthy and cannot be liquidated." | "Good news — this borrower has repaid their loan. No liquidation needed." |
| `HEALTHY` | "This position is healthy and cannot be liquidated." | "This loan is now safe (their guarantee covers the debt). Look for positions with a red safety indicator." |
| `INSUF_COLL` (liquidation) | "Not enough collateral available for liquidation." | "There isn't enough guarantee to cover this liquidation. Try a smaller amount." |
| Generic error | "Something went wrong." | "Something didn't work. Your funds are safe. Please try again." |
| Network/RPC error | *(crashes or console.error)* | "We're having trouble connecting to the network. Check your internet and try again." |
| Insufficient gas | *(not always caught)* | "You need a small amount of POL for network fees. Your POL balance is too low." |

---

## 7. Tooltips & Education Layer

### First-Visit Experience
On first visit to Lending Hub, show a dismissible card:

```
┌──────────────────────────────────────────────────────┐
│  👋 New to Lending?                                   │
│                                                      │
│  Here's how it works in 30 seconds:                  │
│                                                      │
│  1. EARN — Lend your USDC to earn interest daily     │
│  2. BORROW — Put up crypto as guarantee, borrow USDC │
│  3. STAY SAFE — Keep your safety meter in the green  │
│                                                      │
│  That's it! Start with "Earn" if you just want to    │
│  grow your USDC.                                     │
│                                                      │
│  [Got it!]                              [Learn More] │
└──────────────────────────────────────────────────────┘
```

### Inline Tooltips (ⓘ icons)
| Element | Tooltip Text |
|---------|-------------|
| Annual Rate | "This is how much you'll earn (or owe) over a full year. The actual rate changes based on supply and demand." |
| Loan Safety | "This shows how safe your loan is. Green (>1.5) = safe. Yellow (1.0-1.5) = be careful. Red (<1.0) = your guarantee may be sold to repay the loan." |
| Guarantee | "Crypto you lock as a promise to repay your loan. If you can't repay, this crypto is used instead." |
| Available to Borrow | "Based on the value of your locked guarantee. You can borrow up to ~70% of its value (varies by pool)." |
| Network Fee | "A small fee paid to the Polygon network to process your transaction. Usually less than $0.01." |
| Available Liquidity | "The total amount of USDC currently available in the pool. If many people are borrowing, less may be available for withdrawal." |

---

## 8. Mobile Considerations

### Current Issues on Mobile
- Modals are cramped (all 7 modals rendered at `class="mx-2"`)
- 4 position cards in a horizontal row don't fit on mobile
- Collateral flip-card animation is unintuitive on touch
- Liquidation table too wide for small screens

### Proposed Mobile Layout

#### Lending Hub (Mobile)
```
┌─────────────────────────┐
│   YOUR LENDING OVERVIEW  │
│                         │
│ Earning     $1,240      │
│ Borrowed    $500        │
│ Locked      $2,100      │
│ Safety  ████████░░ Safe │
├─────────────────────────┤
│ ← Bitcoin | Ethereum | → │
│   Pool     Pool   (swipe)│
│  Earn 4.2%  Earn 3.8%   │
├─────────────────────────┤
│ ┌─────────────────────┐ │
│ │  💰 Earn             │ │
│ │  $1,240 at 4.2%     │ │
│ │  [Earn]  [Cash Out]  │ │
│ └─────────────────────┘ │
│ ┌─────────────────────┐ │
│ │  🏦 Borrow           │ │
│ │  $500 at 6.1%       │ │
│ │  [Borrow] [Pay Back] │ │
│ └─────────────────────┘ │
│ ┌─────────────────────┐ │
│ │  🔒 Guarantee        │ │
│ │  $2,100 locked       │ │
│ │  [Add]  [Unlock]     │ │
│ └─────────────────────┘ │
└─────────────────────────┘
```

#### Action Pages (Mobile)
- Full-screen pages (not modals)
- Large touch targets for buttons (min 48px height)
- Amount input with large numeric keyboard
- Safety meter takes full width
- Progress stepper shown inline below the button

#### Pool Selector (Mobile)
- Horizontal swipeable cards (Swiper) instead of side-by-side grid
- Each card shows pool name + both rates
- Selected card has gradient border

---

## Summary of Files to Create/Modify

| Action | File | Purpose |
|--------|------|---------|
| **Create** | `views/dashboard/lending/LendingHub.vue` | New entry point replacing `LendingProtocol.vue` |
| **Create** | `views/dashboard/lending/EarnPage.vue` | Deposit + Withdraw tabs |
| **Create** | `views/dashboard/lending/BorrowPage.vue` | Borrow + Repay tabs |
| **Create** | `views/dashboard/lending/CollateralPage.vue` | Add + Remove guarantee tabs |
| **Create** | `views/dashboard/lending/LiquidationPage.vue` | Advanced liquidation (list + detail) |
| **Create** | `components/lending/SafetyMeter.vue` | Reusable loan safety visualization |
| **Create** | `components/lending/ActionStepper.vue` | Approve + Action progress indicator |
| **Create** | `components/lending/PoolSelector.vue` | Human-friendly pool cards |
| **Create** | `components/lending/FirstVisitCard.vue` | Dismissible education card |
| **Modify** | `router/index.ts` | Add lending sub-routes under dashboard |
| **Modify** | `components/dashboard/my-wallet/MyWalletTabs.vue` | Point "Lending" tab to new LendingHub |
| **Deprecate** | `components/dashboard/my-wallet/lending/*` | All 13 current lending components |
| **Modify** | `composables/useLending.ts` | No changes needed — composable is clean |
| **Modify** | `utils/healthFactor.ts` | Add color/label mapping for Safety Meter |
