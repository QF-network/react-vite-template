# CLAUDE.md

Quantum Fusion Network (QFN) asset management app - React 19 + TypeScript + polkadot-api

## CODE QUALITY RULES (Apply to Every Change)

**State Management:**
- NEVER use `useReducer` - use `useState` or context
- Component state with `useState`, shared state via Context
- Server state via TanStack Query only

**TypeScript:**
- NEVER use `any` - use `unknown` and narrow types
- NEVER use type assertions (`as`) - let types prove correctness
- NEVER create redundant type definitions - leverage inference
- Prefer narrow types (literals, discriminated unions) over broad types

**Architecture:**
- Components are presentational - minimal logic
- Business logic in `lib/` folder and custom hooks
- Pure functions, immutability, early returns
- ALL exports through barrel files (`index.ts`) - import from `@/lib`, `@/hooks`, `@/components`

## STATE MANAGEMENT ARCHITECTURE

**Wallet State:** `WalletContext` + `useWallet` hook
- Extension connection, account selection, auto-reconnect
- Persists to localStorage

**Connection State:** `ConnectionContext` + `useConnectionStatus` hook
- Creates polkadot-api client (NO separate chain.ts file)
- Connection status, auto-reconnect with query invalidation

**Transaction State:** `TransactionContext` + `useTransaction` hook
- Tracks: signing → broadcasting → inBlock → finalized/error
- `useTransactionToasts` observes and displays notifications

**Server State:** TanStack Query (30s stale, 5min GC)

## KEY FILE LOCATIONS

**lib/** - Business logic and utilities (all exported via `lib/index.ts`)
- `balance/` - Balance conversion and formatting utilities
  - `toPlanck.ts` - Convert human-readable to Planck units (bigint)
  - `fromPlanck.ts` - Convert Planck units to human-readable string
  - `format.ts` - formatBalance with locale, rounding, symbol support
  - `config.ts` - Balance constants and error codes
- `utils.ts` - cn() for Tailwind class merging
- `walletStorage.ts` - localStorage persistence for wallet connection
- `toastConfigs.ts` - Transaction toast configurations (ToastConfig type)
- `queryClient.ts` - TanStack Query setup
- `errorParsing.ts` / `errorMessages.ts` / `transactionErrors.ts` - Error handling

**hooks/** - Custom React hooks (all exported via `hooks/index.ts`)
- `useConnectionStatus.ts` - Creates polkadot-api client, manages connection
- `useConnectionContext.ts` - Access connection state and API client
- `useWallet.ts` - Core wallet connection logic
- `useWalletContext.ts` - Access wallet state
- `useTransactionManager.ts` - Internal transaction lifecycle manager (used by TransactionProvider)
- `useTransaction.ts` - Execute transactions with lifecycle tracking
- `useTransactionContext.ts` - Access transaction state
- `useTransactionToasts.ts` - Display transaction notifications
- `useFee.ts` - Calculate transaction fees with real-time estimation

**contexts/** - React Context providers (exported via `contexts/index.ts`)
- `WalletContext.tsx` - Wallet state provider (uses `useWallet`)
- `ConnectionContext.tsx` - Connection state provider (uses `useConnectionStatus`)
- `TransactionContext.tsx` - Transaction state provider (uses `useTransactionManager`)

**components/** - UI components (all exported via `components/index.ts`)
- `WalletConnector.tsx` - Wallet connection UI
- `AccountSelector.tsx` - Account selection dropdown
- `AccountDashboard.tsx` - Account balance and faucet link
- `ConnectionBanner.tsx` - Connection status banner
- `FeeDisplay.tsx` - Display transaction fee
- `TransactionFormFooter.tsx` - Common form footer with fee display
- `TransactionReview.tsx` - Review transaction before signing (JSON preview)
- `MutationError.tsx` - Display mutation errors
- `error-boundaries/` - Error boundary components
  - `AppErrorBoundary.tsx` - Top-level error boundary
  - `FeatureErrorBoundary.tsx` - Feature-level error boundary
  - `ComponentErrorBoundary.tsx` - Component-level error boundary

## TRANSACTION FLOW

1. Component uses mutation hook (or custom logic) with operation function
2. On mutation trigger, `useTransaction().executeTransaction()` is called
3. TransactionManager tracks lifecycle: idle → signing → broadcasting → inBlock → finalized
4. `useTransactionToasts` observes transaction state and displays notifications
5. Cleanup after 500ms, queries invalidated on success

## BALANCE UTILITIES

**Conversion:**
- `toPlanck(value: string, decimals: number): bigint` - Convert "1.5" to 1500000000000000000n
- `fromPlanck(value: bigint, decimals?: number): string` - Convert 1500000000000000000n to "1.5"

**Formatting:**
- `formatBalance(value: string, options?: FormatBalanceOptions): string`
  - Options: `symbol`, `displayDecimals`, `locale`, `roundingMode`
  - Handles locale-specific formatting (commas, periods)
  - Supports rounding modes: 'round', 'floor', 'ceil'
  - Example: `formatBalance("1234.5678", { symbol: "QF", displayDecimals: 2 })` → "1,234.57 QF"

**Import from main barrel:**
```ts
import { toPlanck, fromPlanck, formatBalance } from '@/lib'
```

## FEE ESTIMATION

**useFee Hook:**
- Real-time fee estimation with 500ms debounce (uses `useDeferredValue`)
- Call `transaction.getPaymentInfo(address, { at: 'best' })`
- Returns: `{ fee, isLoading, error }`

**Usage:**
```ts
import { useFee } from '@/hooks'

const { mutation, transaction } = useMyMutation({ ... })
const feeState = useFee(transaction, selectedAccount?.address)

// Use in footer
<TransactionFormFooter feeState={feeState} {...} />
```

## SHARED FORM COMPONENTS

**TransactionFormFooter** - Consistent footer for all transaction forms:
```ts
<TransactionFormFooter
  feeState={{ fee, isLoading, error }}
  isDisabled={!isFormValid}
  isPending={mutation.isPending}
  actionText="Submit Transaction"
  variant="default"  // or "destructive"
/>
```

**TransactionReview** - JSON preview for forms:
```ts
<TransactionReview
  data={formData}
  variant="default"  // or "destructive"
/>
```

**AccountDashboard** - Native balance + faucet link:
```ts
<AccountDashboard />  // Shows QF balance, faucet button
```

**Standard Form Layout:**
```tsx
<div className="space-y-6">
  <AccountDashboard />
  <Card>
    <form onSubmit={handleSubmit}>
      <div className="grid grid-cols-1 gap-6 lg:grid-cols-3">
        <div className="lg:col-span-2">{/* Form fields */}</div>
        <div className="lg:col-span-1">
          <TransactionReview data={formData} />
        </div>
      </div>
      <TransactionFormFooter feeState={feeState} {...} />
    </form>
  </Card>
</div>
```

## BLOCKCHAIN SPECIFICS

**Network:** QF Network testnet `wss://test.qfnetwork.xyz`

**polkadot-api:**
- Descriptors in `.papi/descriptors` (auto-generated via `papi` command)
- Metadata in `.papi/metadata/*.scale` files
- Client created in `useConnectionStatus` hook

## PROJECT CONVENTIONS

- Toast duration: 30s (Sonner)
- TanStack Query: 30s stale time, 5min garbage collection
- Wallet auto-reconnects on page load if previously connected
- Error boundaries at app/feature/component levels
- All exports through barrel files for clean imports
- Balance decimals: Default 18 for QF native token
- Query refetch: 10s for balances, 5min stale time for metadata

## USING THIS TEMPLATE

**Pre-built infrastructure:**
- ✅ Wallet connection (WalletContext + useWallet)
- ✅ Chain connection (ConnectionContext + useConnectionStatus)
- ✅ Transaction management (TransactionContext + useTransaction)
- ✅ Fee estimation (useFee + FeeDisplay)
- ✅ Balance utilities (toPlanck, fromPlanck, formatBalance)
- ✅ Shared form components (TransactionFormFooter, TransactionReview, AccountDashboard)
- ✅ Error handling (errorParsing, transactionErrors, error boundaries)
- ✅ UI components (shadcn/ui + generic components)

**To generate features:**
1. Create wizard-config.json (or use wizard)
2. Open Cursor Composer
3. Type: "Generate my app based on wizard-config.json"
4. Run: `pnpm install && pnpm papi && pnpm dev`

**Never regenerate infrastructure files.** They're tested and follow established patterns.

## ADDING FEATURES

**Generate from config:**
```bash
# Generate prompts from wizard config
pnpm tsx workflow/generate-prompts.ts

# Or ask Claude directly
"Add [feature] based on wizard-config.json"
```

**Add features iteratively:**
```
"Add NFT minting capability"
"Add freeze/thaw operations for assets"
```

Claude will:
1. Load relevant skills from `.claude/skills/`
2. Generate operation functions and components
3. Update App.tsx navigation
4. Use template infrastructure (balance utilities, shared components, hooks)
