# Polkadot dApp Template

A production-ready template for building Polkadot dApps with polkadot-api, React 19, and TypeScript.

## Features

✨ **Pre-built Infrastructure**
- Wallet connection (Polkadot.js, Talisman, SubWallet support)
- Chain connection with auto-reconnect
- Transaction lifecycle management
- Comprehensive error handling
- Toast notifications
- TanStack Query integration

🎨 **Modern Stack**
- React 19 + TypeScript
- Vite build system
- Tailwind CSS + Radix UI
- polkadot-api (type-safe blockchain interactions)

📦 **Included Example**
- Full asset management implementation (create, mint, transfer, destroy)
- Ready to customize or extend

## Quick Start

### 1. Clone Template

```bash
npx degit user/polkadot-dapp-template my-polkadot-app
cd my-polkadot-app
```

### 2. Install Dependencies

```bash
pnpm install
```

This will automatically:
- Install all dependencies
- Generate type descriptors via `papi` (postinstall hook)

### 3. Start Development Server

```bash
pnpm dev
```

Open [http://localhost:5173](http://localhost:5173) in your browser.

### 4. Connect Wallet

Install a Polkadot wallet extension:
- [Polkadot.js Extension](https://polkadot.js.org/extension/)
- [Talisman](https://www.talisman.xyz/)
- [SubWallet](https://subwallet.app/)

## Customization with Claude Code

This template is designed to work with the Polkadot dApp workflow generator:

### Option 1: Use Wizard (Recommended)

1. Complete the wizard at [wizard-url] to generate your config
2. Run the provided `npx degit` command
3. Open in Claude Code: `claude-code .`
4. Run: `/setup`

The workflow will customize the template based on your selections.

### Option 2: Manual Customization

1. Clone this template
2. Create `polkadot-config.json` (see `polkadot-config.example.json`)
3. Open in Claude Code
4. Run: `/setup`

## Project Structure

```
.
├── src/
│   ├── components/       # React components
│   │   ├── ui/          # Reusable UI components (buttons, cards, etc.)
│   │   └── ...          # Feature components
│   ├── contexts/        # React Context providers
│   │   ├── WalletContext.tsx
│   │   ├── ConnectionContext.tsx
│   │   └── TransactionContext.tsx
│   ├── hooks/           # Custom React hooks
│   │   ├── useWallet.ts
│   │   ├── useTransaction.ts
│   │   └── ...
│   ├── lib/             # Business logic & utilities
│   │   ├── assetOperations.ts
│   │   ├── errorParsing.ts
│   │   └── utils.ts
│   ├── App.tsx          # Main application
│   └── main.tsx         # Entry point
├── .papi/               # polkadot-api configuration
│   ├── polkadot-api.json
│   └── metadata/
├── .claude/             # Claude Code workflow assets
│   ├── commands/
│   └── docs/
├── public/              # Static assets
├── CLAUDE.md            # Project documentation for AI assistants
└── package.json
```

## Configuration

### Chain Configuration

Edit `.papi/polkadot-api.json` to connect to different chains:

```json
{
  "version": 0,
  "descriptorPath": ".papi/descriptors",
  "entries": {
    "qfn": {
      "wsUrl": "wss://test.qfnetwork.xyz",
      "metadata": ".papi/metadata/qfn.scale"
    }
  }
}
```

### Add New Chain

```bash
# From WSS URL
pnpm papi add mychain -w wss://my-chain-url

# From well-known chain
pnpm papi add dot -n polkadot
```

Then update `src/hooks/useConnectionStatus.ts` to use the new descriptor.

## Included Features

The template includes a complete asset management implementation:

- **Create Asset** - Create new fungible tokens
- **Mint Tokens** - Mint tokens to addresses
- **Transfer Tokens** - Transfer tokens between accounts
- **Destroy Asset** - Permanently destroy assets
- **Portfolio View** - View all assets and balances

## Extending the Template

### Add New Pallet Operations

1. Create operation functions in `src/lib/operations.ts`
2. Create component in `src/components/YourFeature.tsx`
3. Add toast config in `src/lib/toastConfigs.ts`
4. Update navigation in `src/App.tsx`

See `src/lib/assetOperations.ts` for reference patterns.

### Add Query Hooks

1. Create hook in `src/hooks/useYourQuery.ts`
2. Use TanStack Query with polkadot-api observables
3. Export from `src/hooks/index.ts`

See `src/hooks/useNextAssetId.ts` for reference.

## Key Patterns

### Transaction Flow

```typescript
import { useTransaction } from '@/hooks'

const { executeTransaction } = useTransaction(toastConfig)

// Execute transaction
await executeTransaction('operationType', observable, params)
```

### Query Pattern

```typescript
import { useQuery } from '@tanstack/react-query'
import { useConnectionContext } from '@/hooks'

const { api } = useConnectionContext()

const query = useQuery({
  queryKey: ['key', param],
  queryFn: async () => await api.query.Pallet.Storage.getValue(param),
  staleTime: 30_000,
})
```

### Error Handling

All errors are automatically parsed and displayed with user-friendly messages. See `src/lib/errorParsing.ts` for implementation.

## Scripts

- `pnpm dev` - Start development server
- `pnpm build` - Build for production
- `pnpm preview` - Preview production build
- `pnpm lint` - Lint and fix code
- `pnpm lint:check` - Check linting without fixing
- `pnpm typecheck` - Type check without emitting
- `pnpm format` - Format code with Prettier

## Documentation

- **CLAUDE.md** - Comprehensive project documentation following best practices
- **polkadot-api** - [Official docs](https://papi.how)
- **React 19** - [React docs](https://react.dev)

## Architecture Decisions

See `CLAUDE.md` for detailed architecture documentation, including:
- State management patterns
- Transaction lifecycle
- Error handling strategy
- Query configuration
- TypeScript conventions

## License

MIT

## Contributing

This is a template repository. For issues or improvements, please visit [repo-url].
