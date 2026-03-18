# WalletConnect Integration — Philcoin Wallet ↔ Phil-DeFi Web App

> **Goal:** Allow users to connect the Philcoin mobile wallet (with a built-in HD wallet) to the phil-defi web application by scanning a WalletConnect QR code.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Current Architecture Analysis](#2-current-architecture-analysis)
3. [How WalletConnect Works](#3-how-walletconnect-works)
4. [Web App (phil-defi) — Changes Required](#4-web-app-phil-defi--changes-required)
5. [Mobile App (Philcoin Wallet) — Full Implementation Guide](#5-mobile-app-philcoin-wallet--full-implementation-guide)
6. [Data Flow Diagrams](#6-data-flow-diagrams)
7. [JSON-RPC Methods to Support](#7-json-rpc-methods-to-support)
8. [Security Considerations](#8-security-considerations)
9. [Testing Checklist](#9-testing-checklist)

---

## 1. Overview

### Current State
- The web app has two connection types: **MetaMask** (injected browser provider) and **NativeProvider** (local HD wallet stored in SQLite, encrypted, signing done in-browser).
- The `WalletConnector.vue` modal shows a "Philsocial" button **but it has no `@click` handler** — it does nothing.
- The right side of the modal already shows a `wallet_connect.png` image but it's decorative only.

### Target State
- User opens phil-defi web app → clicks "Philsocial" → a WalletConnect QR code appears.
- User opens the Philcoin mobile app → scans the QR code.
- A secure P2P session is established via the WalletConnect relay.
- The web app can request accounts, send transactions for signing, etc.
- The mobile app signs transactions locally with its HD wallet and returns the result.
- **No private keys ever leave the mobile device.**

### Why WalletConnect v2
- Industry standard for wallet ↔ dApp communication.
- Works cross-platform (mobile native ↔ web).
- Encrypted communication via relay server.
- Session persistence — user doesn't need to re-scan every time.
- Multi-chain support (Polygon, Ethereum, BSC, Solana in the future).

---

## 2. Current Architecture Analysis

### Provider System

```
HdWallet extends Blockchains
    │
    └── Blockchains (src/services/connectors/metamask/index.ts)
          │
          ├── mappedBlockchain(name) → returns blockchain-specific methods
          │     ethereum.ts / polygon.ts / bsc.ts / solana.ts / etc.
          │     (balance fetching, tx building for NativeProvider, RPC calls, fee estimation)
          │
          ├── mappedProvider(name) → returns provider-specific methods
          │     'Metamask'       → metamask.ts  (getAccounts, buildTransaction, sendRawTransaction via window.ethereum)
          │     'NativeProvider'  → {}           (empty object — blockchain methods handle everything)
          │
          └── get blockchain() → { ...parent, ...provider }
                Provider methods OVERRIDE blockchain methods via object spread.
```

### Key Provider Methods (metamask.ts interface)

The WalletConnect provider must implement **the same methods** that `metamask.ts` implements:

| Method | Purpose | MetaMask Implementation |
|--------|---------|------------------------|
| `getAccounts(dflt)` | Get connected wallet address(es) | `eth_requestAccounts` via `window.ethereum` |
| `buildTransaction(data, chainId)` | Build a JSON tx object for signing | Constructs `{from, to, value, gas, gasPrice, data}` |
| `sendRawTransaction(rawTx, blockchain, nodes, chainId)` | Send transaction to network | `eth_sendTransaction` via `window.ethereum` |
| `onChangeAccounts(callback)` | Listen for account changes | `provider.on('accountsChanged', ...)` |
| `checkNetwork(chainId)` | Verify current chain | `eth_chainId` |
| `switchNetwork(chainId, name, rpcUrls)` | Request chain switch | `wallet_switchEthereumChain` |

### WalletStore Transaction Flow

```
1. composable (useLending, usePayments, etc.) calls walletStore.buildTransaction(data)
2. WalletStore reads defaultProvider() from localStorage  →  'Metamask' | 'NativeProvider'
3. WalletStore creates HdWallet(blockchain, provider)
4. HdWallet.buildTransaction(data, chainId) dispatches to provider's buildTransaction

5. composable calls walletStore.sendRawTransaction(rawTx, blockchain)
6. WalletStore reads defaultProvider() again
7. If 'Metamask'       → sends JSON object via eth_sendTransaction
8. If 'NativeProvider'  → signs hex tx locally with encrypted wallet → sends signed hex via RPC
```

---

## 3. How WalletConnect Works

```
┌──────────────────┐         WalletConnect Relay         ┌───────────────────┐
│                  │        (cloud.walletconnect.com)      │                   │
│   phil-defi      │                                       │  Philcoin Mobile  │
│   (Web dApp)     │◄─────── Encrypted P2P Messages ──────►│  (Wallet)         │
│                  │                                       │                   │
│  Uses:           │    1. dApp generates pairing URI       │  Uses:            │
│  @walletconnect/ │    2. Displays as QR code              │  @walletconnect/  │
│  ethereum-       │    3. Wallet scans QR                  │  web3wallet       │
│  provider        │    4. Session established              │                   │
│                  │    5. dApp sends JSON-RPC requests      │  Signs locally    │
│                  │    6. Wallet responds with results      │  with HD wallet   │
└──────────────────┘                                       └───────────────────┘
```

### Key Concepts
- **Project ID** — free, obtained from [cloud.walletconnect.com](https://cloud.walletconnect.com)
- **Pairing URI** — `wc:abc123@2?relay-protocol=irn&symKey=xyz` — encoded in QR code
- **Session** — persistent connection; survives page reloads; has an expiry
- **Namespaces** — chains and methods the dApp requires (e.g., `eip155:137` for Polygon)
- **JSON-RPC** — standard Ethereum methods sent through the relay

---

## 4. Web App (phil-defi) — Changes Required

### 4.1. Install Package

```bash
pnpm add @walletconnect/ethereum-provider
```

### 4.2. Create WalletConnect Provider

Create `src/services/connectors/walletconnect/walletconnect.ts`:

```typescript
import { EthereumProvider } from '@walletconnect/ethereum-provider'

const PROJECT_ID = import.meta.env.VITE_WALLETCONNECT_PROJECT_ID

let wcProvider: Awaited<ReturnType<typeof EthereumProvider.init>> | null = null

const WalletConnectProviderMethods = {

  /**
   * Initialize and return the WalletConnect EthereumProvider.
   * Shows the QR code modal automatically on first call.
   */
  async getWCProvider(): Promise<typeof wcProvider> {
    if (wcProvider) return wcProvider

    wcProvider = await EthereumProvider.init({
      projectId: PROJECT_ID,
      chains: [137],                         // Polygon mainnet
      optionalChains: [1, 56],               // Ethereum, BSC (optional)
      showQrModal: true,                     // Built-in QR modal
      metadata: {
        name: 'PhilSocial Pro',
        description: 'PhilSocial DeFi Platform',
        url: window.location.origin,
        icons: [window.location.origin + '/PHL.svg'],
      },
    })

    return wcProvider
  },

  /**
   * Get connected accounts. Triggers QR modal if no session exists.
   */
  async getAccounts(dflt: string[]): Promise<string[]> {
    const provider = await this.getWCProvider()
    if (!provider) throw new Error('WalletConnect provider not available')

    // If no active session, this will show the QR modal
    if (!provider.session) {
      await provider.connect()
    }

    const accounts = provider.accounts
    if (accounts && accounts.length > 0) return accounts

    // Fallback: request accounts explicitly
    const requested = await provider.request({ method: 'eth_requestAccounts' }) as string[]
    return requested
  },

  /**
   * Build a transaction object (same format as MetaMask provider).
   * WalletConnect uses eth_sendTransaction — the wallet signs it.
   */
  async buildTransaction(data: BuildTransacionOpt, chainId = 0): Promise<string | null> {
    try {
      const dataRaw = data.rawtx
      if (dataRaw == null) return null

      const provider = await this.getWCProvider()
      if (!provider) return null

      // Check and switch network if needed
      const currentChainId = provider.chainId
      if (chainId > 0 && currentChainId !== chainId) {
        try {
          await provider.request({
            method: 'wallet_switchEthereumChain',
            params: [{ chainId: `0x${chainId.toString(16)}` }],
          })
        } catch (e) {
          console.warn('Failed to switch chain via WalletConnect:', e)
        }
      }

      const transaction: any = {
        from: data.from[0],
        to: data.to[0],
        value: `0x${BigInt(data.amount).toString(16)}`,
        gas: `0x${data.feerate.gasLimit.toString(16)}`,
        gasPrice: `0x${data.feerate.gasPrice.toString(16)}`,
      }
      if (dataRaw.length > 0) {
        transaction.data = dataRaw
      }
      return JSON.stringify(transaction)
    } catch (error) {
      console.error('Error building WalletConnect transaction:', error)
      return null
    }
  },

  /**
   * Send transaction via WalletConnect.
   * The mobile wallet receives the request, shows it to the user,
   * signs it locally, broadcasts it, and returns the txHash.
   */
  async sendRawTransaction(
    rawTx: string,
    blockchain: string,
    nodes: Nodes[],
    chainId = 0,
  ): Promise<string> {
    const provider = await this.getWCProvider()
    if (!provider) throw new Error('WalletConnect provider not available')

    // Expect JSON tx object (same as MetaMask path)
    const trimmed = (rawTx || '').trim()
    if (trimmed.startsWith('0x')) {
      throw new Error('Raw hex not supported for WalletConnect. Expected JSON tx object.')
    }

    let txObject: any
    try {
      txObject = JSON.parse(rawTx)
    } catch (e) {
      throw new Error('Invalid transaction payload: expected JSON stringified object')
    }

    // The mobile wallet will prompt the user, sign, and broadcast
    const txHash = await provider.request({
      method: 'eth_sendTransaction',
      params: [txObject],
    })

    return txHash as string
  },

  /**
   * Listen for account changes in the WalletConnect session.
   */
  async onChangeAccounts(callback: (accounts: string[]) => void): Promise<void> {
    const provider = await this.getWCProvider()
    if (!provider) return

    provider.on('accountsChanged', (accounts: string[]) => {
      callback(accounts)
    })
  },

  /**
   * Check if the wallet is on the expected chain.
   */
  async checkNetwork(chainId: number): Promise<boolean> {
    const provider = await this.getWCProvider()
    if (!provider) return false
    return provider.chainId === chainId
  },

  /**
   * Request the wallet to switch chains.
   */
  async switchNetwork(chainId: number, chainName: string, rpcUrls: string[]): Promise<void> {
    const provider = await this.getWCProvider()
    if (!provider) return

    try {
      await provider.request({
        method: 'wallet_switchEthereumChain',
        params: [{ chainId: `0x${chainId.toString(16)}` }],
      })
    } catch (switchError: any) {
      if (switchError.code === 4902) {
        await provider.request({
          method: 'wallet_addEthereumChain',
          params: [{
            chainId: `0x${chainId.toString(16)}`,
            chainName,
            rpcUrls,
          }],
        })
      }
    }
  },

  /**
   * Disconnect the WalletConnect session.
   */
  async disconnect(): Promise<void> {
    if (wcProvider) {
      await wcProvider.disconnect()
      wcProvider = null
    }
  },
}

export default WalletConnectProviderMethods
```

### 4.3. Register Provider in Blockchains Class

Update `src/services/connectors/metamask/index.ts`:

```typescript
import WalletConnectProvider from '../walletconnect/walletconnect'

class Blockchains {
  BLOCKCHAIN: string
  Provider: 'Metamask' | 'NativeProvider' | 'WalletConnect'

  // ...

  mappedProvider(provider: string) {
    switch (provider) {
      case 'Metamask':
        return Metamask
      case 'WalletConnect':
        return WalletConnectProvider    // ← NEW
      case 'NativeProvider':
        return {}
      default:
        throw new Error(`Provider ${provider} not supported`)
    }
  }
}
```

### 4.4. Update WalletConnector.vue

Add the click handler to the Philsocial button:

```vue
<!-- Before (no handler) -->
<div class="flex flex-col justify-start gap-1 cursor-pointer hover:opacity-80">
  <Image class="w-10 h-10 m-auto" :src="philsocial" alt="Philsocial" />
  <h3 class="text-sm text-white font-bold">Philsocial</h3>
</div>

<!-- After (with handler) -->
<div
  @click="connectInjectedWallet('WalletConnect')"
  class="flex flex-col justify-start gap-1 cursor-pointer hover:opacity-80"
>
  <Image class="w-10 h-10 m-auto" :src="philsocial" alt="Philsocial" />
  <h3 class="text-sm text-white font-bold">Philsocial</h3>
</div>
```

That's it — `connectInjectedWallet('WalletConnect')` already calls `walletStore.createInjectedWallet(provider, 'polygon')`, which calls `hdWallet(blockchain, provider).getAccounts([], provider)`. The WalletConnect provider's `getAccounts()` will trigger the QR modal.

### 4.5. Update WalletStore `sendRawTransaction`

Add a `WalletConnect` branch (it follows the same path as MetaMask — JSON tx object, not raw hex):

```typescript
// In sendRawTransaction, after the Metamask guard:
if (providerPref === 'WalletConnect' && isHex) {
  throw new Error('WalletConnect path refuses raw hex. Build a JSON tx object.')
}
```

### 4.6. Update WalletStore `canSign`

```typescript
function canSign(): boolean {
  const provider = defaultProvider()
  if (!provider) return false
  if (!connectedAddress.value) return false
  if (provider === 'Metamask') return true
  if (provider === 'WalletConnect') return true   // ← NEW
  if (provider === 'NativeProvider') {
    return Password.value.length > 0
  }
  return false
}
```

### 4.7. Add Environment Variable

Add to `.env`:
```
VITE_WALLETCONNECT_PROJECT_ID=your_project_id_here
```

---

## 5. Mobile App (Philcoin Wallet) — Full Implementation Guide

This is the detailed specification for what needs to be built on the mobile side.

### 5.1. Overview

The mobile app acts as a **WalletConnect Wallet** (not a dApp). It receives JSON-RPC requests from the web app, shows them to the user for approval, signs transactions using the local HD wallet, broadcasts them to the network, and returns results.

### 5.2. Package to Use

```bash
# For React Native / Expo
npm install @walletconnect/web3wallet @walletconnect/core

# For native iOS/Android, use the official WalletConnect Swift/Kotlin SDKs
# Swift: https://github.com/WalletConnect/WalletConnectSwiftV2
# Kotlin: https://github.com/WalletConnect/WalletConnectKotlinV2
```

### 5.3. Core Architecture

```
┌───────────────────────────────────────────────────┐
│                  Philcoin Mobile App                │
├───────────────────────────────────────────────────┤
│                                                     │
│  ┌─────────────┐   ┌──────────────┐                │
│  │ QR Scanner   │──►│ WC Core      │                │
│  │ (Camera)     │   │ (Pairing)    │                │
│  └─────────────┘   └──────┬───────┘                │
│                           │                         │
│                    ┌──────▼───────┐                 │
│                    │ Web3Wallet   │                  │
│                    │ SDK          │                  │
│                    └──────┬───────┘                 │
│                           │                         │
│              ┌────────────┼────────────┐            │
│              ▼            ▼            ▼            │
│    ┌─────────────┐ ┌──────────┐ ┌──────────┐      │
│    │session_      │ │session_  │ │session_  │      │
│    │proposal      │ │request   │ │delete    │      │
│    │Handler       │ │Handler   │ │Handler   │      │
│    └──────┬──────┘ └────┬─────┘ └──────────┘      │
│           │             │                           │
│    ┌──────▼──────┐ ┌────▼─────────────────┐        │
│    │ Approval    │ │ Request Router        │        │
│    │ Screen      │ │                       │        │
│    │ (show user  │ │ eth_sendTransaction   │        │
│    │  chains &   │ │ → Sign + Broadcast    │        │
│    │  methods)   │ │                       │        │
│    └─────────────┘ │ personal_sign         │        │
│                    │ → Sign message         │        │
│                    │                       │        │
│                    │ eth_signTypedData_v4  │        │
│                    │ → EIP-712 sign         │        │
│                    └───────┬───────────────┘        │
│                            │                        │
│                    ┌───────▼───────┐                │
│                    │ HD Wallet     │                │
│                    │ (local keys)  │                │
│                    │               │                │
│                    │ ethers.js /   │                │
│                    │ web3.js       │                │
│                    └───────────────┘                │
└───────────────────────────────────────────────────┘
```

### 5.4. Step-by-Step Implementation

#### Step 1: Initialize Web3Wallet SDK

```typescript
import { Core } from '@walletconnect/core'
import { Web3Wallet } from '@walletconnect/web3wallet'

const PROJECT_ID = 'your_project_id'  // Same project or a separate one

const core = new Core({
  projectId: PROJECT_ID,
})

const web3wallet = await Web3Wallet.init({
  core,
  metadata: {
    name: 'Philcoin Wallet',
    description: 'Philcoin HD Wallet',
    url: 'https://philcoin.com',
    icons: ['https://philcoin.com/icon.png'],
  },
})
```

#### Step 2: QR Code Scanning

When the user taps "Scan QR" in the mobile app:

```typescript
// After scanning QR code, you get a URI like:
// wc:abc123@2?relay-protocol=irn&symKey=xyz789...

async function onQRCodeScanned(uri: string) {
  await web3wallet.pair({ uri })
  // This triggers the 'session_proposal' event
}
```

#### Step 3: Handle Session Proposal

When the web app requests a connection, the mobile app receives a proposal. **You must show a UI to the user asking them to approve.**

```typescript
web3wallet.on('session_proposal', async (proposal) => {
  const { id, params } = proposal

  // params.requiredNamespaces contains what the dApp needs:
  // {
  //   eip155: {
  //     chains: ['eip155:137'],           // Polygon
  //     methods: ['eth_sendTransaction', 'personal_sign', ...],
  //     events: ['accountsChanged', 'chainChanged']
  //   }
  // }

  // ──────────────────────────────────────────────
  // SHOW APPROVAL UI TO USER HERE
  // Display:
  //   - dApp name: "PhilSocial Pro"
  //   - dApp URL: the web app URL
  //   - Requested chains: Polygon (137)
  //   - Requested methods: send transactions, sign messages
  //   - Ask: "Do you want to connect?"
  // ──────────────────────────────────────────────

  // If user approves:
  const userAddress = getCurrentHDWalletAddress()  // Your existing HD wallet logic

  const session = await web3wallet.approveSession({
    id,
    namespaces: {
      eip155: {
        accounts: [
          `eip155:137:${userAddress}`,    // Polygon
          `eip155:1:${userAddress}`,      // Ethereum (optional)
          `eip155:56:${userAddress}`,     // BSC (optional)
        ],
        methods: [
          'eth_sendTransaction',
          'personal_sign',
          'eth_sign',
          'eth_signTypedData',
          'eth_signTypedData_v4',
          'wallet_switchEthereumChain',
          'wallet_addEthereumChain',
        ],
        events: ['accountsChanged', 'chainChanged'],
        chains: ['eip155:137', 'eip155:1', 'eip155:56'],
      },
    },
  })

  // Session is established! The web app now knows the user's address.

  // If user rejects:
  // await web3wallet.rejectSession({
  //   id,
  //   reason: { code: 5000, message: 'User rejected' },
  // })
})
```

#### Step 4: Handle Session Requests (THE CORE LOGIC)

This is where the mobile app processes requests from the web app. **Every request must be shown to the user before executing.**

```typescript
web3wallet.on('session_request', async (event) => {
  const { id, topic, params } = event
  const { request, chainId } = params
  const { method, params: rpcParams } = request

  switch (method) {

    // ═══════════════════════════════════════════════
    // eth_sendTransaction
    // The web app wants to send a transaction.
    // The mobile app must:
    //   1. Parse the transaction
    //   2. Show details to user (to, value, data, gas)
    //   3. If approved: sign with HD wallet, broadcast, return txHash
    //   4. If rejected: return error
    // ═══════════════════════════════════════════════
    case 'eth_sendTransaction': {
      const tx = rpcParams[0]
      // tx = { from, to, value, gas, gasPrice, data }

      // ── SHOW TRANSACTION APPROVAL UI ──
      // Display:
      //   To: tx.to
      //   Value: formatEther(tx.value) POL
      //   Gas: tx.gas
      //   Contract interaction: tx.data ? "Yes" : "No"
      //   Estimated fee: gas * gasPrice
      //
      // If it's a known contract (lending pool, payments, rewards),
      // you can decode tx.data to show human-readable info like:
      //   "Deposit 100 USDC to Lending Pool"

      // If user approves:
      const signedTx = await signTransactionWithHDWallet(tx, chainId)
      const txHash = await broadcastTransaction(signedTx, chainId)

      await web3wallet.respondSessionRequest({
        topic,
        response: {
          id,
          jsonrpc: '2.0',
          result: txHash,
        },
      })

      // If user rejects:
      // await web3wallet.respondSessionRequest({
      //   topic,
      //   response: {
      //     id,
      //     jsonrpc: '2.0',
      //     error: { code: 4001, message: 'User rejected the request' },
      //   },
      // })
      break
    }

    // ═══════════════════════════════════════════════
    // personal_sign
    // Sign a plaintext message (e.g., login verification)
    // ═══════════════════════════════════════════════
    case 'personal_sign': {
      const message = rpcParams[0]  // hex-encoded message
      const address = rpcParams[1]

      // ── SHOW MESSAGE SIGNING UI ──
      // Display the decoded message to the user

      const signature = await signMessageWithHDWallet(message, address)

      await web3wallet.respondSessionRequest({
        topic,
        response: { id, jsonrpc: '2.0', result: signature },
      })
      break
    }

    // ═══════════════════════════════════════════════
    // eth_signTypedData_v4
    // EIP-712 structured data signing (used by some DeFi protocols)
    // ═══════════════════════════════════════════════
    case 'eth_signTypedData':
    case 'eth_signTypedData_v4': {
      const address = rpcParams[0]
      const typedData = JSON.parse(rpcParams[1])

      // ── SHOW TYPED DATA UI ──
      // Display domain, message type, and values

      const signature = await signTypedDataWithHDWallet(address, typedData)

      await web3wallet.respondSessionRequest({
        topic,
        response: { id, jsonrpc: '2.0', result: signature },
      })
      break
    }

    // ═══════════════════════════════════════════════
    // wallet_switchEthereumChain
    // The web app wants to switch to a different network
    // ═══════════════════════════════════════════════
    case 'wallet_switchEthereumChain': {
      const targetChainId = parseInt(rpcParams[0].chainId, 16)

      // Update internal state to the requested chain
      // Emit chainChanged event back to the dApp
      web3wallet.emitSessionEvent({
        topic,
        event: { name: 'chainChanged', data: targetChainId },
        chainId: `eip155:${targetChainId}`,
      })

      await web3wallet.respondSessionRequest({
        topic,
        response: { id, jsonrpc: '2.0', result: null },
      })
      break
    }

    // ═══════════════════════════════════════════════
    // eth_accounts / eth_requestAccounts
    // Return the connected address
    // ═══════════════════════════════════════════════
    case 'eth_accounts':
    case 'eth_requestAccounts': {
      const address = getCurrentHDWalletAddress()
      await web3wallet.respondSessionRequest({
        topic,
        response: { id, jsonrpc: '2.0', result: [address] },
      })
      break
    }

    // ═══════════════════════════════════════════════
    // eth_chainId
    // Return current chain ID
    // ═══════════════════════════════════════════════
    case 'eth_chainId': {
      const currentChain = getCurrentChainId()  // e.g., 137
      await web3wallet.respondSessionRequest({
        topic,
        response: {
          id,
          jsonrpc: '2.0',
          result: `0x${currentChain.toString(16)}`,
        },
      })
      break
    }

    default: {
      // Unsupported method
      await web3wallet.respondSessionRequest({
        topic,
        response: {
          id,
          jsonrpc: '2.0',
          error: { code: -32601, message: `Method not supported: ${method}` },
        },
      })
    }
  }
})
```

#### Step 5: Signing and Broadcasting (Mobile Wallet Core)

These are the functions referenced above. They use your existing HD wallet:

```typescript
import { ethers } from 'ethers'

// ── Get the HD wallet signer ──
async function getHDWalletSigner(chainId: string): Promise<ethers.Wallet> {
  // Use your existing mnemonic/seed phrase stored in the mobile app
  const mnemonic = await getStoredMnemonic()  // Your secure storage
  const hdNode = ethers.HDNodeWallet.fromMnemonic(
    ethers.Mnemonic.fromPhrase(mnemonic),
    "m/44'/60'/0'/0/0"
  )
  const rpcUrl = getRpcUrlForChain(chainId)  // Your RPC node mapping
  const provider = new ethers.JsonRpcProvider(rpcUrl)
  return new ethers.Wallet(hdNode.privateKey, provider)
}

// ── Sign and broadcast a transaction ──
async function signTransactionWithHDWallet(
  tx: { from: string; to: string; value: string; gas: string; gasPrice: string; data?: string },
  chainId: string
): Promise<string> {
  const numericChainId = parseInt(chainId.split(':')[1])
  const signer = await getHDWalletSigner(chainId)

  const txRequest: ethers.TransactionRequest = {
    to: tx.to,
    value: tx.value,
    gasLimit: tx.gas,
    gasPrice: tx.gasPrice,
    data: tx.data || '0x',
    chainId: numericChainId,
  }

  // Get fresh nonce
  txRequest.nonce = await signer.getNonce()

  // Sign the transaction
  const signedTx = await signer.signTransaction(txRequest)
  return signedTx
}

async function broadcastTransaction(signedTx: string, chainId: string): Promise<string> {
  const numericChainId = parseInt(chainId.split(':')[1])
  const rpcUrl = getRpcUrlForChain(chainId)
  const provider = new ethers.JsonRpcProvider(rpcUrl)
  const txResponse = await provider.broadcastTransaction(signedTx)
  return txResponse.hash
}

// ── Sign a personal message ──
async function signMessageWithHDWallet(
  hexMessage: string,
  address: string
): Promise<string> {
  const signer = await getHDWalletSigner('eip155:137')
  const message = ethers.toUtf8String(hexMessage)
  const signature = await signer.signMessage(message)
  return signature
}

// ── Sign EIP-712 typed data ──
async function signTypedDataWithHDWallet(
  address: string,
  typedData: any
): Promise<string> {
  const signer = await getHDWalletSigner('eip155:137')
  const { domain, types, message } = typedData
  // Remove EIP712Domain from types (ethers handles it)
  delete types.EIP712Domain
  const signature = await signer.signTypedData(domain, types, message)
  return signature
}
```

#### Step 6: Handle Session Disconnect

```typescript
web3wallet.on('session_delete', (event) => {
  const { topic } = event
  // Clean up session data
  // Update UI to show disconnected state
  console.log('Session disconnected:', topic)
})

// To disconnect from the wallet side:
async function disconnectSession(topic: string) {
  await web3wallet.disconnectSession({
    topic,
    reason: { code: 6000, message: 'User disconnected' },
  })
}
```

#### Step 7: Session Persistence

WalletConnect v2 sessions persist automatically. On app launch:

```typescript
async function restoreSessions() {
  const sessions = web3wallet.getActiveSessions()
  // sessions is a map of { [topic]: SessionStruct }
  // Restore UI state for active sessions
  for (const [topic, session] of Object.entries(sessions)) {
    console.log('Active session with:', session.peer.metadata.name)
    // Show connected dApp in the wallet's "Connected Apps" screen
  }
}
```

### 5.5. UI Screens Required on Mobile

| Screen | When Shown | What to Display |
|--------|-----------|-----------------|
| **QR Scanner** | User taps "Scan" or "Connect dApp" | Camera viewfinder, QR decode |
| **Connection Approval** | After scanning QR (session_proposal) | dApp name, URL, icon, requested chains, "Approve" / "Reject" buttons |
| **Transaction Approval** | On `eth_sendTransaction` | To address, value (formatted), gas cost, contract data preview, "Confirm" / "Reject" |
| **Message Signing** | On `personal_sign` | Decoded message text, "Sign" / "Reject" |
| **Typed Data Signing** | On `eth_signTypedData_v4` | Structured data preview, "Sign" / "Reject" |
| **Connected Apps** | Settings / management area | List of active WC sessions, disconnect button per session |
| **Network Switch** | On `wallet_switchEthereumChain` | "PhilSocial Pro wants to switch to Polygon. Allow?" |

### 5.6. RPC Nodes for the Mobile App

The mobile app needs RPC endpoints to broadcast transactions and fetch nonces. Use the same nodes as the web app:

```typescript
function getRpcUrlForChain(chainId: string): string {
  const id = parseInt(chainId.split(':')[1])
  switch (id) {
    case 137:   return 'https://polygon-rpc.com'
    case 1:     return 'https://eth.llamarpc.com'
    case 56:    return 'https://bsc-dataseed.binance.org'
    default:    return 'https://polygon-rpc.com'
  }
}
```

### 5.7. Push Notifications (Optional but Recommended)

WalletConnect v2 supports push notifications via [Echo Server](https://docs.walletconnect.com/advanced/echo-server) so the mobile app gets notified when the web app sends a request (even when the app is in the background):

```typescript
// During Web3Wallet init, register for push
await web3wallet.registerDeviceToken({
  token: 'firebase_fcm_token_here',
  environment: 'production', // or 'sandbox' for iOS
  platform: 'android',       // or 'ios'
})
```

---

## 6. Data Flow Diagrams

### 6.1. Initial Connection Flow

```
User (Web)                   phil-defi                WC Relay          Philcoin Mobile
    │                           │                        │                    │
    │  Click "Philsocial"      │                        │                    │
    │─────────────────────────►│                        │                    │
    │                          │ createInjectedWallet   │                    │
    │                          │ ('WalletConnect',      │                    │
    │                          │  'polygon')            │                    │
    │                          │                        │                    │
    │                          │ getAccounts()          │                    │
    │                          │───────────────────────►│                    │
    │                          │ (no session → connect) │                    │
    │                          │                        │                    │
    │  ◄─── QR Code Modal ────│                        │                    │
    │                          │                        │                    │
    │                          │                        │  User (Mobile)     │
    │                          │                        │       │            │
    │                          │                        │  Scan QR          │
    │                          │                        │◄──────────────────│
    │                          │                        │                    │
    │                          │                        │  session_proposal  │
    │                          │                        │──────────────────►│
    │                          │                        │                    │
    │                          │                        │     [Approve UI]   │
    │                          │                        │                    │
    │                          │                        │  approveSession    │
    │                          │                        │◄──────────────────│
    │                          │                        │                    │
    │                          │ Session established    │                    │
    │                          │◄──────────────────────│                    │
    │                          │                        │                    │
    │                          │ accounts: [0xABC...]   │                    │
    │                          │                        │                    │
    │                          │ connectedAddress =     │                    │
    │                          │   0xABC...             │                    │
    │                          │ provider = WalletConnect│                   │
    │                          │                        │                    │
    │  ◄─── Dashboard ────────│                        │                    │
```

### 6.2. Transaction Signing Flow (e.g., Lending Deposit)

```
User (Web)              phil-defi                 WC Relay           Philcoin Mobile
    │                      │                         │                     │
    │  Click "Deposit"    │                         │                     │
    │────────────────────►│                         │                     │
    │                      │                         │                     │
    │                      │ buildTransaction()      │                     │
    │                      │ (WalletConnect provider) │                    │
    │                      │ → JSON tx object         │                    │
    │                      │                         │                     │
    │                      │ sendRawTransaction()    │                     │
    │                      │ → eth_sendTransaction   │                     │
    │                      │────────────────────────►│                     │
    │                      │                         │  session_request    │
    │                      │                         │───────────────────►│
    │                      │                         │                     │
    │                      │                         │   [TX Approval UI]  │
    │                      │                         │   "Deposit 100 USDC │
    │                      │                         │    to Lending Pool" │
    │                      │                         │                     │
    │                      │                         │  User approves      │
    │                      │                         │                     │
    │                      │                         │  Sign with HD key   │
    │                      │                         │  Broadcast to RPC   │
    │                      │                         │  Get txHash         │
    │                      │                         │                     │
    │                      │                         │  respondSession     │
    │                      │                         │  (txHash)           │
    │                      │                         │◄──────────────────│
    │                      │                         │                     │
    │                      │ txHash received         │                     │
    │                      │◄───────────────────────│                     │
    │                      │                         │                     │
    │                      │ waitForTransaction()    │                     │
    │                      │ (polls RPC directly)    │                     │
    │                      │                         │                     │
    │  ◄── "TX Confirmed" │                         │                     │
```

---

## 7. JSON-RPC Methods to Support

### Required (minimum for phil-defi to work)

| Method | Direction | Purpose | Priority |
|--------|-----------|---------|----------|
| `eth_requestAccounts` | dApp → Wallet | Get wallet address | **Must have** |
| `eth_accounts` | dApp → Wallet | Get wallet address (no prompt) | **Must have** |
| `eth_sendTransaction` | dApp → Wallet | Sign & send a transaction | **Must have** |
| `eth_chainId` | dApp → Wallet | Get current chain ID | **Must have** |
| `wallet_switchEthereumChain` | dApp → Wallet | Switch to different network | **Must have** |
| `wallet_addEthereumChain` | dApp → Wallet | Add a new network | Nice to have |

### Optional (for future features)

| Method | Purpose |
|--------|---------|
| `personal_sign` | Sign a message (login verification, proofs) |
| `eth_signTypedData_v4` | EIP-712 structured signing (permit, gasless approvals) |
| `eth_sign` | Legacy message signing |
| `eth_getBalance` | Get native balance (can proxy to RPC) |
| `eth_call` | Read-only contract call (can proxy to RPC) |

### Events to Emit (Wallet → dApp)

| Event | When |
|-------|------|
| `accountsChanged` | User switches wallet account in mobile app |
| `chainChanged` | User switches network in mobile app |

---

## 8. Security Considerations

### On the Mobile App Side

1. **Never expose private keys or seed phrases** over WalletConnect. Only signed results travel over the relay.
2. **Always show transaction details** before signing. Decode `tx.data` if possible to show human-readable info (e.g., "Approve USDC spending" or "Deposit 100 USDC").
3. **Validate chain ID** — ensure the requested chain matches what the wallet expects. Don't auto-switch without user consent.
4. **Session management** — allow users to view and revoke active sessions. Set reasonable session expiry (e.g., 7 days).
5. **Secure storage** — the mnemonic/seed must be stored in platform-secure storage (iOS Keychain, Android Keystore), never in plain text.
6. **Rate limiting** — don't allow rapid-fire transaction requests without user interaction.
7. **Domain verification** — show the dApp URL and metadata clearly so users can verify they're connecting to the real PhilSocial Pro.

### On the Web App Side

1. **Move WalletConnect Project ID to env variable** — don't hardcode it.
2. **Handle session expiry gracefully** — if the session expires, prompt the user to re-scan.
3. **Handle mobile app being closed** — requests will time out; show appropriate UI (e.g., "Open your Philcoin Wallet to approve").
4. **Don't fall back to NativeProvider** — if the user connected via WalletConnect, never try to sign locally.

---

## 9. Testing Checklist

### Web App

- [ ] Click "Philsocial" → QR code modal appears
- [ ] QR code contains valid `wc:` URI
- [ ] After mobile scan + approve → web app shows connected address
- [ ] `defaultProvider()` returns `'WalletConnect'`
- [ ] Balance fetching works (uses RPC directly, not provider)
- [ ] Lending deposit: buildTransaction → sendTransaction → QR modal does NOT reappear → tx goes to mobile
- [ ] After mobile approves → txHash returned → waitForTransaction works
- [ ] Lending borrow, repay, withdraw all work similarly
- [ ] Payments flow works
- [ ] Rewards claim works
- [ ] Campaign contributions work
- [ ] Stable pool operations work
- [ ] Network switch (if user switches to Ethereum in web app) → mobile gets wallet_switchEthereumChain
- [ ] Session survives page reload (no re-scan needed)
- [ ] Disconnect from web → mobile session cleaned up
- [ ] Disconnect from mobile → web shows disconnected state
- [ ] Error handling: mobile rejects tx → web shows proper error toast

### Mobile App

- [ ] QR scanner opens and scans correctly
- [ ] Session proposal screen shows dApp metadata (name, URL, icon)
- [ ] Approve → session established, address sent to dApp
- [ ] Reject → dApp shows error
- [ ] Transaction approval screen shows: to, value, gas, decoded data
- [ ] Approve tx → signed with HD wallet → broadcast → txHash returned
- [ ] Reject tx → error returned to dApp
- [ ] Message signing shows decoded message
- [ ] Chain switch request handled
- [ ] Connected apps screen shows active sessions
- [ ] Disconnect per session works
- [ ] Push notification on incoming request (if implemented)
- [ ] App killed and reopened → sessions restored
- [ ] Multiple simultaneous sessions work

---

## Summary — What to Build Where

| Component | Web App (phil-defi) | Mobile App (Philcoin Wallet) |
|-----------|--------------------|-----------------------------|
| **Package** | `@walletconnect/ethereum-provider` | `@walletconnect/web3wallet` + `@walletconnect/core` (or native Swift/Kotlin SDK) |
| **New files** | `walletconnect.ts` provider (1 file) | QR Scanner, Session Handler, TX Approval UI, Message Signing UI, Connected Apps screen |
| **Modified files** | `Blockchains` class, `WalletConnector.vue`, `WalletStore.ts` (minor) | Integrate with existing HD wallet signing logic |
| **Effort estimate** | ~1-2 days | ~1-2 weeks (including UI + testing) |
| **Dependency** | WalletConnect Project ID | Same Project ID + RPC node endpoints |
