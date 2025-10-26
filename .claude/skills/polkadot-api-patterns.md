# Skill: polkadot-api Patterns

## Description

Master patterns for using `polkadot-api` (PAPI) correctly. This skill teaches the fundamental differences from the legacy `@polkadot/api` library and ensures type-safe blockchain interactions.

## Core Principle

**ALWAYS use `polkadot-api`, NEVER use `@polkadot/api`**

The polkadot-api library (PAPI) provides:
- Full TypeScript type safety via generated descriptors
- Simpler API surface
- Better performance
- Observable-based patterns
- No manual type definitions needed

## Pattern 1: Client Initialization

### Correct Pattern

```typescript
import { createClient } from 'polkadot-api'
import { getWsProvider } from 'polkadot-api/ws-provider'
import { qfn as chain } from '@polkadot-api/descriptors'

// Create WebSocket provider with callbacks
const provider = getWsProvider('wss://test.qfnetwork.xyz', {
  onStatusChanged: (status) => {
    console.log('Connection status:', status)

    // Handle reconnection
    if (status.type === 'CONNECTED' && hasConnectedBefore) {
      // Invalidate queries on reconnection
      queryClient.invalidateQueries()
    }
  }
})

// Create client (do this ONCE per application)
const client = createClient(provider)

// Get typed API for chain
const api = client.getTypedApi(chain)
```

### Key Points

- Create provider, client, and API **once** (typically in a hook or context)
- Use `getWsProvider` with status callbacks for reconnection handling
- Import chain descriptor from `@polkadot-api/descriptors`
- `getTypedApi()` returns fully typed API based on chain metadata

### Common Mistakes

❌ Creating multiple clients
❌ Not handling reconnection
❌ Using raw `createClient()` without provider
❌ Missing typed API (using untyped client directly)

## Pattern 2: Type Handling - MultiAddress

### The Problem

Substrate chains use `MultiAddress` enum for account addressing. You must wrap addresses correctly.

### Correct Pattern

```typescript
import { MultiAddress } from '@polkadot-api/descriptors'

// For recipient/beneficiary/target parameters
const recipient = MultiAddress.Id(recipientAddress)

// In transaction
api.tx.Assets.mint({
  id: assetId,
  beneficiary: MultiAddress.Id(beneficiaryAddress),
  amount: amount
})

// In query
const balance = await api.query.Assets.Account.getValue(
  assetId,
  MultiAddress.Id(accountAddress)
)
```

### Common Mistakes

❌ Passing raw string: `beneficiary: recipientAddress`
❌ Manual object construction: `{ Id: address }`
❌ Wrong import source

### When to Use

Use `MultiAddress.Id()` for:
- `beneficiary` parameters
- `target` parameters
- `admin` parameters
- Any parameter typed as `MultiAddress` in chain metadata

## Pattern 3: Binary Conversion

### The Problem

Chain metadata (names, symbols) requires `Binary` type, not strings.

### Correct Pattern

```typescript
import { Binary } from 'polkadot-api'

// Convert strings to Binary
api.tx.Assets.set_metadata({
  id: assetId,
  name: Binary.fromText('My Token'),
  symbol: Binary.fromText('MTK'),
  decimals: 12
})
```

### Common Mistakes

❌ Passing raw strings: `name: 'My Token'`
❌ Manual hex encoding
❌ Using `TextEncoder`

### Reading Binary Values

```typescript
// Binary values can be read back to strings
const metadata = await api.query.Assets.Metadata.getValue(assetId)
const name = Binary.asText(metadata.name)
const symbol = Binary.asText(metadata.symbol)
```

## Pattern 4: Batch Operations

### The Problem

Multiple operations need atomic execution (all succeed or all fail).

### Correct Pattern

```typescript
import type { TxCallData } from 'polkadot-api'

// Create individual calls and get decodedCall
const createCall = api.tx.Assets.create({
  id: assetId,
  admin: MultiAddress.Id(adminAddress),
  min_balance: minBalance
}).decodedCall  // ← KEY: .decodedCall property

const metadataCall = api.tx.Assets.set_metadata({
  id: assetId,
  name: Binary.fromText(name),
  symbol: Binary.fromText(symbol),
  decimals: decimals
}).decodedCall  // ← KEY: .decodedCall property

// Collect calls in typed array
const calls: TxCallData[] = [createCall, metadataCall]

// Batch with object syntax (not array!)
return api.tx.Utility.batch_all({ calls })
```

### Key Points

- Always use `.decodedCall` property to get the call data
- Type array as `TxCallData[]`
- Pass to `batch_all` as object: `{ calls }`
- NOT as array: `batch_all(calls)` ❌

### Common Mistakes

❌ Forgetting `.decodedCall`: `const call = api.tx.Assets.create({...})`
❌ Wrong parameter format: `batch_all(calls)` instead of `batch_all({ calls })`
❌ Not importing `TxCallData` type

### Conditional Batching

```typescript
const calls: TxCallData[] = [createCall, metadataCall]

// Add optional operations
if (shouldMint) {
  const mintCall = api.tx.Assets.mint({
    id: assetId,
    beneficiary: MultiAddress.Id(recipient),
    amount: amount
  }).decodedCall

  calls.push(mintCall)
}

return api.tx.Utility.batch_all({ calls })
```

## Pattern 5: Transaction Observables

### The Problem

Transactions are async operations with multiple lifecycle stages.

### Correct Pattern

```typescript
import type { TxBroadcastEvent } from 'polkadot-api'

// Create transaction
const tx = api.tx.Assets.mint({
  id: assetId,
  beneficiary: MultiAddress.Id(recipient),
  amount: amount
})

// Subscribe to lifecycle events
tx.subscribe({
  next: (event: TxBroadcastEvent) => {
    if (event.type === 'signed') {
      console.log('Transaction signed by user')
    }
    else if (event.type === 'broadcasted') {
      console.log('Transaction broadcasted:', event.txHash)
    }
    else if (event.type === 'txBestBlocksState') {
      if (event.found) {
        console.log('Transaction in block:', event.block.hash)
        console.log('Block index:', event.block.index)
      }
    }
    else if (event.type === 'finalized') {
      console.log('Transaction finalized!')
      console.log('Block hash:', event.block.hash)

      // Check for dispatch errors
      if (event.dispatchError) {
        handleDispatchError(event.dispatchError)
      }
    }
  },
  error: (error: Error) => {
    console.error('Transaction error:', error)

    // Handle different error types
    if (error instanceof InvalidTxError) {
      // Transaction validation failed
      handleInvalidTx(error)
    } else if (isUserRejection(error)) {
      // User cancelled in wallet
      handleUserRejection()
    }
  }
})
```

### Lifecycle Stages

1. **signed** - User signed in wallet
2. **broadcasted** - Sent to network (get txHash here)
3. **txBestBlocksState** - Included in a block
4. **finalized** - Block is finalized (wait for this!)

### Error Handling

**InvalidTxError** - Transaction validation failed (before broadcast):
```typescript
import { InvalidTxError } from 'polkadot-api'

if (error instanceof InvalidTxError) {
  // Common causes: insufficient balance, stale nonce, bad proof
  const { type, value } = parseInvalidTxError(error)

  if (value === 'Payment') {
    return 'Insufficient balance to pay fees'
  }
}
```

**Dispatch Error** - Transaction failed during execution (after finalized):
```typescript
if (event.type === 'finalized' && event.dispatchError) {
  const parsed = parseDispatchError(event.dispatchError)
  // parsed: { pallet: 'Assets', errorName: 'InsufficientBalance' }

  const userMessage = getErrorMessage(parsed.pallet, parsed.errorName)
}
```

### Common Mistakes

❌ Treating transaction as Promise: `await tx()`
❌ Not handling all event types
❌ Missing error handler
❌ Not checking `dispatchError` in finalized event
❌ Not unsubscribing when done

### Proper Cleanup

```typescript
const subscription = tx.subscribe({ next, error })

// Later, cleanup
subscription.unsubscribe()
```

## Pattern 6: Query with TanStack Query

### The Problem

Chain state queries need caching, invalidation, and reactivity.

### Correct Pattern

```typescript
import { useQuery } from '@tanstack/react-query'
import { useConnectionContext } from '@/hooks'

function useAssetMetadata(assetId: number) {
  const { api } = useConnectionContext()

  return useQuery({
    queryKey: ['assetMetadata', assetId],
    queryFn: async () => {
      // PAPI queries return values directly (no .toJSON())
      const metadata = await api.query.Assets.Metadata.getValue(assetId)

      return {
        name: Binary.asText(metadata.name),
        symbol: Binary.asText(metadata.symbol),
        decimals: metadata.decimals
      }
    },
    staleTime: 30_000,  // Consider stale after 30s
    gcTime: 300_000,    // Garbage collect after 5min
    enabled: assetId !== undefined  // Only run if we have ID
  })
}
```

### Key Points

- `queryKey` must include all dependencies
- PAPI returns decoded values directly (no `.toJSON()`)
- Use `getValue()` for single values
- Use `getEntries()` for multi-key queries
- Set appropriate `staleTime` and `gcTime`

### Multi-Key Queries

```typescript
// Query all accounts for an asset
const entries = await api.query.Assets.Account.getEntries(assetId)

// Returns array of [key, value] tuples
entries.forEach(([key, value]) => {
  const [assetId, accountAddress] = key.args
  console.log(`Account ${accountAddress} has ${value.balance}`)
})
```

### Live Queries (Observables)

```typescript
// For real-time updates
queryFn: () => new Promise((resolve, reject) => {
  const unsubscribe = api.query.Assets.Account.watchValue(
    assetId,
    MultiAddress.Id(address),
    (value) => {
      resolve(value)
      unsubscribe()
    }
  )
})
```

### Common Mistakes

❌ Forgetting to include dependencies in `queryKey`
❌ Using `.toJSON()` (not needed with PAPI)
❌ Not setting `staleTime` and `gcTime`
❌ Querying in effects instead of useQuery

## Pattern 7: Token Amount Handling

### The Problem

Blockchain uses smallest unit (like wei), UI uses readable decimals.

### Correct Pattern

```typescript
// Utilities (from template)
export function parseUnits(value: string, decimals: number): bigint {
  const [whole, fraction = ''] = value.split('.')
  const paddedFraction = fraction.padEnd(decimals, '0').slice(0, decimals)
  return BigInt(whole + paddedFraction)
}

export function formatUnits(value: bigint, decimals: number): string {
  const str = value.toString().padStart(decimals + 1, '0')
  const whole = str.slice(0, -decimals) || '0'
  const fraction = str.slice(-decimals).replace(/0+$/, '')
  return fraction ? `${whole}.${fraction}` : whole
}

// Usage in operations
function mintTokens(amount: string, decimals: number) {
  const chainAmount = parseUnits(amount, decimals)

  return api.tx.Assets.mint({
    id: assetId,
    beneficiary: MultiAddress.Id(recipient),
    amount: chainAmount  // bigint
  })
}

// Usage in display
function AssetBalance({ balance, decimals }: Props) {
  const displayAmount = formatUnits(balance, decimals)
  return <div>{displayAmount} tokens</div>
}
```

### Key Points

- Always use `bigint` for chain amounts
- Never use `Number()` (loses precision for large values)
- User input (string) → `parseUnits` → bigint → chain
- Chain response (bigint) → `formatUnits` → string → display

### Common Mistakes

❌ Using `Number()`: `Number(userInput) * 10 ** decimals`
❌ Manual decimal math
❌ Displaying raw bigint values
❌ Not handling fraction precision

## Pattern 8: TypedApi for Type Safety

### The Problem

Generic APIs lose type information.

### Correct Pattern

```typescript
import type { TypedApi } from 'polkadot-api'
import { qfn } from '@polkadot-api/descriptors'

// Define typed API type
type QfnApi = TypedApi<typeof qfn>

// Use in function signatures
export function createAsset(
  api: QfnApi,
  params: CreateAssetParams
) {
  // TypeScript knows all pallets and extrinsics
  return api.tx.Assets.create({
    id: params.id,
    admin: MultiAddress.Id(params.admin),
    min_balance: params.minBalance
  })
  // Full autocomplete and type checking!
}

// Get type of specific pallet transaction
type AssetCreateTx = ReturnType<QfnApi['tx']['Assets']['create']>
```

### Benefits

- Full autocomplete in IDE
- Compile-time type checking
- Refactoring safety
- Self-documenting code

### Common Mistakes

❌ Using `any` for API parameter
❌ Not importing `TypedApi` type
❌ Losing type information in function boundaries

## Pattern 9: Constants and Metadata

### Constants

```typescript
// Query chain constants (free, no transaction)
const existentialDeposit = await api.constants.Balances.ExistentialDeposit()
const maxAssets = await api.constants.Assets.MaxAssets()

// Use in validation
if (balance < existentialDeposit) {
  throw new Error(`Balance must be >= ${existentialDeposit}`)
}
```

### Runtime Version

```typescript
// Get runtime version
const runtime = await api.runtime.latest()
console.log('Runtime version:', runtime.specVersion)
```

## Pattern 10: Testing Transactions

### Fee Estimation

```typescript
// Get fee without executing
const feeEstimate = await api.tx.Assets.transfer({
  id: assetId,
  target: MultiAddress.Id(recipient),
  amount: amount
}).getEstimatedFees(fromAddress)

console.log('Fee:', formatUnits(feeEstimate, 12), 'QFN')
```

### Dry Run

```typescript
// Test if transaction would succeed
try {
  const result = await api.tx.Assets.transfer({
    id: assetId,
    target: MultiAddress.Id(recipient),
    amount: amount
  }).dryRun(fromAddress)

  if (result.success) {
    console.log('Transaction would succeed')
  } else {
    console.error('Would fail with:', result.dispatchError)
  }
} catch (error) {
  console.error('Validation failed:', error)
}
```

## Anti-Patterns to Avoid

### ❌ Don't Mix PAPI and Legacy API

```typescript
// NEVER do this
import { ApiPromise } from '@polkadot/api'  // ❌
import { createClient } from 'polkadot-api'  // ✅
```

### ❌ Don't Use Type Assertions

```typescript
// Bad
const tx = api.tx.Assets.mint({...}) as any

// Good - let types infer
const tx = api.tx.Assets.mint({...})
```

### ❌ Don't Create Clients in Components

```typescript
// Bad - creates new client on every render
function Component() {
  const client = createClient(getWsProvider(url))  // ❌
}

// Good - use context or top-level hook
const { api } = useConnectionContext()  // ✅
```

### ❌ Don't Ignore Errors

```typescript
// Bad
tx.subscribe({
  next: (event) => {
    if (event.type === 'finalized') {
      // Assuming success ❌
      showSuccess()
    }
  }
})

// Good
tx.subscribe({
  next: (event) => {
    if (event.type === 'finalized') {
      if (event.dispatchError) {
        handleError(event.dispatchError)  // ✅
      } else {
        showSuccess()
      }
    }
  },
  error: (err) => handleError(err)  // ✅
})
```

## Checklist for New Features

When implementing a new blockchain feature:

- [ ] Import from `polkadot-api` (never `@polkadot/api`)
- [ ] Use `TypedApi` for type safety
- [ ] Wrap addresses with `MultiAddress.Id()`
- [ ] Convert strings with `Binary.fromText()`
- [ ] Use `.decodedCall` for batch operations
- [ ] Handle all transaction event types
- [ ] Check for `dispatchError` in finalized events
- [ ] Implement proper error handling
- [ ] Use `parseUnits`/`formatUnits` for amounts
- [ ] Set up TanStack Query with appropriate stale times
- [ ] Test with fee estimation before execution

## Resources

- Official docs: https://papi.how
- Codegen guide: https://papi.how/codegen
- Examples: See `src/lib/assetOperations.ts` in template
- Discord: Polkadot API channel

## Updates

This skill is based on `polkadot-api` v1.20.0. Patterns may evolve with new versions.
