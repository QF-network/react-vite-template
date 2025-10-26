# Skill: Pallet Operations

## Description

Implement blockchain operations for specific pallets (Assets, Balances, etc.) following established patterns. This skill teaches how to construct extrinsic calls correctly for different Substrate pallets.

## Core Principles

1. **Operation functions are pure** - No side effects, return transaction observables
2. **Parameters are typed** - Define interfaces for each operation
3. **Amounts use BigInt** - Never use Number for token amounts
4. **Batch when logical** - Group related operations atomically
5. **Follow naming conventions** - `palletNameOperation` (e.g., `mintTokens`, `transferAsset`)

## Assets Pallet Operations

### Create Asset

**Purpose:** Create a new fungible token (asset class)

**Pattern:**
```typescript
export interface CreateAssetParams {
  assetId: string
  minBalance: string  // In display units
  name: string
  symbol: string
  decimals: string
  initialMintAmount?: string  // Optional
  initialMintBeneficiary?: string  // Required if initialMintAmount > 0
}

export const createAssetBatch = (
  api: QfnApi,
  params: CreateAssetParams,
  signerAddress: string
) => {
  const assetId = parseInt(params.assetId)
  const minBalance = BigInt(params.minBalance) * 10n ** BigInt(params.decimals)

  // Create the asset
  const createCall = api.tx.Assets.create({
    id: assetId,
    admin: MultiAddress.Id(signerAddress),
    min_balance: minBalance
  }).decodedCall

  // Set metadata
  const metadataCall = api.tx.Assets.set_metadata({
    id: assetId,
    name: Binary.fromText(params.name),
    symbol: Binary.fromText(params.symbol),
    decimals: parseInt(params.decimals)
  }).decodedCall

  const calls: TxCallData[] = [createCall, metadataCall]

  // Optional: Initial mint
  if (params.initialMintAmount && parseFloat(params.initialMintAmount) > 0) {
    const mintAmount = parseUnits(params.initialMintAmount, parseInt(params.decimals))

    const mintCall = api.tx.Assets.mint({
      id: assetId,
      beneficiary: MultiAddress.Id(params.initialMintBeneficiary!),
      amount: mintAmount
    }).decodedCall

    calls.push(mintCall)
  }

  return api.tx.Utility.batch_all({ calls })
}
```

**Why Batch:** Asset creation requires both creating and setting metadata. Batching ensures atomicity.

**Key Points:**
- Parse strings to appropriate types
- minBalance is in smallest units: `minBalance * 10^decimals`
- Admin is usually the signer
- Metadata requires Binary conversion
- Optional initial mint added conditionally

### Mint Tokens

**Purpose:** Create new tokens for an existing asset

**Pattern:**
```typescript
export interface MintParams {
  assetId: string
  recipient: string
  amount: string  // In display units
  decimals: number
}

export const mintTokens = (api: QfnApi, params: MintParams) => {
  const assetId = parseInt(params.assetId)
  const amount = parseUnits(params.amount, params.decimals)

  return api.tx.Assets.mint({
    id: assetId,
    beneficiary: MultiAddress.Id(params.recipient),
    amount
  })
}
```

**Requirements:**
- Signer must be asset admin/issuer
- Recipient must exist (or have sufficient ED)
- Asset must not be frozen

**Key Points:**
- Single operation (no batch needed)
- Convert display amount to chain units
- Use MultiAddress for beneficiary

### Transfer Tokens

**Purpose:** Transfer asset tokens between accounts

**Pattern:**
```typescript
export interface TransferParams {
  assetId: string
  recipient: string
  amount: string  // In display units
  decimals: number
}

export const transferTokens = (api: QfnApi, params: TransferParams) => {
  const assetId = parseInt(params.assetId)
  const amount = parseUnits(params.amount, params.decimals)

  return api.tx.Assets.transfer({
    id: assetId,
    target: MultiAddress.Id(params.recipient),
    amount
  })
}
```

**Requirements:**
- Sender must have sufficient balance
- Recipient must meet minimum balance requirement
- Asset must not be frozen

**Key Points:**
- Parameter is `target` not `beneficiary`
- Sender is implicit (signer)
- Amount must be >= asset's minBalance

### Destroy Asset

**Purpose:** Permanently remove an asset and all its state

**Pattern:**
```typescript
export interface DestroyAssetParams {
  assetId: string
}

export const destroyAssetBatch = (api: QfnApi, params: DestroyAssetParams) => {
  const assetId = parseInt(params.assetId)

  // Step 1: Freeze asset (no new operations)
  const freezeCall = api.tx.Assets.freeze_asset({
    id: assetId
  }).decodedCall

  // Step 2: Start destruction process
  const startDestroyCall = api.tx.Assets.start_destroy({
    id: assetId
  }).decodedCall

  // Step 3: Remove all approvals
  const destroyApprovalsCall = api.tx.Assets.destroy_approvals({
    id: assetId
  }).decodedCall

  // Step 4: Remove all account balances
  const destroyAccountsCall = api.tx.Assets.destroy_accounts({
    id: assetId
  }).decodedCall

  // Step 5: Finish and remove asset
  const finishDestroyCall = api.tx.Assets.finish_destroy({
    id: assetId
  }).decodedCall

  // Ordered batch - must execute in sequence
  const calls: TxCallData[] = [
    freezeCall,
    startDestroyCall,
    destroyApprovalsCall,
    destroyAccountsCall,
    finishDestroyCall
  ]

  return api.tx.Utility.batch_all({ calls })
}
```

**Requirements:**
- Signer must be asset admin
- Asset must exist
- All accounts will lose their balances

**Why Batch:** Destruction is multi-step. Must execute all or none.

**Key Points:**
- Order matters! Cannot change sequence
- All holders lose balances (permanent!)
- Approvals removed before accounts
- `batch_all` ensures atomicity

### Freeze Asset

**Purpose:** Prevent all transfers temporarily

**Pattern:**
```typescript
export interface FreezeAssetParams {
  assetId: string
}

export const freezeAsset = (api: QfnApi, params: FreezeAssetParams) => {
  const assetId = parseInt(params.assetId)

  return api.tx.Assets.freeze_asset({
    id: assetId
  })
}
```

**Effect:** All transfers disabled until thawed

### Thaw Asset

**Purpose:** Re-enable transfers after freeze

**Pattern:**
```typescript
export interface ThawAssetParams {
  assetId: string
}

export const thawAsset = (api: QfnApi, params: ThawAssetParams) => {
  const assetId = parseInt(params.assetId)

  return api.tx.Assets.thaw_asset({
    id: assetId
  })
}
```

### Burn Tokens

**Purpose:** Permanently destroy tokens from an account

**Pattern:**
```typescript
export interface BurnParams {
  assetId: string
  amount: string
  decimals: number
}

export const burnTokens = (api: QfnApi, params: BurnParams) => {
  const assetId = parseInt(params.assetId)
  const amount = parseUnits(params.amount, params.decimals)

  return api.tx.Assets.burn({
    id: assetId,
    who: MultiAddress.Id(signerAddress),  // Burns from this account
    amount
  })
}
```

**Difference from Destroy:**
- Burn: Reduce supply, asset continues to exist
- Destroy: Remove entire asset and all balances

## Balances Pallet Operations

### Transfer Native Token

**Purpose:** Transfer chain's native token (e.g., QFN, DOT)

**Pattern:**
```typescript
export interface NativeTransferParams {
  recipient: string
  amount: string  // In display units
  decimals: number  // Chain's native decimals (usually 12)
}

export const transferNative = (api: QfnApi, params: NativeTransferParams) => {
  const amount = parseUnits(params.amount, params.decimals)

  return api.tx.Balances.transfer_keep_alive({
    dest: MultiAddress.Id(params.recipient),
    value: amount
  })
}
```

**Why `transfer_keep_alive`:** Prevents sender from going below existential deposit

**Alternative: `transfer`**
```typescript
// Allows draining account completely
return api.tx.Balances.transfer({
  dest: MultiAddress.Id(params.recipient),
  value: amount
})
```

**When to use each:**
- `transfer_keep_alive`: Default (safer, prevents account reaping)
- `transfer`: When intentionally closing account
- `transfer_all`: Send entire balance minus fee

### Transfer All

**Purpose:** Send maximum possible amount

**Pattern:**
```typescript
export const transferAll = (api: QfnApi, recipient: string) => {
  return api.tx.Balances.transfer_all({
    dest: MultiAddress.Id(recipient),
    keep_alive: false  // Close sender account
  })
}
```

**Use case:** Migrating to new account, closing old account

## Query Patterns for Operations

### Check Asset Exists

```typescript
export function useAssetExists(assetId: number) {
  const { api } = useConnectionContext()

  return useQuery({
    queryKey: ['assetExists', assetId],
    queryFn: async () => {
      const metadata = await api.query.Assets.Metadata.getValue(assetId)
      return metadata !== undefined
    }
  })
}
```

### Get Asset Balance

```typescript
export function useAssetBalance(assetId: number, address: string) {
  const { api } = useConnectionContext()

  return useQuery({
    queryKey: ['assetBalance', assetId, address],
    queryFn: async () => {
      const account = await api.query.Assets.Account.getValue(
        assetId,
        MultiAddress.Id(address)
      )
      return account?.balance ?? 0n
    }
  })
}
```

### Get Native Balance

```typescript
export function useNativeBalance(address: string) {
  const { api } = useConnectionContext()

  return useQuery({
    queryKey: ['nativeBalance', address],
    queryFn: async () => {
      const account = await api.query.System.Account.getValue(address)
      return account.data.free
    }
  })
}
```

### List All Assets

```typescript
export function useAllAssets() {
  const { api } = useConnectionContext()

  return useQuery({
    queryKey: ['allAssets'],
    queryFn: async () => {
      const entries = await api.query.Assets.Metadata.getEntries()

      return entries.map(([key, value]) => ({
        id: key.args[0],
        name: Binary.asText(value.name),
        symbol: Binary.asText(value.symbol),
        decimals: value.decimals
      }))
    },
    staleTime: 60_000  // Assets don't change frequently
  })
}
```

## Operation Validation

### Pre-Flight Checks

Always validate before submitting:

```typescript
export function validateMintParams(params: MintParams): string | null {
  // Check asset ID
  if (!params.assetId || isNaN(parseInt(params.assetId))) {
    return 'Invalid asset ID'
  }

  // Check amount
  if (!params.amount || parseFloat(params.amount) <= 0) {
    return 'Amount must be greater than 0'
  }

  // Check recipient
  if (!params.recipient || params.recipient.length !== 48) {
    return 'Invalid recipient address'
  }

  return null  // Valid
}
```

### Using in Component

```typescript
const mutation = useMutation({
  mutationFn: async (params: MintParams) => {
    // Validate first
    const error = validateMintParams(params)
    if (error) throw new Error(error)

    // Execute transaction
    const tx = mintTokens(api, params)
    await executeTransaction('mint', tx, params)
  }
})
```

## Error Handling

### Common Asset Errors

```typescript
export const assetErrorMessages: Record<string, string> = {
  'Unknown': 'Asset does not exist',
  'InUse': 'Asset is currently in use',
  'NoPermission': 'You do not have permission for this operation',
  'Frozen': 'Asset is frozen',
  'BalanceLow': 'Insufficient balance',
  'MinBalanceNotMet': 'Amount is below minimum balance requirement'
}
```

### Handling in UI

```typescript
if (error.pallet === 'Assets') {
  const message = assetErrorMessages[error.errorName] || 'Asset operation failed'
  toast.error(message)
}
```

## Toast Configurations

### Pattern for Each Operation

```typescript
export const mintTokensToasts: ToastConfig<MintParams> = {
  signing: 'Please sign the mint transaction in your wallet',
  broadcasting: () => 'Submitting mint transaction...',
  inBlock: 'Mint transaction included in block...',
  finalized: (details) => {
    return details
      ? `${details.amount} tokens minted to ${details.recipient.slice(0, 8)}... for Asset ID ${details.assetId}!`
      : 'Tokens minted successfully!'
  },
  error: (_error: string) => `Mint transaction failed!`
}
```

**Customize for each operation:**
- Signing: Tell user what they're signing
- Broadcasting: Indicate submission
- InBlock: Transaction accepted
- Finalized: Success with details
- Error: Clear failure message

## Component Integration

### Standard Form Component Pattern

```typescript
export function MintTokens() {
  const { selectedAccount } = useWalletContext()
  const { api } = useConnectionContext()
  const [formData, setFormData] = useState<MintParams>({
    assetId: '',
    recipient: '',
    amount: '',
    decimals: 12
  })

  const { mutation, transaction } = useAssetMutation<MintParams>({
    params: formData,
    operationFn: (params) => mintTokens(api, params),
    toastConfig: mintTokensToasts,
    transactionKey: 'mint',
    isValid: (params) =>
      params.assetId !== '' &&
      params.recipient !== '' &&
      params.amount !== '' &&
      parseFloat(params.amount) > 0,
    onSuccess: async () => {
      await invalidateAssetQueries(queryClient)
      setFormData(initialState)
    }
  })

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault()
    mutation.mutate()
  }

  return (
    <form onSubmit={handleSubmit}>
      {/* Form fields */}
      <FeeDisplay transaction={transaction} />
      <Button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Minting...' : 'Mint Tokens'}
      </Button>
      <MutationError error={mutation.error} />
    </form>
  )
}
```

## Testing Operations

### Fee Estimation Hook

```typescript
export function useFee(transaction: Transaction | null, signerAddress?: string) {
  const { api } = useConnectionContext()

  return useQuery({
    queryKey: ['fee', transaction?.observable, signerAddress],
    queryFn: async () => {
      if (!transaction?.observable || !signerAddress) return null

      const fee = await transaction.observable.getEstimatedFees(signerAddress)
      return fee
    },
    enabled: !!transaction && !!signerAddress
  })
}
```

### Dry Run Before Submit

```typescript
const { data: dryRunResult } = useQuery({
  queryKey: ['dryRun', params],
  queryFn: async () => {
    const tx = mintTokens(api, params)
    return await tx.dryRun(selectedAccount.address)
  },
  enabled: isFormValid
})

// Show warning if dry run fails
{dryRunResult && !dryRunResult.success && (
  <Alert>Transaction would fail: {dryRunResult.dispatchError}</Alert>
)}
```

## Checklist for New Operations

When implementing a new pallet operation:

- [ ] Define typed parameter interface
- [ ] Create operation function returning observable
- [ ] Use parseUnits for amount parameters
- [ ] Use MultiAddress.Id for address parameters
- [ ] Use Binary.fromText for string parameters
- [ ] Use batch if multiple extrinsics needed
- [ ] Create toast configuration
- [ ] Add validation function
- [ ] Create query hook if needed
- [ ] Add error messages for pallet errors
- [ ] Create form component
- [ ] Test fee estimation
- [ ] Test dry run
- [ ] Handle success (invalidate queries)
- [ ] Export from lib/index.ts

## Common Patterns Summary

| Pattern | When to Use | Example |
|---------|-------------|---------|
| Single extrinsic | Simple operation | mint, transfer |
| Batch | Multiple related ops | create (+ metadata + mint) |
| parseUnits | User input → chain | Amount fields |
| formatUnits | Chain → display | Balance display |
| MultiAddress.Id | Address params | beneficiary, target, admin |
| Binary.fromText | String metadata | name, symbol |
| .decodedCall | Batch operations | Getting call data |
| TxCallData[] | Batch array type | calls parameter |

## Resources

- Assets pallet docs: https://docs.substrate.io/reference/frame-pallets/#assets
- Balances pallet docs: https://docs.substrate.io/reference/frame-pallets/#balances
- Template examples: See `src/lib/assetOperations.ts`
- PAPI patterns skill: For core PAPI usage
