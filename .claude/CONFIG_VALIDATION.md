# Configuration Validation Guide

## JSON Schema

The `polkadot-config.json` file must conform to the JSON schema defined in `polkadot-config.schema.json`.

## Validation Rules

### Required Fields

**Minimum valid configuration:**
```json
{
  "version": "1.0",
  "project": {
    "name": "my-app",
    "chain": "qfn"
  },
  "features": {
    "assets": {
      "enabled": true,
      "operations": ["create"]
    }
  }
}
```

### Project Configuration

**name**:
- Must be kebab-case (lowercase, hyphens only)
- 3-50 characters
- Pattern: `^[a-z0-9-]+$`
- Valid: `my-polkadot-app`, `dao-manager`, `token-swap`
- Invalid: `MyApp`, `my_app`, `my app`

**chain**:
- Phase 1: Only `"qfn"` supported
- Future: `"paseo_asset_hub"`, `"dot_asset_hub"`, `"custom"`

**description**:
- Optional string
- Max 200 characters

### Feature Configuration

**At least one feature must be enabled.**

#### Assets Feature

When `enabled: true`:
- `operations` array is required
- At least one operation must be specified
- Valid operations: `create`, `mint`, `transfer`, `destroy`, `freeze`, `thaw`, `burn`
- Operations must be unique

Example:
```json
{
  "assets": {
    "enabled": true,
    "operations": ["create", "mint", "transfer", "destroy"]
  }
}
```

#### Balances Feature

When `enabled: true`:
- `operations` array is required
- At least one operation must be specified
- Valid operations: `transfer`, `transfer_keep_alive`, `transfer_all`

Example:
```json
{
  "balances": {
    "enabled": true,
    "operations": ["transfer", "transfer_keep_alive"]
  }
}
```

#### Governance & Staking

Currently not implemented (Phase 2):
- `enabled` must be `false`
- Including these features will not break validation but will be ignored

### UI Configuration

All UI fields are optional with sensible defaults.

**branding.appName**:
- 3-50 characters
- Will be used in page title and header
- Default: `"Polkadot dApp"`

**branding.logoUrl**:
- Must be valid URI or `null`
- `null` uses default QF Network logo
- Custom logos should be publicly accessible URLs

**theme**:
- `"light"`, `"dark"`, or `"system"`
- Default: `"system"` (follows OS preference)

**layout**:
- `"sidebar"` or `"topnav"`
- Default: `"sidebar"`

### Advanced Configuration

All advanced fields are optional.

**errorHandling**:
- `"basic"`: User-friendly error messages only
- `"verbose"`: Include technical details
- Default: `"basic"`

**queryStaleTime**:
- Integer in milliseconds
- Range: 1000-300000 (1s-5min)
- Default: 30000 (30s)

**toastDuration**:
- Integer in milliseconds
- Range: 5000-60000 (5s-60s)
- Default: 30000 (30s)

**includeDevtools**:
- Boolean
- Default: `true`

## Validation Examples

### ✅ Valid Configurations

**Minimal config:**
```json
{
  "version": "1.0",
  "project": { "name": "my-app", "chain": "qfn" },
  "features": {
    "assets": { "enabled": true, "operations": ["transfer"] }
  }
}
```

**Full config:**
```json
{
  "version": "1.0",
  "project": {
    "name": "dao-manager",
    "chain": "qfn",
    "description": "Manage assets and balances on QF Network"
  },
  "features": {
    "assets": {
      "enabled": true,
      "operations": ["create", "mint", "transfer", "destroy"]
    },
    "balances": {
      "enabled": true,
      "operations": ["transfer", "transfer_keep_alive"]
    }
  },
  "ui": {
    "branding": {
      "appName": "DAO Manager",
      "logoUrl": "https://example.com/logo.png"
    },
    "theme": "dark",
    "layout": "sidebar"
  },
  "advanced": {
    "errorHandling": "verbose",
    "queryStaleTime": 60000,
    "toastDuration": 15000,
    "includeDevtools": false
  }
}
```

### ❌ Invalid Configurations

**Missing required fields:**
```json
{
  "version": "1.0",
  "project": { "name": "my-app" }
  // Missing: project.chain, features
}
```

**Invalid project name:**
```json
{
  "project": { "name": "My App", "chain": "qfn" }
  // Spaces not allowed
}
```

**Feature enabled without operations:**
```json
{
  "features": {
    "assets": { "enabled": true }
    // Missing: operations array
  }
}
```

**Invalid operation:**
```json
{
  "features": {
    "assets": {
      "enabled": true,
      "operations": ["create", "invalid_operation"]
    }
  }
}
```

**No features enabled:**
```json
{
  "features": {
    "assets": { "enabled": false },
    "balances": { "enabled": false }
  }
  // At least one feature must be enabled
}
```

## Validation Workflow

The workflow validates configuration in this order:

1. **Schema validation** - Check JSON structure against schema
2. **Semantic validation** - Check business rules
3. **Pallet availability** - Verify operations exist in chain metadata
4. **Dependency check** - Ensure required operations are included

### Validation Script

Use the provided validation script:

```bash
./scripts/validate-config.sh polkadot-config.json
```

The script checks:
- ✅ File exists
- ✅ Valid JSON syntax
- ✅ Conforms to schema
- ✅ Required fields present
- ✅ At least one feature enabled
- ✅ Feature operations are valid

## Error Messages

Common validation errors and fixes:

**"Missing 'version'"**
- Add: `"version": "1.0"`

**"Invalid project name"**
- Use kebab-case: `my-app` not `My App` or `my_app`

**"Chain 'xyz' not supported"**
- Phase 1 only supports `"qfn"`

**"Feature enabled without operations"**
- Add `"operations": ["transfer"]` or set `"enabled": false`

**"No features enabled"**
- Enable at least one feature: `"assets": { "enabled": true, ... }`

**"Invalid operation 'xyz' for pallet"**
- Check spelling and available operations for that pallet

## Migration Guide

### From Future Versions

When schema version changes, migration guides will be provided here.

**1.0 → 2.0** (Future):
- Multi-chain support added
- Governance features implemented
- Breaking changes: TBD

## Programmatic Validation

### TypeScript Types

```typescript
interface PolkadotConfig {
  version: '1.0'
  project: {
    name: string
    chain: 'qfn'
    description?: string
  }
  features: {
    assets?: {
      enabled: boolean
      operations?: Array<'create' | 'mint' | 'transfer' | 'destroy' | 'freeze' | 'thaw' | 'burn'>
    }
    balances?: {
      enabled: boolean
      operations?: Array<'transfer' | 'transfer_keep_alive' | 'transfer_all'>
    }
  }
  ui?: {
    branding?: {
      appName?: string
      logoUrl?: string | null
    }
    theme?: 'light' | 'dark' | 'system'
    layout?: 'sidebar' | 'topnav'
  }
  advanced?: {
    errorHandling?: 'basic' | 'verbose'
    queryStaleTime?: number
    toastDuration?: number
    includeDevtools?: boolean
  }
}
```

## Best Practices

1. **Start minimal** - Use minimal config first, add features incrementally
2. **Validate early** - Run validation before starting workflow
3. **Use defaults** - Omit optional fields to use sensible defaults
4. **Document changes** - If editing config, document why in commit message
5. **Version control** - Commit config to track changes over time
