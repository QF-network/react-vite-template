# Asset Operations Reference Implementation

This document provides the complete, production-ready implementation of Assets pallet operations. Use this as a template when implementing asset management features.

## Complete Implementation

### File: `src/lib/assetOperations.ts`

```typescript
import { Binary, type TxCallData, type TypedApi } from 'polkadot-api'

import { MultiAddress, type qfn } from '@polkadot-api/descriptors'

import { parseUnits } from './utils'

// Define typed API for QFN chain
type QfnApi = TypedApi<typeof qfn>

// ========== TYPE DEFINITIONS ==========

export interface CreateAssetParams {
  assetId: string              // Asset ID (numeric string)
  minBalance: string           // Minimum balance in display units
  name: string                 // Token name (e.g., "My Token")
  symbol: string               // Token symbol (e.g., "MTK")
  decimals: string             // Decimal places (numeric string)
  initialMintAmount: string    // Optional initial mint amount
  initialMintBeneficiary: string  // Required if initialMintAmount > 0
}

export interface MintParams {
  assetId: string    // Asset ID to mint
  recipient: string  // Beneficiary address
  amount: string     // Amount in display units
  decimals: number   // Token decimals
}

export interface TransferParams {
  assetId: string    // Asset ID to transfer
  recipient: string  // Target address
  amount: string     // Amount in display units
  decimals: number   // Token decimals
}

export interface DestroyAssetParams {
  assetId: string    // Asset ID to destroy
}

// ========== OPERATIONS ==========

/**
 * Create Asset (Batch Operation)
 *
 * Creates a new fungible asset with metadata and optional initial mint.
 * Uses batch_all to ensure atomicity - all operations succeed or all fail.
 *
 * Steps:
 * 1. Create asset class
 * 2. Set metadata (name, symbol, decimals)
 * 3. Optionally mint initial supply
 *
 * @param api - Typed API instance
 * @param params - Creation parameters
 * @param signerAddress - Admin address (becomes asset owner)
 * @returns Transaction observable
 */
export const createAssetBatch = (
  api: QfnApi,
  params: CreateAssetParams,
  signerAddress: string
) => {
  // Parse string parameters to appropriate types
  const assetId = parseInt(params.assetId)
  const minBalance = BigInt(params.minBalance) * 10n ** BigInt(params.decimals)

  // Step 1: Create the asset class
  // Admin can mint, freeze, and destroy the asset
  // min_balance prevents dust accounts
  const createCall = api.tx.Assets.create({
    id: assetId,
    admin: MultiAddress.Id(signerAddress),  // Wrap address in MultiAddress
    min_balance: minBalance,                 // In smallest units
  }).decodedCall  // ← CRITICAL: Get decoded call data for batching

  // Step 2: Set asset metadata
  // This is separate from creation and can be updated later
  const metadataCall = api.tx.Assets.set_metadata({
    id: assetId,
    name: Binary.fromText(params.name),      // Convert string to Binary
    symbol: Binary.fromText(params.symbol),  // Convert string to Binary
    decimals: parseInt(params.decimals),
  }).decodedCall  // ← CRITICAL: Get decoded call data

  // Collect calls for batching
  const calls: TxCallData[] = [createCall, metadataCall]

  // Step 3: Optional initial mint
  // Only add if user specified an initial amount
  if (params.initialMintAmount && parseFloat(params.initialMintAmount) > 0) {
    // Convert display amount to chain units (smallest denomination)
    const mintAmount = parseUnits(
      params.initialMintAmount,
      parseInt(params.decimals)
    )

    const mintTx = api.tx.Assets.mint({
      id: assetId,
      beneficiary: MultiAddress.Id(params.initialMintBeneficiary),
      amount: mintAmount,
    }).decodedCall  // ← CRITICAL: Get decoded call data

    calls.push(mintTx)
  }

  // Return batched transaction
  // batch_all ensures atomicity: all succeed or all revert
  return api.tx.Utility.batch_all({ calls })  // Note: { calls }, not just calls
}

/**
 * Mint Tokens
 *
 * Create new tokens for an existing asset.
 * Only asset admin/issuer can mint.
 *
 * @param api - Typed API instance
 * @param params - Mint parameters
 * @returns Transaction observable
 */
export const mintTokens = (api: QfnApi, params: MintParams) => {
  const assetId = parseInt(params.assetId)
  // Convert display units to chain units
  const amount = parseUnits(params.amount, params.decimals)

  return api.tx.Assets.mint({
    id: assetId,
    beneficiary: MultiAddress.Id(params.recipient),  // Who receives tokens
    amount,  // BigInt in smallest units
  })
}

/**
 * Transfer Tokens
 *
 * Transfer asset tokens between accounts.
 * Sender is implicit (transaction signer).
 *
 * @param api - Typed API instance
 * @param params - Transfer parameters
 * @returns Transaction observable
 */
export const transferTokens = (api: QfnApi, params: TransferParams) => {
  const assetId = parseInt(params.assetId)
  const amount = parseUnits(params.amount, params.decimals)

  return api.tx.Assets.transfer({
    id: assetId,
    target: MultiAddress.Id(params.recipient),  // Note: parameter name is "target"
    amount,
  })
}

/**
 * Destroy Asset (Batch Operation)
 *
 * Permanently removes an asset and all associated state.
 * This is a multi-step process that must execute in order.
 *
 * WARNING: All token holders will lose their balances!
 *
 * Steps (must be in this order):
 * 1. Freeze asset (prevent new operations)
 * 2. Start destruction (mark for removal)
 * 3. Destroy approvals (remove allowances)
 * 4. Destroy accounts (remove all balances)
 * 5. Finish destruction (remove asset class)
 *
 * @param api - Typed API instance
 * @param params - Destruction parameters
 * @returns Transaction observable
 */
export const destroyAssetBatch = (api: QfnApi, params: DestroyAssetParams) => {
  const assetId = parseInt(params.assetId)

  // Step 1: Freeze asset (no more transfers allowed)
  const freezeCall = api.tx.Assets.freeze_asset({
    id: assetId,
  }).decodedCall

  // Step 2: Start destruction process
  const startDestroyCall = api.tx.Assets.start_destroy({
    id: assetId,
  }).decodedCall

  // Step 3: Remove all approvals first
  const destroyApprovalsCall = api.tx.Assets.destroy_approvals({
    id: assetId,
  }).decodedCall

  // Step 4: Remove all account balances
  // WARNING: All holders lose their tokens permanently
  const destroyAccountsCall = api.tx.Assets.destroy_accounts({
    id: assetId,
  }).decodedCall

  // Step 5: Complete destruction, remove asset
  const finishDestroyCall = api.tx.Assets.finish_destroy({
    id: assetId,
  }).decodedCall

  // Order matters! Cannot be rearranged
  const calls: TxCallData[] = [
    freezeCall,
    startDestroyCall,
    destroyApprovalsCall,  // Must come before accounts
    destroyAccountsCall,
    finishDestroyCall,
  ]

  // batch_all ensures atomicity
  return api.tx.Utility.batch_all({ calls })
}
```

## Key Pattern Explanations

### 1. Type Safety with TypedApi

```typescript
type QfnApi = TypedApi<typeof qfn>
```

This gives full TypeScript autocomplete and type checking for all pallets and extrinsics on QFN chain.

### 2. String Parameters in Interfaces

```typescript
interface CreateAssetParams {
  assetId: string      // From form input
  decimals: string     // From form input
  amount: string       // From form input
}
```

**Why strings?**
- Form inputs are always strings
- Prevents precision loss for large numbers
- Parse to correct type inside operation function

### 3. Amount Conversion Pattern

```typescript
// User enters "100.5" with 12 decimals
const displayAmount = "100.5"
const decimals = 12

// Convert to chain units (100500000000000n)
const chainAmount = parseUnits(displayAmount, decimals)

// Use in transaction
api.tx.Assets.mint({ amount: chainAmount })
```

**Always:** Display units → `parseUnits` → BigInt → chain

### 4. MultiAddress Wrapping

```typescript
// WRONG
beneficiary: recipientAddress

// CORRECT
beneficiary: MultiAddress.Id(recipientAddress)
```

Substrate uses `MultiAddress` enum for flexibility. Always wrap addresses.

### 5. Binary Conversion

```typescript
// WRONG
name: "My Token"

// CORRECT
name: Binary.fromText("My Token")
```

Chain metadata requires `Binary` type, not raw strings.

### 6. Batch Operation Pattern

```typescript
// Get decoded calls
const call1 = api.tx.Pallet.extrinsic({...}).decodedCall
const call2 = api.tx.Pallet.extrinsic({...}).decodedCall

// Type array
const calls: TxCallData[] = [call1, call2]

// Batch with object syntax
return api.tx.Utility.batch_all({ calls })
```

**Critical points:**
- Use `.decodedCall` property
- Type as `TxCallData[]`
- Pass as `{ calls }` not `calls`

### 7. Conditional Operations

```typescript
const calls: TxCallData[] = [requiredCall1, requiredCall2]

if (optionalCondition) {
  const optionalCall = api.tx.Pallet.extrinsic({...}).decodedCall
  calls.push(optionalCall)
}

return api.tx.Utility.batch_all({ calls })
```

Build array dynamically, batch at end.

## Usage in Components

### With useAssetMutation Hook

```typescript
import { useAssetMutation } from '@/hooks'
import { createAssetBatch, type CreateAssetParams } from '@/lib'

function CreateAsset() {
  const { selectedAccount } = useWalletContext()
  const { api } = useConnectionContext()
  const [formData, setFormData] = useState<CreateAssetParams>({...})

  const { mutation, transaction } = useAssetMutation<CreateAssetParams>({
    params: formData,
    operationFn: (params) => createAssetBatch(api, params, selectedAccount.address),
    toastConfig: createAssetToasts,
    transactionKey: 'createAsset',
    isValid: (params) => params.name !== '' && params.symbol !== '',
    onSuccess: async () => {
      await invalidateAssetQueries(queryClient)
      setFormData(initialState)
    }
  })

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault()
    mutation.mutate()
  }

  return <form onSubmit={handleSubmit}>...</form>
}
```

## Error Handling

### Common Asset Pallet Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Unknown` | Asset doesn't exist | Check asset ID |
| `InUse` | Asset has active state | Cannot destroy |
| `NoPermission` | Not admin | Use admin account |
| `Frozen` | Asset is frozen | Thaw first |
| `BalanceLow` | Insufficient balance | Check balance |
| `MinBalanceNotMet` | Below min balance | Increase amount |

### Error Messages Mapping

```typescript
// In src/lib/errorMessages.ts
export function getAssetErrorMessage(errorName: string): string {
  const messages: Record<string, string> = {
    'Unknown': 'Asset does not exist',
    'InUse': 'Asset is currently in use and cannot be destroyed',
    'NoPermission': 'You do not have permission to perform this operation',
    'Frozen': 'Asset is frozen. Please thaw it first.',
    'BalanceLow': 'Insufficient balance for this operation',
    'MinBalanceNotMet': 'Amount is below the minimum balance requirement'
  }

  return messages[errorName] || 'Asset operation failed'
}
```

## Testing

### Fee Estimation

```typescript
const fee = await createAssetBatch(api, params, address).getEstimatedFees(address)
console.log('Estimated fee:', formatUnits(fee, 12), 'QFN')
```

### Dry Run

```typescript
const result = await createAssetBatch(api, params, address).dryRun(address)

if (result.success) {
  console.log('Transaction would succeed')
} else {
  console.error('Would fail:', result.dispatchError)
}
```

## Validation

```typescript
export function validateCreateParams(params: CreateAssetParams): string | null {
  if (!params.name || params.name.length === 0) {
    return 'Token name is required'
  }

  if (!params.symbol || params.symbol.length === 0) {
    return 'Token symbol is required'
  }

  if (!params.decimals || isNaN(parseInt(params.decimals))) {
    return 'Invalid decimals'
  }

  if (parseInt(params.decimals) < 0 || parseInt(params.decimals) > 18) {
    return 'Decimals must be between 0 and 18'
  }

  if (!params.minBalance || parseFloat(params.minBalance) <= 0) {
    return 'Minimum balance must be greater than 0'
  }

  if (params.initialMintAmount && parseFloat(params.initialMintAmount) > 0) {
    if (!params.initialMintBeneficiary || params.initialMintBeneficiary.length !== 48) {
      return 'Valid beneficiary address required for initial mint'
    }
  }

  return null  // Valid
}
```

## Complete File Exports

```typescript
// src/lib/index.ts
export {
  createAssetBatch,
  mintTokens,
  transferTokens,
  destroyAssetBatch,
  type CreateAssetParams,
  type MintParams,
  type TransferParams,
  type DestroyAssetParams
} from './assetOperations'
```

## Summary

This reference implementation demonstrates:
- ✅ Full type safety with TypedApi
- ✅ Correct PAPI patterns (MultiAddress, Binary, batch)
- ✅ Amount conversion (parseUnits/formatUnits)
- ✅ Batch operations with .decodedCall
- ✅ Conditional batching
- ✅ Clear interfaces and documentation
- ✅ Production-ready error handling

Use this as the template for all pallet operations in generated apps.
