# Chainrails Payment Integration

This document describes how Chainrails cross-chain payment support was added to the Death Mountain application.

## Overview

Chainrails integration allows users to pay for dungeon entry using credit cards or tokens from other blockchains (Base, Ethereum, etc.). The payment is bridged to Starknet as USDC, then automatically swapped to TICKET tokens and used to enter the dungeon.

## Architecture

```
User (another chain) 
    │
    │ (PaymentModal from @chainrails/react)
    ▼
Chainrails Bridge
    │ (swaps to USDC on Starknet)
    ▼
Controller Wallet (USDC deposited)
    │
    │ (onSuccess callback triggers buyDungeonTicket)
    │ (swap USDC → TICKET via Ekubo)
    ▼
Dungeon Contract
    │
    ▼
Game Token (minted to user)
```

## Implementation Details

### 1. Client-Side Changes

#### Payment Flow State Management
- Added payment flow states: `INITIAL`, `CONTROLLER`, `CHAINRAILS`
- Users first choose between "Pay from Starknet" or "Pay from another chain"
- Chainrails flow shows USDC amount and opens payment modal

**File:** `client/src/components/PaymentOptionsModal.tsx`

**Key Changes:**
- Added `PaymentModal` component from `@chainrails/react`
- Integrated `useChainrailsPayment` hook
- Modified `buyDungeonTicket()` to accept `isChainrails` parameter
- Added USDC quote fetching (`fetchUSDCAmount`)
- Added back button navigation between payment flows
- Implemented dynamic subtitle text that changes based on payment flow state:
  - `INITIAL`: "Select how you want to pay for access"
  - `CONTROLLER`: "Select any token in your controller wallet"
  - `CHAINRAILS`: "Pay with a token on another chain"

#### Custom Hook
**File:** `client/src/hooks/useChainrailsPayment.ts`

- Wraps `usePaymentSession` from `@chainrails/react`
- Constructs session URL pointing to server endpoint
- Handles payment success callback to trigger dungeon entry

**Session URL Format:**
```
http://localhost:3000/create-session?recipient={controllerAddress}&destinationChain=STARKNET&token=USDC
```

### 2. Server-Side Changes

#### Express Server
**File:** `server/index.ts`

- Creates `/create-session` endpoint
- Uses Chainrails SDK to generate payment session tokens
- Accepts query parameters: `recipient`, `destinationChain`, `token`
- Returns session token for `PaymentModal` component

**Dependencies:**
- `@chainrails/sdk` - Chainrails SDK for session creation
- `express` - Server framework
- `cors` - CORS middleware
- `dotenv` - Environment variable management

### 3. Payment Flow

1. **User selects "Pay from another chain"**
   - Sets payment flow to `CHAINRAILS`
   - Fetches USDC quote for dungeon entry

2. **User clicks payment button**
   - Opens `PaymentModal` from `@chainrails/react`
   - User completes payment on their preferred chain

3. **Chainrails processes payment**
   - Bridges payment to Starknet as USDC
   - Deposits USDC to user's controller wallet address

4. **Payment success callback**
   - `onSuccess` callback in `useChainrailsPayment` triggers
   - Calls `buyDungeonTicket(true)` with `isChainrails` flag

5. **Automatic swap and entry**
   - `buyDungeonTicket()` uses USDC token data
   - Gets swap quote from Ekubo (USDC → TICKET)
   - Generates swap calls and executes in single transaction
   - Enters dungeon with TICKET tokens

## Key Files Modified

### Client
- `client/src/components/PaymentOptionsModal.tsx` - Main payment UI with Chainrails integration
- `client/src/hooks/useChainrailsPayment.ts` - Custom hook for Chainrails payment session
- `client/package.json` - Added `@chainrails/react` and `@chainrails/sdk` dependencies

### Server
- `server/index.ts` - Express server with Chainrails session endpoint
- `server/package.json` - Server dependencies

## Configuration

### Environment Variables
- `CHAINRAILS_API_KEY` - Chainrails API key (set in `server/index.ts` or `.env`)

### Server Setup
- Server runs on port 3000 (default)
- Must be running for Chainrails payment sessions to work
- Session URL in `useChainrailsPayment.ts` points to `http://localhost:3000/create-session`

## Integration Points

### Controller Address
- Uses `useController().address` as recipient for Chainrails payments
- This is the Cartridge controller wallet address where USDC is deposited

### Swap Integration
- Reuses existing Ekubo swap infrastructure
- Uses `getSwapQuote()` and `generateSwapCalls()` from `client/src/api/ekubo.ts`
- Swaps USDC → TICKET, then enters dungeon with TICKET tokens

### Dungeon Entry
- Uses existing `enterDungeon()` function from `useController()`
- Combines swap calls + approve + buy_game in single transaction
