# Setup Command - Generate Custom Polkadot dApp

Read `polkadot-config.json` and generate a custom Polkadot dApp based on the configuration.

## Workflow Overview

You are generating a custom Polkadot dApp from the template. The template already includes all infrastructure (wallet connection, chain connection, transaction management, error handling). Your job is to generate feature-specific code based on user selections.

## Step 1: Load Configuration

Read and parse `polkadot-config.json`:

```typescript
{
  "version": "1.0",
  "project": { "name": string, "chain": "qfn", "description": string },
  "features": {
    "assets": { "enabled": boolean, "operations": string[] },
    "balances": { "enabled": boolean, "operations": string[] }
  },
  "ui": { "branding": { "appName": string }, "theme": string, "layout": string },
  "advanced": { ... }
}
```

## Step 2: Validate Configuration

Run validation script:
```bash
./scripts/validate-config.sh polkadot-config.json
```

If validation fails, report errors to user and stop.

## Step 3: Load Skills

Load these skills for code generation:
- `.claude/skills/polkadot-api-patterns.md` - Core PAPI patterns
- `.claude/skills/pallet-operations.md` - Pallet-specific patterns

## Step 4: Load Reference Documents

Based on enabled features, load:
- If `assets.enabled`: `.claude/docs/asset-operations-reference.md`

## Step 5: Generate Feature Code

For each enabled feature, generate the following files:

### For Assets Feature

If `features.assets.enabled === true`:

**A. Create `src/lib/assetOperations.ts`**
- Import patterns from `asset-operations-reference.md`
- Include only operations listed in `features.assets.operations`
- Generate type interfaces for each operation
- Example operations: create, mint, transfer, destroy, freeze, thaw, burn

**B. Create Components**
For each operation in `features.assets.operations`, create corresponding component:
- `src/components/CreateAsset.tsx` (if "create" in operations)
- `src/components/MintTokens.tsx` (if "mint" in operations)
- `src/components/TransferTokens.tsx` (if "transfer" in operations)
- `src/components/DestroyAsset.tsx` (if "destroy" in operations)

Each component should:
- Use `useAssetMutation` hook for transaction execution
- Include form with proper validation
- Use `FeeDisplay` component
- Use `MutationError` component
- Wrap in `FeatureErrorBoundary`

**C. Create Toast Configurations**
Add to `src/lib/toastConfigs.ts`:
- Toast configs for each operation (signing, broadcasting, inBlock, finalized, error)

**D. Add Error Messages**
Update `src/lib/errorMessages.ts`:
- Add Assets pallet error messages

**E. Create Query Hooks (if needed)**
- `src/hooks/useAssetMetadata.ts`
- `src/hooks/useAssetBalance.ts`
- `src/hooks/useNextAssetId.ts`

### For Balances Feature

If `features.balances.enabled === true`:

**A. Create `src/lib/balanceOperations.ts`**
- Implement operations from `features.balances.operations`
- Operations: transfer, transfer_keep_alive, transfer_all

**B. Create Components**
- `src/components/TransferNative.tsx` (if "transfer" in operations)

**C. Add Toast Configurations**
- Balance operation toasts

**D. Add Query Hooks**
- `src/hooks/useNativeBalance.ts`

## Step 6: Update Application Structure

**A. Update `src/App.tsx`**
- Add navigation items for all generated components
- Update navigationItems array with:
  - id, label, icon (from lucide-react), component, section

Example:
```typescript
const navigationItems = [
  {
    id: 'assets' as const,
    label: 'Portfolio',
    icon: LayoutDashboard,
    component: AssetList,
    section: 'main'
  },
  {
    id: 'create' as const,
    label: 'Create Asset',
    icon: Plus,
    component: CreateAsset,
    section: 'main'
  },
  // Add generated features here...
]
```

**B. Update `src/components/index.ts`**
- Export all new components

**C. Update `src/hooks/index.ts`**
- Export all new hooks

**D. Update `src/lib/index.ts`**
- Export all new operation functions and types

## Step 7: Update Branding

**A. Update `package.json`**
- Change `name` field to `project.name` from config

**B. Update `index.html`**
- Change `<title>` to `ui.branding.appName` from config

**C. Update App Title**
In `src/App.tsx`, if `ui.branding.appName` specified:
- Update the `<h1>` text to use custom app name

## Step 8: Validation

**A. Type Check**
```bash
pnpm typecheck
```
Fix any TypeScript errors that arise.

**B. Test Connection**
```bash
./scripts/test-connection.sh
```
Verify chain connection works.

**C. Ensure Metadata is Generated**
```bash
pnpm install
```
This triggers `postinstall: papi` to generate descriptors.

## Step 9: Report to User

Provide a summary of what was generated:

```markdown
## ✅ dApp Generated Successfully!

### Configuration
- **Project**: {project.name}
- **Chain**: QF Network testnet
- **App Name**: {ui.branding.appName}

### Generated Features

#### Assets Pallet
- Operations: {comma-separated operations}
- Components created:
  - CreateAsset.tsx (if applicable)
  - MintTokens.tsx (if applicable)
  - TransferTokens.tsx (if applicable)
  - DestroyAsset.tsx (if applicable)
- Operation functions: src/lib/assetOperations.ts
- Query hooks: src/hooks/useAssetMetadata.ts, etc.

#### Balances Pallet (if enabled)
- Operations: {comma-separated operations}
- Components: TransferNative.tsx
- Operation functions: src/lib/balanceOperations.ts

### Next Steps

1. **Start development server:**
   ```bash
   pnpm dev
   ```

2. **Install wallet extension:**
   - Polkadot.js Extension
   - Talisman
   - SubWallet

3. **Connect and test:**
   - Open http://localhost:5173
   - Connect your wallet
   - Try the generated features!

### Files Modified
- src/App.tsx (navigation)
- package.json (name)
- index.html (title)

### Files Created
- src/lib/assetOperations.ts
- src/components/CreateAsset.tsx
- ... (list all created files)

### TypeScript Status
✅ All types valid
```

## Critical Rules

### Code Quality
- **NEVER use `any` type** - Use `unknown` and narrow
- **NEVER use `@polkadot/api`** - Only `polkadot-api`
- **NEVER use type assertions** - Let types infer
- **Always use MultiAddress.Id()** for addresses
- **Always use Binary.fromText()** for strings
- **Always use parseUnits/formatUnits** for amounts
- **Always use `.decodedCall`** for batch operations
- **Always type batch calls** as `TxCallData[]`

### Architecture
- Components are presentational (minimal logic)
- Business logic in `lib/` functions
- Server state via TanStack Query only
- Use `useState` (never `useReducer`)

### Conventions (from CLAUDE.md)
- Follow all conventions in CLAUDE.md
- 30s toast duration
- 30s TanStack Query stale time
- Use error boundaries at feature level

## Error Handling

If generation fails:
1. Report the specific error
2. Show which file/step failed
3. Suggest fix if possible
4. DO NOT leave partial generation (clean up)

## Testing

Before reporting success:
- ✅ All files created
- ✅ TypeScript compiles cleanly
- ✅ No ESLint errors
- ✅ Chain connection tested
- ✅ Metadata generated

## Notes

- Template already has wallet, connection, transaction contexts
- Only generate feature-specific code
- Follow existing patterns from template
- Use reference documents as templates
- Adapt patterns to selected operations
