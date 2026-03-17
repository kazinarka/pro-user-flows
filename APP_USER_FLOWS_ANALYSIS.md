# PhilSocial DeFi — Complete App Analysis & User Flows

> **Generated:** March 17, 2026  
> **Stack:** Vue 3 + Ionic Vue + Pinia + TypeORM (SQLite) + ethers.js + TailwindCSS  
> **Blockchains:** Polygon (primary), Ethereum, BSC, Solana (+ testnets)  
> **Backend:** AWS (App Runner + Lambda/API Gateway + S3)

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [User Flow 1 — Registration & Onboarding](#2-user-flow-1--registration--onboarding)
3. [User Flow 2 — Authentication (Login)](#3-user-flow-2--authentication-login)
4. [User Flow 3 — Password Reset](#4-user-flow-3--password-reset)
5. [User Flow 4 — Wallet Creation (Native HD Wallet)](#5-user-flow-4--wallet-creation-native-hd-wallet)
6. [User Flow 5 — Wallet Import (Seed Phrase)](#6-user-flow-5--wallet-import-seed-phrase)
7. [User Flow 6 — Wallet Connection (MetaMask)](#7-user-flow-6--wallet-connection-metamask)
8. [User Flow 7 — Subscription Payment](#8-user-flow-7--subscription-payment)
9. [User Flow 8 — Dashboard Home](#9-user-flow-8--dashboard-home)
10. [User Flow 9 — My Wallet (Assets & Balances)](#10-user-flow-9--my-wallet-assets--balances)
11. [User Flow 10 — Send / Transfer Crypto](#11-user-flow-10--send--transfer-crypto)
12. [User Flow 11 — Receive Crypto](#12-user-flow-11--receive-crypto)
13. [User Flow 12 — Swap (USDT → PHL)](#13-user-flow-12--swap-usdt--phl)
14. [User Flow 13 — Buy PHL (Fiat On-Ramp)](#14-user-flow-13--buy-phl-fiat-on-ramp)
15. [User Flow 14 — DeFi Stable Pool (Staking)](#15-user-flow-14--defi-stable-pool-staking)
16. [User Flow 15 — Lending Protocol](#16-user-flow-15--lending-protocol)
17. [User Flow 16 — My Referrals](#17-user-flow-16--my-referrals)
18. [User Flow 17 — My PRO-Rewards](#18-user-flow-17--my-pro-rewards)
19. [User Flow 18 — My Causes](#19-user-flow-18--my-causes)
20. [User Flow 19 — My Give Back](#20-user-flow-19--my-give-back)
21. [User Flow 20 — My Followers](#21-user-flow-20--my-followers)
22. [User Flow 21 — My Posts](#22-user-flow-21--my-posts)
23. [User Flow 22 — My Airdrops (Campaigns)](#23-user-flow-22--my-airdrops-campaigns)
24. [User Flow 23 — Campaign Goal Pages (Public)](#24-user-flow-23--campaign-goal-pages-public)
25. [User Flow 24 — Account Settings](#25-user-flow-24--account-settings)
26. [User Flow 25 — Transaction History](#26-user-flow-25--transaction-history)
27. [User Flow 26 — PhilAds (Advertising)](#27-user-flow-26--philads-advertising)
28. [Implementation Status Summary](#28-implementation-status-summary)
29. [Known Issues & Technical Debt](#29-known-issues--technical-debt)

---

## 1. Architecture Overview

### App Structure
```
┌──────────────────────────────────────────────────┐
│                    App.vue                        │
│  ┌─────────────┐ ┌──────────────────────────────┐│
│  │ HeaderMain   │ │ HeaderDashboard              ││
│  │ (welcome)    │ │ (dashboard pages)            ││
│  └─────────────┘ └──────────────────────────────┘│
│  ┌─────────────┐ ┌──────────────────────────────┐│
│  │ HeaderMobile │ │ Sidebar (dashboard)          ││
│  │ (always)     │ │                              ││
│  └─────────────┘ └──────────────────────────────┘│
│  ┌──────────────────────────────────────────────┐│
│  │                RouterView                     ││
│  │  ┌─────────┐ ┌───────────┐ ┌──────────────┐  ││
│  │  │Register │ │  Wallet   │ │  Dashboard   │  ││
│  │  │  Pages  │ │   Pages   │ │    Pages     │  ││
│  │  └─────────┘ └───────────┘ └──────────────┘  ││
│  └──────────────────────────────────────────────┘│
│  ┌──────────────────────────────────────────────┐│
│  │ Global Modals: PasswordModal,                 ││
│  │ WaitForConfirmation, WalletConnector          ││
│  └──────────────────────────────────────────────┘│
└──────────────────────────────────────────────────┘
```

### Route Guard Logic
```
Every route navigation → beforeResolve:
  1. If requiresAuth → check JWT validity via useAuthStore.iAmLoggedIn()
     → redirect to /login if invalid
  2. If requiresWallet → check wallet exists via useWalletStore.existWallet()
     → redirect to /home if no wallet
  3. If requiresPayment → check subscription via useIsPayment.isPayment()
     → redirect to /payment if needed
```

### Data Layer Summary
| Layer | Technology | Purpose |
|-------|-----------|---------|
| REST API | OpenAPI auto-gen SDK (`@hey-api/openapi-ts`) | Auth, social, causes, rewards, metrics, posts, campaigns |
| Blockchain | ethers.js + custom adapters | Wallet ops, transfers, DeFi, payments, rewards claims |
| Local DB | TypeORM + Capacitor SQLite | Wallet storage, asset metadata, balances, tx cache |
| State | Pinia (20+ stores) | Reactive app state |
| Auth Tokens | localStorage | JWT access/refresh/id tokens |

---

## 2. User Flow 1 — Registration & Onboarding

### Status: ✅ FULLY IMPLEMENTED

### Flow
```
Welcome Page (/) → "Get Started"
    ↓
Register Page (/register)
    ├── Google Sign-In (OAuth popup) ──→ Social register API ──→ Dashboard
    ├── Apple Sign-In (popup) ──→ Social register API ──→ Dashboard
    └── Email Sign-Up (/signup)
         ↓
    Sign-Up Form (email, password, confirm)
         │ reCAPTCHA v3 verification
         │ POST /auth/signup
         ↓
    Email Verification (/signup/email)
         │ "Check your email" confirmation screen
         ↓
    Login (/login)
         ↓
    MFA Setup (/verification) — if first login
         │ TOTP QR code generation (authStore.fmaAssociate)
         │ User scans QR, enters 6-digit OTP
         │ authStore.mfaVerify()
         ↓
    Interest Selection (/interests)
         │ Fetches interests from API (UserClient.getInterestsAllGet)
         │ User selects 4-15 interests
         │ COPPA age confirmation toggle
         ↓
    Profile Completion (/onboarding)
         │ First name, last name, gender, location, DOB, phone
         │ useUserProfile.createUserProfile()
         ↓
    Dashboard (/dashboard)
```

### Components Involved
| Component | File | Role |
|-----------|------|------|
| WelcomeView | `views/register/WelcomeView.vue` | Landing page with CTA |
| RegisterProfile | `views/register/RegisterProfile.vue` | Google/Apple/Email registration |
| MySignUp | `components/register/signup/MySignUp.vue` | Email registration form |
| SuccesSignUp | `components/register/signup/SuccesSignUp.vue` | Email confirmation screen |
| MfaVerify | `views/register/MfaVerify.vue` | TOTP setup with QR code |
| InterestProfile | `views/register/InterestProfile.vue` | Interest selection grid |
| OnboardingProfile | `views/register/OnboardingProfile.vue` | Profile completion form |

### API Calls (Real)
- `POST /auth/signup` — account creation
- `POST /auth/social-register` — Google/Apple OAuth
- `POST /auth/mfa-associate` — generate TOTP secret
- `POST /auth/mfa-verify` — verify OTP code
- `GET /interests/all` — fetch available interests
- `POST /user/profile` — create user profile

---

## 3. User Flow 2 — Authentication (Login)

### Status: ✅ FULLY IMPLEMENTED

### Flow
```
Login Page (/login)
    ├── Email + Password login
    │    │ reCAPTCHA v3 verification
    │    │ POST /auth/signin
    │    ├── MFA challenge detected → MFA Modal
    │    │    │ Enter 6-digit TOTP code
    │    │    │ POST /auth/signin-confirm
    │    │    └── Dashboard
    │    └── No MFA → Dashboard
    ├── Google Sign-In → Social register → Dashboard
    └── Apple Sign-In → Social register → Dashboard
```

### Components Involved
| Component | File | Role |
|-----------|------|------|
| LoginView | `views/register/LoginView.vue` | Wrapper |
| MyLogin | `components/register/login/MyLogin.vue` | Login form with social auth |
| MfaVerifyModal | `views/register/MfaVerifyModal.vue` | MFA OTP entry modal |

### API Calls (Real)
- `POST /auth/signin` — email/password login
- `POST /auth/signin-confirm` — MFA OTP verification
- `POST /auth/social-register` — social OAuth

### Auth Token Storage
- `localStorage.accessToken` — JWT access token
- `localStorage.refreshToken` — JWT refresh token
- `localStorage.idToken` — JWT identity token

---

## 4. User Flow 3 — Password Reset

### Status: ✅ FULLY IMPLEMENTED

### Flow
```
Login Page → "Forgot Password?"
    ↓
Reset Page (/reset)
    │ Enter email → Send Verification Code
    │ authStore.forgotPassword()
    ↓
    Enter OTP code (6 digits)
    ↓
    Enter new password + confirm
    │ authStore.confirmResetPassword()
    ↓
Success Page (/reset/success)
    │ "Login to my account" button
    └── Login Page
```

### API Calls (Real)
- `POST /auth/reset-password` — send verification code
- `POST /auth/confirm-reset-password` — set new password with OTP

---

## 5. User Flow 4 — Wallet Creation (Native HD Wallet)

### Status: ✅ FULLY IMPLEMENTED

### Flow
```
Home Page (/home) → "Create a new Philcoin Wallet"
    ↓
Terms & Conditions (/conditions)
    │ 3 checkboxes about seed phrase custody
    ↓
Choose Seed Length (/seed-length)
    │ 12 or 24 words
    ↓
Create Seed (/create-seed/:seedLength)
    │ HdWallet.createMnemonic(seedLength)
    │ Displays mnemonic words in Swiper carousel
    │ Derives address from seed
    ↓
Verify & Create Wallet (/create-wallet-from-seed/:address)
    │ User re-enters seed words for verification
    │ Words validated against BIP39 English wordlist
    │ Address validated against expected
    │ PasswordModal → set wallet password
    │ walletStore.createNativeWallet(mnemonic, password)
    │   → Encrypts mnemonic with AES
    │   → Stores EncryptedWalletEntity in SQLite
    │   → Creates WalletEntity for each blockchain
    │   → Seeds blockchain configs into DB
    ↓
Dashboard (/dashboard)
```

### Key Services
| Service | File | Role |
|---------|------|------|
| HdWallet | `services/hdwallet.ts` | BIP39 mnemonic generation, address derivation |
| WalletStore | `stores/WalletStore.ts` | Wallet persistence, encryption, DB management |
| PasswordModal | `components/PasswordModal.vue` | Password entry/confirmation |

### Database Operations
- `EncryptedWalletEntity` — stores AES-encrypted mnemonic
- `WalletEntity` — per-blockchain derived addresses
- `BlockchainEntity` — seeded from `blockchainConfig.ts`
- `NodesEntity` — RPC endpoints per chain
- `AssetsEntity` — tokens per chain from metadata JSON files

---

## 6. User Flow 5 — Wallet Import (Seed Phrase)

### Status: ✅ FULLY IMPLEMENTED

### Flow
```
Home Page (/home) → "I already have a Philcoin Wallet"
    ↓
Import Wallet (/import-wallet)
    │ Enter 12/24 seed words (Swiper carousel input)
    │ Each word validated against BIP39 wordlist
    │ PasswordModal → set wallet password
    │ walletStore.createNativeWallet(mnemonic, password)
    ↓
Dashboard (/dashboard)
```

### Notes
- Uses the same `CreateOrImportWalletView.vue` component as wallet creation
- The route name `ImportWallet` does not pass an expected address prop, so the address-matching validation step is skipped

---

## 7. User Flow 6 — Wallet Connection (MetaMask)

### Status: ✅ FULLY IMPLEMENTED

### Flow
```
Home Page (/home) → "Connect My Wallet"
    ↓
WalletConnector Modal
    │ Shows MetaMask & PhilSocial options
    │ User clicks MetaMask
    │   → window.ethereum.request({ method: 'eth_requestAccounts' })
    │   → walletStore.connectWallet('Metamask', address)
    │   → Stores connectedAddress in localStorage
    │   → Stores provider type in localStorage
    │   → useIsPayment.isPayment() check
    │     ├── Payment needed → /payment
    │     └── No payment → /dashboard
    ↓
Dashboard or Payment
```

### MetaMask Event Handling (App.vue)
- **Account switch**: `window.ethereum.on('accountsChanged')` → updates `connectedAddress`, refreshes balances
- **Disconnect**: If accounts array empty → `walletStore.clearWalletSession()`
- **Permission revoke**: `wallet_revokePermissions` on disconnect

### Wallet Unlock Flow (App startup)
```
App.vue onMounted:
    │ walletStore.hasStoredNativeWallet()
    ├── Yes + wallet locked → PasswordModal.showUnlockModal()
    │    ├── Correct password → walletStore.unlockWallet(password) → continue
    │    └── Wrong password → walletStore.clearWalletSession()
    └── No → continue
```

---

## 8. User Flow 7 — Subscription Payment

### Status: ✅ FULLY IMPLEMENTED (Blockchain-based)

### Flow
```
Route guard detects payment needed → /payment
    ↓
Payment Page (/payment)
    │ Shows connected wallet address
    │ Shows available balance (PHL or USDT)
    │ Price: $9.99/month
    │
    │ Select payment token: PHL or USDT
    │   (Credit Card, Apple Pay, Google Pay, Stripe, PayPal — all DISABLED)
    │
    ├── First-time subscription:
    │    │ 1. Approve ERC20 spending
    │    │    → usePaymentProcessor.calculateRegisterFee()
    │    │    → useWallet.sentApprove() (ERC20 approve tx)
    │    │ 2. Register subscription
    │    │    → usePaymentProcessor.register()
    │    │    → On-chain tx to PaymentsProcessor contract
    │    │ 3. Poll for confirmation (setInterval 5s)
    │    └── Dashboard
    │
    ├── Paused subscription:
    │    │ 1. Resume subscription
    │    │    → usePaymentProcessor.resume()
    │    └── Dashboard
    │
    └── Wallet-already-linked error:
         │ Modal: "This wallet is already linked to another account"
         └── Close modal

Contract: 0xB1d569605C6E9B5D0cb1F6921D75DBC5Bbeb9FBB (Polygon)
```

### Smart Contract Methods
- `register(uuid, token)` — register new subscription
- `resume(uuid)` — resume paused subscription
- `cancel(uuid)` — cancel subscription
- `getUserByUUID(uuid)` → `{ userSubscription, lastDueDate, ... }`
- `getPaymentTokens()` → list of accepted tokens

---

## 9. User Flow 8 — Dashboard Home

### Status: ✅ FULLY IMPLEMENTED

### Flow
```
Dashboard Home (/dashboard/home)
    │
    │ Parallel API fetches on mount:
    │   1. metricsStore.fetchMetrics()
    │   2. metricsStore.fetchMetricsReach()
    │   3. rewardsStore.fetchRewardsSocialTasks()
    │   4. rewardsStore.fetchRewardsGivenBack()
    │   5. causesStore.fetchCausesSummary()
    │   6. referralsStore.fetchReferralsSummary()
    │   7. nevoStore.fetchNevoPost()
    │
    │ Left Column:
    │   ├── Bar Chart: Account Activity (7-day)
    │   ├── Bar Chart: Post Engagement (7-day)
    │   ├── Bar Chart: Time Spent (7-day)
    │   ├── Account Health Gauge (semi-circle doughnut)
    │   └── Social Rewards Qualification Checklist
    │
    │ Right Column:
    │   ├── Summary Cards (5): Total Referrals, PRO-Rewards,
    │   │    Claimable, Given Back, Causes
    │   ├── My Referrals Card (with Claim button)
    │   ├── Transaction Count Card (this week vs last)
    │   └── Account Reach Chart (followers vs non-followers)
    │
    └── "Download Report" button (⚠️ UI-ONLY — no handler)
```

### Data Sources
| Widget | Store | API Endpoint |
|--------|-------|-------------|
| Activity chart | metricsStore | `/metrics/activity` |
| Engagement chart | metricsStore | `/metrics/engagement` |
| Time Spent chart | metricsStore | `/metrics/impressions` |
| Health gauge | useUserProfile | `/user/health` |
| Social rewards | rewardsStore | `/rewards/social-tasks` |
| Referrals summary | referralsStore | `/referrals/summary` |
| Given back total | rewardsStore | `/rewards/given-back` |
| Causes summary | causesStore | `/causes/summary` |
| Reach chart | metricsStore | `/metrics/reach` |
| Transactions | WalletStore + HdWallet | Blockchain RPC (tx history) |

---

## 10. User Flow 9 — My Wallet (Assets & Balances)

### Status: ✅ FULLY IMPLEMENTED

### Flow
```
My Wallet (/dashboard/my-wallet)
    │
    ├── TopSelector: Choose blockchain network
    │    │ Polygon, Ethereum, BSC, Solana (+ testnets)
    │    └── Updates allAssets/balances reactively
    │
    ├── BalanceCard: Total USD value + staked balance
    │    │ Real-time price from CoinGecko + on-chain DEX
    │    └── ⚠️ Staked balance hardcoded to 0
    │
    ├── ButtonGroup: Send / Receive / Swap / Buy PHL
    │    ├── Send → /dashboard/transfer/:address/:blockchain
    │    ├── Receive → ReceiveAddressModal (QR code)
    │    ├── Swap → /dashboard/swap/:address/:blockchain
    │    └── Buy PHL → Onramper iframe widget
    │
    ├── MyAssetsSlider: Horizontal token cards
    │
    └── MyWalletTabs (IonSegment):
         ├── Assets Tab: Token list with live balances
         │    └── TokensListTab — grid of token cards with Transfer button
         ├── DeFi Tab: StablePool deposit/redeem interface
         │    └── DefiTab — full DeFi staking workflow
         └── Lending Protocol Tab (Polygon only)
              └── LendingProtocol — full lending dashboard
```

### Balance Update Mechanism
1. `walletStore.updateAllAssets()` → seeds DB with blockchain-specific tokens
2. `walletStore.updateAllBalances()` → batch RPC calls via `balancesProxy` contract
3. `BalanceEntity` emits `UpdateBalanceEntity` event → UI reactively updates
4. Prices fetched via CoinGecko API + on-chain DEX quote for PHL

---

## 11. User Flow 10 — Send / Transfer Crypto

### Status: ✅ FULLY IMPLEMENTED

### Flow
```
Transfer Page (/dashboard/transfer/:address/:blockchain/:asset?/:type?)
    │
    │ 1. Select "From" address (dropdown of user's wallets)
    │ 2. Enter/paste recipient address
    │    │ Validates address format via walletStore.isValidAddress()
    │ 3. Enter amount (or click "Max")
    │    │ BigInt precision handling (6-18 decimals)
    │    │ Max: deducts estimated gas for native currency
    │ 4. Adjust fee priority (Low / Medium / High slider)
    │    │ walletStore.suggestedFee() → gets gas estimates
    │    │ Fee displayed in native + USD value
    │ 5. Click "Send" → Confirmation dialog
    │    │ Shows: from, to, amount, fee, total
    │ 6. Confirm
    │    │ useWallet.buildTransfer() → creates tx object
    │    │ useWallet.sendTransfer() → signs & broadcasts
    │    │ WaitForConfirmation modal → polls for confirmation
    │ 7. Transaction confirmed → success
    │
    └── Supported: Native currencies + ERC20 tokens on all chains
```

### Fee Calculation
- `suggestedFee()` returns `{ low, medium, high }` gas price tiers
- For native transfers: amount + gas checked against balance
- For ERC20 transfers: amount checked against token balance, gas checked against native balance

---

## 12. User Flow 11 — Receive Crypto

### Status: ✅ FULLY IMPLEMENTED

### Flow
```
ButtonGroup → "Receive" button
    ↓
ReceiveAddressModal (Teleport to #receive-modal)
    │ Select blockchain network
    │ Select wallet address (if multiple)
    │ Displays QR code (vue-qrcode)
    │ Copy address to clipboard
    └── Close
```

---

## 13. User Flow 12 — Swap (USDT → PHL)

### Status: ✅ FULLY IMPLEMENTED

### Flow
```
Swap Page (/dashboard/swap/:address/:blockchain)
    │
    │ Direction: USDT → PHL (one-way)
    │ Shows: current PHL price, available USDT balance
    │
    │ 1. Enter USDT amount (or click "Available" for max)
    │ 2. Check ERC20 allowance
    │    ├── Insufficient allowance:
    │    │    │ Calculate approve fee
    │    │    │ Click "Approve" → ERC20 approve tx
    │    │    └── Wait for confirmation
    │    └── Sufficient allowance: skip
    │ 3. Calculate swap fee
    │ 4. Check POL balance for gas
    │ 5. Click "Swap"
    │    │ walletStore.swapExactUSDTForPHLEncode()
    │    │ walletStore.buildTransaction(encoded)
    │    │ walletStore.sendTransaction()
    │ 6. Wait for confirmation
    │
    └── Uses hardcoded swap contract address on Polygon
```

---

## 14. User Flow 13 — Buy PHL (Fiat On-Ramp)

### Status: ✅ FULLY IMPLEMENTED (via 3rd party)

### Flow
```
ButtonGroup → "Buy PHL" button
    ↓
Opens Onramper widget in iframe
    │ URL: https://buy.onramper.com/?...
    │ Params: apiKey, defaultCrypto=pol_phil, walletAddress, mode=buy
    │ Onramper handles the full fiat→crypto purchase
    └── Closes on completion
```

### Notes
- Uses Onramper's hosted widget
- Pre-fills user's connected wallet address
- Limited to PHL token on Polygon network

---

## 15. User Flow 14 — DeFi Stable Pool (Staking)

### Status: ✅ FULLY IMPLEMENTED (Blockchain)

### Flow
```
My Wallet → DeFi Tab
    ↓
DefiTab Component
    │
    ├── View existing deposit positions
    │    │ For each position: amount, deposit date, unlock date, time remaining
    │    ├── "Force Redeem" → forceWithdraw (early exit with penalty)
    │    └── "Redeem" → if matured:
    │         │ Check PHLD allowance
    │         │ Approve PHLD if needed
    │         └── sendRedeem() → on-chain
    │
    └── New Deposit
         │ Select asset (USDT)
         │ Enter amount
         │ Calculate gas fee
         │ 1. Approve USDT → ERC20 approve tx
         │ 2. Deposit → StablePool.deposit()
         └── Wait for confirmation

Contract: 0x95A8399f7cF8d4007D491b2F4822DA89b19c0de8 (Polygon)
```

### StablePool Methods Used
- `deposit(amount, asset)` — lock stablecoins
- `redeem(positionId)` — withdraw matured position
- `forceWithdraw(positionId)` — early withdrawal
- `getDepositPositions(address)` — view all positions
- `calculateAllowance(token)` — check ERC20 allowance

---

## 16. User Flow 15 — Lending Protocol

### Status: ✅ FULLY IMPLEMENTED (Blockchain)

### Flow
```
My Wallet → Lending Protocol Tab (Polygon only)
    ↓
LendingProtocol Component
    │
    ├── Pool Selector: WBTC-USDC / WXRP-USDC / WETH-USDC
    │
    ├── Market Statistics (live from blockchain):
    │    │ Asset price, Total Supply, Total Borrow,
    │    │ Utilization Rate, Supply APY, Borrow APY
    │
    ├── User Position Cards (flip animation):
    │    ├── My Supply (USDC deposited)
    │    ├── My Borrow (USDC borrowed)
    │    ├── My Collateral (WBTC/WXRP/WETH locked)
    │    └── Health Factor (progress bar with color coding)
    │
    └── 7 Action Modals:
         │
         ├── Deposit USDC
         │    │ Enter amount → preview (shares, APY)
         │    │ Approve USDC → Deposit
         │    │ Validations: supply cap, min deposit
         │
         ├── Withdraw USDC
         │    │ Enter amount → preview (shares burned)
         │    │ Max withdraw calculation
         │    │ Error handling: PAUSED, INSUF_SHARES, INSUF_CASH
         │
         ├── Borrow USDC
         │    │ Enter amount → preview (health factor change, daily interest)
         │    │ Max borrow = f(collateral, LTV)
         │    │ Liquidation warning at low HF
         │
         ├── Repay USDC
         │    │ 25% / 50% / Max quick buttons
         │    │ Preview: HF improvement, interest saved
         │    │ Approve USDC → Repay
         │
         ├── Add Collateral (WBTC/WXRP/WETH)
         │    │ Enter amount → preview (new borrow limit)
         │    │ Approve token → Add collateral
         │
         ├── Remove Collateral
         │    │ Enter amount → preview (HF change)
         │    │ Max remove = f(outstanding borrow, HF threshold)
         │    │ Safety status indicator
         │
         └── Liquidation Dashboard
              │ Paginated list of liquidatable positions
              │ Select borrower → see debt/collateral details
              │ Enter repay amount → preview (WBTC seized, profit)
              └── Approve USDC → Liquidate

Pool Contracts (Polygon):
  - WBTC/USDC: contract address in LENDING_POOLS config
  - WXRP/USDC: contract address in LENDING_POOLS config  
  - WETH/USDC: contract address in LENDING_POOLS config
```

### Key Health Factor Logic
| HF Range | Status | Color |
|----------|--------|-------|
| ≥ 1.5 | Safe | Green |
| 1.0–1.5 | At Risk | Orange |
| < 1.0 | Liquidatable | Red |

---

## 17. User Flow 16 — My Referrals

### Status: ✅ FULLY IMPLEMENTED

### Flow
```
My Referrals (/dashboard/my-referrals)
    │
    │ referralsStore.getReferralsByDate(startDate, endDate)
    │
    ├── Header:
    │    │ Total Referrals, Total Rewards (PHL), Monthly change %
    │    └── Fetches previous month data for comparison
    │
    ├── L1 Referrals List (direct referrals)
    │    │ Each entry: name, date, PHL reward amount, Claim button
    │    └── Click Claim → ModalClaimReward
    │
    ├── L2 Referrals List (indirect referrals)
    │    │ Same structure, secondary rewards
    │    └── Click Claim → ModalClaimReward
    │
    └── "Claim All Rewards" button → opens claim modal
         ↓
    ModalClaimReward
         │ Select cause to donate % of reward
         │ Paginated causes list from API
         │ useRewardsStore.claimReferralReward()
         └── Success
```

### API Calls
- `GET /referrals/by-date` — list referrals with rewards
- `GET /referrals/summary` — totals
- `POST /rewards/referral-claim` — claim reward

---

## 18. User Flow 17 — My PRO-Rewards

### Status: ✅ FULLY IMPLEMENTED (API + Blockchain)

### Flow
```
My PRO-Rewards (/dashboard/my-pro-rewards)
    │
    ├── Top Panel: 3 summary cards
    │    ├── Total PRO-Rewards (from rewardsStore)
    │    ├── Claimable Rewards (from blockchain — Rewards contract)
    │    └── Referral Rewards (from referralsStore)
    │
    ├── Mid Panel: 4-column layout
    │    ├── Social Rewards Checklist
    │    ├── Time Spent bar chart
    │    ├── Post Engagement bar chart
    │    └── Referrals mini-card
    │
    └── Bottom Panel: Rewards Counter Table
         │ Aggregates: Post rewards, Referral rewards, Social task rewards,
         │   Cause rewards, Giveback rewards
         │ Each shows count + PHL reward amount
         │
         └── "Claim All" button
              │ useRewards.getAllClaimableRequests() → from blockchain
              │ useRewards.calculateClaimAllFee() → gas estimate
              │ useRewards.claimAll() → on-chain tx
              │ Handles insufficient funds error
              └── WaitForConfirmation

Rewards Contract: 0x3E4144Fc80d18AC54B5A6e080005964896730F8B (Polygon)
```

### Blockchain Methods
- `getAllClaimableRequests(address)` — pending rewards from contract
- `claimAll(address)` — batch claim all pending rewards
- `calculateClaimAllFee()` — estimate gas cost

---

## 19. User Flow 18 — My Causes

### Status: ✅ FULLY IMPLEMENTED

### Flow
```
My Causes (/dashboard/my-causes)
    │
    │ Parallel fetches:
    │   causesStore.fetchCausesSummary()
    │   causesStore.fetchCausesActive()
    │   causesStore.fetchCausesComplete()
    │
    ├── Summary Cards:
    │    ├── Total Causes Created
    │    └── Total Raised (USD)
    │
    ├── Active Causes List
    │    │ Each: image, title, progress bar (raised/goal), navigate
    │    └── Click → /dashboard/my-causes/:causeId
    │
    └── Completed Causes List
         │ Same structure
         └── Click → /dashboard/my-causes/:causeId

Cause Detail (/dashboard/my-causes/:causeId)
    │ Fetches cause by ID from API
    │ Shows: image, description, donor list
    └── Edit icon → /dashboard/my-causes/:causeId/edit

Cause Edit (/dashboard/my-causes/:causeId/edit)
    │ Fetches cause + givers
    │ Editable fields: description, goal, end date, wallet address
    │ Toggle edit mode → modify → "Update"
    │ causesStore.updateCauseById()
    │ ⚠️ "Delete" button exists but has no handler
    └── Cancel → back
```

### API Calls
- `GET /causes/summary` — totals
- `GET /causes/active` — active causes list
- `GET /causes/completed` — completed causes list
- `GET /causes/:id` — cause detail
- `PUT /causes/:id` — update cause
- `GET /causes/:id/givers` — donor list

---

## 20. User Flow 19 — My Give Back

### Status: ✅ FULLY IMPLEMENTED

### Flow
```
My Give Back (/dashboard/my-give-back)
    │
    │ Parallel fetches:
    │   givebackStore.fetchGivebackSummary()
    │   givebackStore.fetchGivebackCategories()
    │   givebackStore.fetchGivebackCauses()
    │   givebackStore.fetchGivebackBadges()
    │   causesStore.fetchCausesSummary()
    │
    ├── Summary Card: Total Given Back (USD)
    │
    ├── Causes List (reuses CausesList component)
    │
    ├── Badges Section
    │    └── ⚠️ PLACEHOLDER — images at opacity 30%, not functional
    │
    └── Top Categories (chips/tags)

Give Back Detail → reuses MyCausesDetailView
Give Back Edit → reuses MyCausesEditView
```

### API Calls
- `GET /giveback/summary`
- `GET /giveback/causes`
- `GET /giveback/badges`
- `GET /giveback/categories`

---

## 21. User Flow 20 — My Followers

### Status: ⚠️ PARTIALLY IMPLEMENTED (API real, composable mocked)

### Flow
```
My Followers (/dashboard/my-followers)
    │
    │ useFollowersStore.fetchGetFollowers(page, size)
    │ useReferralsStore.fetchReferralsSummary()
    │
    ├── Header Cards:
    │    ├── Total Followers (from API)
    │    ├── Followers by Referrals (from API)
    │    └── Total Following (from API)
    │
    ├── Action Menu:
    │    ├── Block User → useUserProfile.blockUser()
    │    └── Export Selected → CSV via papaparse
    │
    ├── Paginated Follower Table
    │    │ Each row: avatar, name, join date, status, action menu
    │    │ Checkbox selection (multi-select)
    │    │ Per-row popover: Follow/Unfollow, Block, Export
    │    └── Pagination controls
    │
    └── "Export All Users" button → CSV download
```

### ⚠️ Note on `useFollower.ts` composable
The composable file `src/composables/useFollower.ts` returns **100% hardcoded mock data** (24 identical "Jerry Lopez" entries). However, the actual `MyFollowersView.vue` and `MyPaginationFollower.vue` components use `useFollowersStore` (the Pinia store) which makes **real API calls**. The composable appears to be legacy/unused.

### API Calls (Real — via followersStore)
- `GET /user/followers` — paginated follower list
- `POST /user/follow` / `POST /user/unfollow` — follow/unfollow
- `POST /user/block` — block user

---

## 22. User Flow 21 — My Posts

### Status: ✅ FULLY IMPLEMENTED

### Flow
```
My Posts (/dashboard/my-posts)
    │
    │ postStore.fetchAllPosts() + postStore.fetchSummaryPost()
    │
    ├── Header Cards:
    │    ├── Total Posts
    │    ├── Music Tracks
    │    └── Live Streaming
    │
    ├── Action Menu:
    │    ├── Delete Selected → postStore.fetchDeletePost()
    │    ├── View Selected → navigate to /dashboard/my-posts/:id
    │    └── Edit Selected → (same navigation)
    │
    ├── "Create Post" button → IonModal
    │    ↓
    │    CreatePost Component
    │    │ Text content, media upload (image/video)
    │    │ Country selection, subscriber-only toggle
    │    │ Tag users
    │    │ Media upload → S3 pre-signed URLs
    │    │ postStore.fetchCreatePost()
    │    └── Close modal
    │
    └── Paginated Posts Table
         │ Each: thumbnail, title, date, engagement stats
         │ Checkbox selection
         └── Click → /dashboard/my-posts/:id

Current Post Detail (/dashboard/my-posts/:id)
    │ Fetches post detail + 6 metrics APIs in parallel:
    │   Activity, Engagement, Impressions, Interactions, Insights, Lives
    │ Shows: post content, media gallery, all metrics charts
    │ Edit mode: modify content, upload/delete media
    │ Delete: with confirmation
    └── Navigation back
```

### API Calls
- `GET /posts` — paginated list
- `GET /posts/summary` — totals
- `POST /posts` — create post
- `GET /posts/:id` — detail
- `PUT /posts/:id` — update
- `DELETE /posts/:id` — delete
- `POST /posts/publish` — publish draft
- S3 pre-signed URL uploads for media

---

## 23. User Flow 22 — My Airdrops (Campaigns)

### Status: ✅ FULLY IMPLEMENTED (API + Blockchain)

### Flow
```
My Airdrops (/dashboard/my-airdrops)
    │
    │ campaignsStore.fetchAllCampaigns()
    │
    ├── Header Cards:
    │    ├── "Create New" → /dashboard/my-airdrops/create
    │    ├── Total Created
    │    └── Total Given Away
    │
    ├── Action Menu:
    │    ├── View → /dashboard/my-airdrops/:id
    │    ├── Pause → useCampaign.pauseCampaign() (blockchain)
    │    ├── Unpause → useCampaign.unpauseCampaign() (blockchain)
    │    └── Delete → useCampaign.deleteCampaign() (blockchain)
    │
    └── Paginated Campaigns Table
         │ Each: name, status, QR code, goal type, dates
         │ Fetches on-chain campaign state
         └── Click → /dashboard/my-airdrops/:id

Create Airdrop (/dashboard/my-airdrops/create)
    │
    ├── Left: Campaign Form
    │    │ Upload image (S3)
    │    │ Campaign name, description
    │    │ Goal type: Follow My Account / Repost My Cause / New Referral
    │    │ Start/End dates
    │    │ Total PHL amount + per-winner amount
    │    │ Max participants
    │    │
    │    │ 1. Approve PHL tokens → ERC20 approve tx
    │    │ 2. Propose campaign → useCampaign.proposeCampaign()
    │    │    → On-chain tx to Campaigns contract
    │    │    → UUID → BigInt conversion for blockchain
    │    └── Wait for confirmation
    │
    └── Right: QR Code Generator
         │ Auto-generates after campaign creation
         └── Copy link / download QR

Current Airdrop (/dashboard/my-airdrops/:id)
    │ View/Edit mode toggle
    │ Shows: image, name, description, amounts, dates, QR
    │ Edit: update end date, amounts (requires on-chain tx)
    │ Lists: rewarded participants with follow/unfollow
    └── ⚠️ Some amount-update logic is commented out pending refactoring

Campaigns Contract: 0xea5838b57DEd6FcDA656d5Aa0F88C31B7b477840 (Polygon)
```

### Campaign Smart Contract Methods
- `proposeCampaign(...)` — create new campaign
- `pauseCampaign(id)` — pause
- `unpauseCampaign(id)` — resume
- `deleteCampaign(id)` — remove
- `getCampaign(id)` — read campaign data
- `addToWhitelist(id, address)` — whitelist participant
- `claimFromCampaign(id)` — participant claims tokens

---

## 24. User Flow 23 — Campaign Goal Pages (Public)

### Status: ✅ FULLY IMPLEMENTED (Blockchain)

These are **public-facing pages** (no auth required) that campaign participants visit via QR code / shared link.

### Flow — Follow My Account
```
/goal/follow-my/:id
    │ Fetches campaign from blockchain
    │ Resolves creator UUID → user profile (API)
    │ Shows: creator avatar, name, Follow + Claim buttons
    │
    ├── "Follow" → useCampaign.addToWhitelist()
    │    │ On-chain tx to whitelist participant
    │    └── Wait for confirmation
    │
    └── "Claim" → useCampaign.claimFromCampaign()
         │ Calculates gas fee
         │ On-chain tx to claim PHL tokens
         └── Wait for confirmation
```

### Flow — Repost My Cause
```
/goal/my-cause/:id
    │ Fetches campaign + cause data
    │ Shows: cause media, title, Repost + Claim buttons
    │
    ├── "Repost" → whitelist via repost action
    └── "Claim" → on-chain claim
```

### Flow — New Referral
```
/goal/new-referral/:id
    │ Fetches campaign + inviter info
    │ Shows: inviter name, referral code, app download links
    │
    ├── Copy referral code
    ├── "Check" → verify whitelist eligibility
    └── "Claim" → on-chain claim
```

---

## 25. User Flow 24 — Account Settings

### Status: ⚠️ MIXED (some fully implemented, some placeholder)

### Flow
```
Account Settings (/dashboard/account-settings)
    │
    ├── Left Sidebar: 10 navigation items
    │
    ├── Edit Account (/dashboard/account-settings/edit)
    │    │ ✅ FULLY IMPLEMENTED
    │    │ View/Edit profile: avatar, name, gender, country, DOB, bio, links
    │    │ Toggle edit mode → modify → useUserProfile.updateUserProfile()
    │    │ Account type buttons (Creator/Business) — ⚠️ no handlers
    │    └── Toast feedback on success/error
    │
    ├── Language (/dashboard/account-settings/language)
    │    │ ⚠️ UI-ONLY — NOT FUNCTIONAL
    │    │ Shows: English, Spanish, Portuguese, Hindi, Arabic, Urdu
    │    │ Selection tracked locally but "Update" button has no handler
    │    └── No i18n integration
    │
    ├── Permissions (/dashboard/account-settings/permissions)
    │    │ ❌ PLACEHOLDER — displays "Permissions" text only
    │    └── No functionality
    │
    ├── My Interests (/dashboard/account-settings/interests)
    │    │ ✅ FULLY IMPLEMENTED
    │    │ Fetches all interests + user's interests from API
    │    │ Toggle selection → useUserProfile.updateMyInterests()
    │    └── Toast feedback
    │
    ├── Manage Subscriptions (/dashboard/account-settings/subscription)
    │    │ ✅ FULLY IMPLEMENTED (Blockchain)
    │    │ Shows: active/inactive status, next due date
    │    │ Cancel subscription → on-chain tx
    │    │ Renew subscription → on-chain tx
    │    │ Change payment token (PHL/USDT) → approve + save
    │    └── Uses PaymentsProcessor contract
    │
    ├── Subscription Wallets (/dashboard/account-settings/wallets)
    │    │ ✅ FULLY IMPLEMENTED (Blockchain)
    │    │ Shows: primary wallet + all linked wallets
    │    │ Switch primary wallet → on-chain fee + tx
    │    │ Add new wallet → checks collision → links on-chain
    │    └── Uses PaymentsProcessor contract
    │
    ├── Change Password (/dashboard/account-settings/password)
    │    │ ⚠️ UI-ONLY — NOT FUNCTIONAL
    │    │ Shows: old, new, confirm password inputs
    │    │ Password visibility toggle works
    │    └── "Update Password" button has NO click handler
    │
    ├── Account Privacy (/dashboard/account-settings/privacy)
    │    │ ❌ PLACEHOLDER — displays "Account Privacy" text only
    │    └── No functionality
    │
    ├── About (/dashboard/account-settings/about)
    │    │ ❌ PLACEHOLDER — displays "About" text only
    │    └── No functionality
    │
    └── Help & Support (/dashboard/account-settings/help)
         │ ❌ PLACEHOLDER — displays "Help & Support" text only
         └── No functionality

Additional actions on the settings page:
    ├── "Log Out" button → useAuthStore.logout() (clears tokens + wallet)
    └── "Delete Account" button → useUserProfile.deleteAccount() + logout
```

---

## 26. User Flow 25 — Transaction History

### Status: ⚠️ MIXED (some real, some mocked)

### Available History Views
```
Transaction History (/dashboard/transaction-history)
    │ TransactionHistoryPagination component
    │ ✅ REAL — queries blockchain via HdWallet service
    │ Shows: date, type, amount, tx hash → links to explorer
    └── Paginated

My Wallet Side Panel → 5 History Modals:
    │
    ├── All Transactions History
    │    │ ✅ REAL — same blockchain query as above
    │    └── Click tx → opens Polygonscan
    │
    ├── Staking History Modal
    │    │ ✅ REAL — queries Polygon logs for StablePool events
    │    │ Events: Deposited, Redeemed, RewardsClaimed
    │    │ Chunked log fetching with retry/backoff
    │    │ Timeframe filter (7d/30d/90d/1y/All)
    │    └── Paginated
    │
    ├── Swap History Modal
    │    │ ✅ REAL — queries Polygon logs for PurchaseExecuted events
    │    │ Filters by tx sender address
    │    │ Timeframe filter
    │    └── Paginated
    │
    ├── Lending Transaction History
    │    │ ✅ REAL — queries Polygon lending contract logs
    │    │ Events: Deposited, Withdrawn, Borrowed, Repaid,
    │    │   CollateralAdded, CollateralRemoved, Liquidated
    │    │ Pool filter + timeframe filter
    │    │ Concurrency-controlled fetching (2 events at a time)
    │    └── Paginated
    │
    └── Locked Positions Modal
         │ ✅ REAL — fetches from StablePool contract
         │ Shows all locked staking positions with dates
         └── Sorted by unlock date
```

### ⚠️ Mocked Transaction Components (Lending sub-views)
| Component | Status |
|-----------|--------|
| `LendingTransactionHistory.vue` (in lending folder — NOT the modal) | ❌ MOCK — `generateMockData()` creates 50 random transactions |
| `TransactionHistoryPagination.vue` (in lending folder — NOT the full view) | ❌ MOCK — `generateMockData()` creates 100 random transactions |

These are **different** from the real modal versions — they appear to be early stubs within the lending protocol sub-view that were never replaced.

---

## 27. User Flow 26 — PhilAds (Advertising)

### Status: ❌ COMMENTED OUT / PLACEHOLDER

The entire PhilAds feature is **commented out** in the router. The route definition exists but is disabled:

```typescript
// {
//   path: 'philads',
//   name: 'PhilAds',
//   component: () => import('../views/dashboard/PhilAdsView.vue'),
//   children: [ ... ]  // Stats, Announcements, Targets, Wallet, Notifications, Documents, Settings
// }
```

### What exists (UI-only):
- `PhilAdsView.vue` — layout wrapper with sidebar
- `PhilStats.vue` — header cards (Total Spent, Health, Reach) — **UI-only**
- `PhilAdsHealthDoughnut.vue` — health gauge chart — **UI-only**
- `ButtonGroupPhilAds.vue` — 4 nav buttons (All Campaigns, Audiences, Billing, Reports) — **static**
- `ActiveCompaigns.vue` — campaign list — **mock/empty**
- 6 placeholder sub-views: Announcements, Targets, Wallet, Notifications, Documents, Settings — all show **just text labels**

---

## 28. Implementation Status Summary

### By Feature — Fully Functional
| Feature | Status | Data Source |
|---------|--------|------------|
| User Registration (Email) | ✅ Full | REST API |
| User Registration (Google/Apple) | ✅ Full | OAuth + REST API |
| MFA Setup & Verification | ✅ Full | REST API (TOTP) |
| Profile Onboarding | ✅ Full | REST API |
| Interest Selection | ✅ Full | REST API |
| Login (Email + MFA) | ✅ Full | REST API |
| Login (Google/Apple) | ✅ Full | OAuth + REST API |
| Password Reset | ✅ Full | REST API |
| Native Wallet Creation | ✅ Full | Local DB (SQLite) |
| Wallet Import (Seed) | ✅ Full | Local DB (SQLite) |
| MetaMask Connection | ✅ Full | MetaMask SDK |
| Subscription Payment | ✅ Full | Polygon Contract |
| Dashboard Home Metrics | ✅ Full | REST API (7 endpoints) |
| Asset Balances (Multi-chain) | ✅ Full | Blockchain RPC |
| Send/Transfer Crypto | ✅ Full | Blockchain RPC |
| Receive Crypto (QR) | ✅ Full | Local |
| Swap USDT→PHL | ✅ Full | Polygon Contract |
| Buy PHL (Fiat) | ✅ Full | Onramper Widget |
| StablePool DeFi (Staking) | ✅ Full | Polygon Contract |
| Lending Protocol (Full) | ✅ Full | Polygon Contract |
| My Referrals + Claims | ✅ Full | REST API + Blockchain |
| My PRO-Rewards + Claims | ✅ Full | REST API + Blockchain |
| My Causes (CRUD) | ✅ Full | REST API |
| My Give Back | ✅ Full | REST API |
| My Posts (CRUD + Media) | ✅ Full | REST API + S3 |
| My Airdrops (CRUD + On-chain) | ✅ Full | REST API + Blockchain |
| Campaign Goal Pages (Public) | ✅ Full | Blockchain |
| Edit Profile | ✅ Full | REST API |
| Manage Subscriptions | ✅ Full | Polygon Contract |
| Subscription Wallets | ✅ Full | Polygon Contract |
| Update Interests | ✅ Full | REST API |
| My Followers | ✅ Full | REST API |
| Delete Account | ✅ Full | REST API |
| Logout | ✅ Full | REST API + localStorage |
| Transaction History (All) | ✅ Full | Blockchain RPC |
| Staking History | ✅ Full | Polygon Logs |
| Swap History | ✅ Full | Polygon Logs |
| Lending History | ✅ Full | Polygon Logs |

### By Feature — UI Only / Mocked / Placeholder
| Feature | Status | Details |
|---------|--------|---------|
| Language Settings | ⚠️ UI-Only | Radio buttons render, selection tracks locally, but no i18n, no API call, no persistence |
| Change Password | ⚠️ UI-Only | Form renders, visibility toggle works, but "Update" button has no click handler |
| Permissions Settings | ❌ Placeholder | Shows "Permissions" text only |
| Account Privacy | ❌ Placeholder | Shows "Account Privacy" text only |
| About | ❌ Placeholder | Shows "About" text only |
| Help & Support | ❌ Placeholder | Shows "Help & Support" text only |
| PhilAds (Advertising) | ❌ Commented Out | Router disabled, 14 components exist but are all UI-only/mock/placeholder |
| Dashboard "Download Report" | ⚠️ UI-Only | Button visible, no click handler |
| Give Back Badges | ⚠️ Placeholder | Images at 30% opacity, not interactive |
| Cause Delete Button | ⚠️ UI-Only | Button in edit view, no handler attached |
| Account Type Switch (Creator/Business) | ⚠️ UI-Only | Buttons in edit profile, no handlers |
| Lending Inline History (non-modal) | ❌ Mocked | `generateMockData()` with random data |
| Notifications (Sidebar) | ❌ Disabled | Sidebar link exists but is greyed out |
| Reports (Sidebar) | ❌ Disabled | Sidebar link exists but is greyed out |
| `useFollower.ts` composable | ❌ Mocked | Returns 24 hardcoded "Jerry Lopez" entries (appears unused — stores use real API) |
| Airdrop Amount Update | ⚠️ Partial | Some update logic commented out pending refactoring |
| Staked Balance on BalanceCard | ⚠️ Hardcoded | Always shows 0 |
| `BalancesView.vue` (old wallet) | ⚠️ Partial | Route commented out, total USD hardcoded ($124.329), action buttons not wired |

### Smart Contracts Used (Polygon Mainnet)
| Contract | Address | Purpose |
|----------|---------|---------|
| PaymentsProcessor | `0xB1d569605C6E9B5D0cb1F6921D75DBC5Bbeb9FBB` | Subscription payments |
| Campaigns | `0xea5838b57DEd6FcDA656d5Aa0F88C31B7b477840` | Airdrop campaigns |
| Rewards | `0x3E4144Fc80d18AC54B5A6e080005964896730F8B` | PRO-Rewards claims |
| StablePool | `0x95A8399f7cF8d4007D491b2F4822DA89b19c0de8` | DeFi staking |
| Lending Pools | Configured per pool (WBTC/WXRP/WETH) | Lending protocol |
| Swap | Configured in Polygon wallet | USDT ↔ PHL swap |

---

## 29. Known Issues & Technical Debt

### Security
1. **Hardcoded secret key** in `stores/useNevoStore.ts` — `GG*I3Q%Qt^W*!)ddwtgD` used for client-side request signing. This should be server-side.
2. **API keys in source** — Etherscan API keys, QuikNode URLs, Sentry DSN are all in committed source code.
3. **Auth interceptor disabled** — Token refresh interceptor in `api/config.ts` is fully commented out.

### Code Quality
4. **Duplicate file** — `database/params/dataSourceParams.ts` is identical to `database/params/index.ts`.
5. **Filename typo** — `types/stbalepool.d.ts` should be `stablepool.d.ts`.
6. **Type name typo** — `BuildTransactionOpt` is typed as `BuildTransacionOpt`.
7. **Duplicate type declarations** in `types/client.d.ts` — `CauseDetail`, `PostsDetail`, `UserProfile`, `FollowUnfollowRequest` defined twice.
8. **Missing referenced types** — `SocialMetric`, `SocialChannel`, `TargetAudience`, `ContentFilter`, `ContentType` used but never defined.
9. **Invalid TS in interfaces** — Default values in interface method signatures (not valid TypeScript).
10. **Unused counter store** — `stores/counter.ts` is the default Pinia scaffold, appears unused.
11. **Empty store file** — `stores/useModalClaim.ts` is completely empty.
12. **Legacy API files** — `api/myPosts.ts` is entirely commented out (replaced by SDK).

### UX Gaps
13. **No error boundaries** — blockchain failures may leave modals stuck.
14. **No offline handling** — no service worker or offline detection for the PWA manifest.
15. **Worker file** (`worker.ts`) exists but purpose is unclear — needs investigation.
16. **Breadcrumb navigation** — uses raw `ionRouter.back()` which may not navigate to expected parent route.
