# Dynamic Prompt Templates

These templates are used to generate context-specific prompts based on the `polkadot-config.json` configuration.

## Base Prompt Template

```markdown
# Generate Polkadot dApp: {{project.name}}

## Configuration Summary
- **Project Name**: {{project.name}}
- **Chain**: {{project.chain}} (QF Network testnet)
- **App Name**: {{ui.branding.appName}}
- **Enabled Features**: {{features.list}}

## Task Overview

Generate a complete Polkadot dApp based on the configuration. The template already includes:
- ✅ Wallet connection (WalletContext, useWallet hook)
- ✅ Chain connection (ConnectionContext, useConnectionStatus hook)
- ✅ Transaction management (TransactionContext, useTransaction hook)
- ✅ Error handling (errorParsing.ts, transactionErrors.ts, error boundaries)
- ✅ TanStack Query setup (queryClient.ts, queryHelpers.ts)
- ✅ Toast notifications (Sonner integration, useTransactionToasts)
- ✅ UI components (Radix UI wrappers, buttons, cards, inputs)
- ✅ Utility functions (formatUnits, parseUnits, cn, formatFee)

Your job: Generate feature-specific code for selected operations.

## Instructions

### 1. Load Skills
- Load `.claude/skills/polkadot-api-patterns.md`
- Load `.claude/skills/pallet-operations.md`

### 2. Load Reference Documents
{{#if features.assets.enabled}}
- Load `.claude/docs/asset-operations-reference.md`
{{/if}}
{{#if features.balances.enabled}}
- Load balance operations patterns from pallet-operations skill
{{/if}}

### 3. Validate Configuration
```bash
./scripts/validate-config.sh polkadot-config.json
```

### 4. Generate Code

{{#if features.assets.enabled}}
#### Assets Pallet

**Operations to implement**: {{features.assets.operations}}

**A. Create `src/lib/assetOperations.ts`**

Using patterns from `asset-operations-reference.md`, create:

{{#each features.assets.operations}}
- {{this}} operation function and types
{{/each}}

Ensure:
- Use `TypedApi<typeof qfn>` for type safety
- All addresses wrapped in `MultiAddress.Id()`
- All strings converted with `Binary.fromText()`
- Amounts use `parseUnits()` for conversion
- Batch operations use `.decodedCall` and `TxCallData[]`

**B. Create Components**

{{#each features.assets.operations}}
{{#if (eq this "create")}}
Create `src/components/CreateAsset.tsx`:
- Form with fields: assetId, name, symbol, decimals, minBalance, initialMintAmount, initialMintBeneficiary
- Use `useAssetMutation` hook with createAssetBatch operation
- Include FeeDisplay, TransactionReview, MutationError components
- Wrap in FeatureErrorBoundary
{{/if}}
{{#if (eq this "mint")}}
Create `src/components/MintTokens.tsx`:
- Form with fields: assetId, recipient, amount
- Use `useAssetMutation` hook with mintTokens operation
- Include FeeDisplay, MutationError components
- Wrap in FeatureErrorBoundary
{{/if}}
{{#if (eq this "transfer")}}
Create `src/components/TransferTokens.tsx`:
- Form with fields: assetId, recipient, amount
- Use `useAssetMutation` hook with transferTokens operation
- Include FeeDisplay, MutationError components
- Wrap in FeatureErrorBoundary
{{/if}}
{{#if (eq this "destroy")}}
Create `src/components/DestroyAsset.tsx`:
- Form with field: assetId
- Warning about permanent destruction
- Use `useAssetMutation` hook with destroyAssetBatch operation
- Include FeeDisplay, MutationError components
- Wrap in FeatureErrorBoundary
{{/if}}
{{/each}}

**C. Add Toast Configurations**

In `src/lib/toastConfigs.ts`, add:
{{#each features.assets.operations}}
- `{{this}}TokensToasts`: signing, broadcasting, inBlock, finalized, error messages
{{/each}}

**D. Update Error Messages**

In `src/lib/errorMessages.ts`, add Assets pallet errors:
- Unknown: "Asset does not exist"
- InUse: "Asset is currently in use"
- NoPermission: "You do not have permission"
- Frozen: "Asset is frozen"
- BalanceLow: "Insufficient balance"
- MinBalanceNotMet: "Amount below minimum balance"

**E. Create Query Hooks**

Create `src/hooks/useNextAssetId.ts` (if "create" in operations)
Create `src/hooks/useAssetMetadata.ts`
Create `src/hooks/useAssetBalance.ts`

{{/if}}

{{#if features.balances.enabled}}
#### Balances Pallet

**Operations to implement**: {{features.balances.operations}}

**A. Create `src/lib/balanceOperations.ts`**

Implement:
{{#each features.balances.operations}}
- {{this}} operation function and types
{{/each}}

**B. Create Components**

{{#each features.balances.operations}}
{{#if (eq this "transfer")}}
Create `src/components/TransferNative.tsx`:
- Form with fields: recipient, amount
- Use native chain decimals (12 for QFN)
- Use `useAssetMutation` hook with transferNative operation
{{/if}}
{{/each}}

**C. Create Query Hooks**

Create `src/hooks/useNativeBalance.ts`

{{/if}}

### 5. Update Application Structure

**A. Update `src/App.tsx`**

Add navigation items for generated components:

```typescript
const navigationItems = [
  // Existing items...
  {{#if features.assets.enabled}}
  {{#each features.assets.operations}}
  {{#if (eq this "create")}}
  {
    id: 'create' as const,
    label: 'Create Asset',
    icon: Plus,
    component: CreateAsset,
    section: 'main',
  },
  {{/if}}
  {{#if (eq this "mint")}}
  {
    id: 'mint' as const,
    label: 'Mint Tokens',
    icon: Coins,
    component: MintTokens,
    section: 'operations',
  },
  {{/if}}
  {{#if (eq this "transfer")}}
  {
    id: 'transfer' as const,
    label: 'Transfer',
    icon: Send,
    component: TransferTokens,
    section: 'operations',
  },
  {{/if}}
  {{#if (eq this "destroy")}}
  {
    id: 'destroy' as const,
    label: 'Destroy Asset',
    icon: Trash2,
    component: DestroyAsset,
    section: 'admin',
  },
  {{/if}}
  {{/each}}
  {{/if}}
]
```

**B. Update Exports**

- `src/components/index.ts`: Export all new components
- `src/hooks/index.ts`: Export all new hooks
- `src/lib/index.ts`: Export all operation functions and types

**C. Update Branding**

- `package.json`: Set `name` to "{{project.name}}"
- `index.html`: Set `<title>` to "{{ui.branding.appName}}"
{{#if ui.branding.appName}}
- `src/App.tsx`: Update header text to "{{ui.branding.appName}}"
{{/if}}

### 6. Validation

**A. Type Check**
```bash
pnpm typecheck
```
Fix any TypeScript errors.

**B. Test Connection**
```bash
./scripts/test-connection.sh
```

**C. Verify Build**
```bash
pnpm build
```

### 7. Success Criteria

- ✅ All operation functions created
- ✅ All components created with proper validation
- ✅ Toast configs added
- ✅ Error messages added
- ✅ Query hooks created
- ✅ Navigation updated
- ✅ TypeScript compiles with no errors
- ✅ Connection test passes

## Critical Rules

### PAPI Patterns (from skills)
- NEVER use `@polkadot/api` - only `polkadot-api`
- NEVER use `any` type
- ALWAYS use `MultiAddress.Id()` for addresses
- ALWAYS use `Binary.fromText()` for strings
- ALWAYS use `parseUnits()/formatUnits()` for amounts
- ALWAYS use `.decodedCall` for batch operations
- ALWAYS type batch arrays as `TxCallData[]`
- ALWAYS pass batch as `{ calls }` not `calls`

### Architecture (from CLAUDE.md)
- Components are presentational
- Business logic in `lib/` functions
- Server state via TanStack Query only
- Use `useState` (not `useReducer`)
- Pure functions, immutability, early returns

Generate all code now.
```

## Template Variables

### Project Variables
- `{{project.name}}` - Project name from config
- `{{project.chain}}` - Chain identifier (always "qfn" in Phase 1)
- `{{project.description}}` - Optional project description

### Feature Variables
- `{{features.list}}` - Comma-separated list of enabled features
- `{{features.assets.enabled}}` - Boolean
- `{{features.assets.operations}}` - Array of operation names
- `{{features.balances.enabled}}` - Boolean
- `{{features.balances.operations}}` - Array of operation names

### UI Variables
- `{{ui.branding.appName}}` - Custom app name
- `{{ui.branding.logoUrl}}` - Custom logo URL (if provided)
- `{{ui.theme}}` - Theme preference
- `{{ui.layout}}` - Layout preference

### Advanced Variables
- `{{advanced.errorHandling}}` - Error verbosity level
- `{{advanced.queryStaleTime}}` - TanStack Query stale time
- `{{advanced.toastDuration}}` - Toast notification duration
- `{{advanced.includeDevtools}}` - Include devtools boolean

## Conditional Blocks

### Feature-Specific Blocks

```handlebars
{{#if features.assets.enabled}}
  Generate assets code...
{{/if}}

{{#if features.balances.enabled}}
  Generate balances code...
{{/if}}
```

### Operation-Specific Blocks

```handlebars
{{#each features.assets.operations}}
  {{#if (eq this "create")}}
    Generate CreateAsset component...
  {{/if}}
  {{#if (eq this "mint")}}
    Generate MintTokens component...
  {{/if}}
{{/each}}
```

## Helper Functions

### List Formatter
```javascript
features.list = Object.entries(config.features)
  .filter(([_, f]) => f.enabled)
  .map(([name, _]) => name)
  .join(', ')
```

### Operation Checker
```javascript
hasOperation(feature, operation) {
  return config.features[feature]?.operations?.includes(operation)
}
```

## Example Substitution

**Input Config:**
```json
{
  "project": { "name": "my-dao-app", "chain": "qfn" },
  "features": {
    "assets": { "enabled": true, "operations": ["create", "mint", "transfer"] }
  },
  "ui": { "branding": { "appName": "DAO Manager" } }
}
```

**Output Prompt:**
```markdown
# Generate Polkadot dApp: my-dao-app

## Configuration Summary
- **Project Name**: my-dao-app
- **Chain**: qfn (QF Network testnet)
- **App Name**: DAO Manager
- **Enabled Features**: assets

...

#### Assets Pallet

**Operations to implement**: create, mint, transfer

**A. Create `src/lib/assetOperations.ts`**

Using patterns from `asset-operations-reference.md`, create:

- create operation function and types
- mint operation function and types
- transfer operation function and types

...
```

## Usage in Workflow

1. Read `polkadot-config.json`
2. Parse and validate configuration
3. Substitute variables into template
4. Pass generated prompt to Claude Code
5. Claude Code executes instructions

## Notes

- Template uses Handlebars-style syntax for clarity
- Actual implementation can use any templating engine
- Keep prompts concise but comprehensive
- Include all necessary context
- Reference skills and documents by path
