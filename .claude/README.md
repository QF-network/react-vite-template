# Claude Code Workflow Assets

This directory contains all assets for generating custom Polkadot dApps using Claude Code.

## Quick Start

**Users:** Run this command to generate your custom dApp:
```bash
/setup
```

**Developers:** See below for asset documentation.

## Directory Structure

```
.claude/
├── README.md (this file)              # Index of all assets
├── polkadot-config.schema.json        # JSON Schema for configuration
├── CONFIG_VALIDATION.md               # Validation guide and examples
├── PROMPT_TEMPLATES.md                # Dynamic prompt generation
├── skills/                            # Reusable knowledge
│   ├── polkadot-api-patterns.md      # Core PAPI patterns (10 patterns)
│   └── pallet-operations.md          # Pallet-specific operations
├── docs/                              # Reference implementations
│   └── asset-operations-reference.md # Complete Assets pallet code
└── commands/                          # Slash commands
    └── setup.md                      # Main workflow command
```

## Asset Descriptions

### Configuration Files

#### `polkadot-config.schema.json`
- **Purpose:** JSON Schema v7 specification for configuration validation
- **Used by:** Validation script, wizard, documentation
- **Format:** JSON Schema Draft 07
- **Validates:** Project structure, features, UI, advanced options

#### `CONFIG_VALIDATION.md`
- **Purpose:** Human-readable validation guide
- **Used by:** Developers, users debugging config issues
- **Contains:** Examples of valid/invalid configs, error messages, TypeScript types
- **Format:** Markdown

#### `PROMPT_TEMPLATES.md`
- **Purpose:** Templates for generating dynamic prompts
- **Used by:** Workflow orchestration system
- **Contains:** Handlebars-style templates, variable substitution, conditional logic
- **Format:** Markdown with code blocks

### Skills

Skills are loaded by Claude Code to provide reusable knowledge across all generations.

#### `skills/polkadot-api-patterns.md`
- **Purpose:** Teach core polkadot-api usage patterns
- **Token cost:** ~3,500 tokens (loaded once per session)
- **Contains:**
  - 10 critical PAPI patterns with examples
  - Client initialization
  - Type handling (MultiAddress, Binary)
  - Transaction observables
  - Query patterns with TanStack
  - Token amount conversion
  - Error handling
  - Anti-patterns to avoid
  - Checklist for new features

#### `skills/pallet-operations.md`
- **Purpose:** Teach pallet-specific operation implementation
- **Token cost:** ~2,500 tokens (loaded once per session)
- **Contains:**
  - Assets pallet operations (7 operations)
  - Balances pallet operations (3 operations)
  - Query patterns for each pallet
  - Validation patterns
  - Error handling
  - Toast configurations
  - Component integration
  - Testing patterns

### Reference Documents

Reference documents provide complete, working implementations to copy/adapt.

#### `docs/asset-operations-reference.md`
- **Purpose:** Complete reference implementation of Assets pallet
- **Token cost:** ~3,000 tokens (loaded when assets feature enabled)
- **Contains:**
  - Full `src/lib/assetOperations.ts` with annotations
  - Type definitions for all operations
  - Pattern explanations
  - Usage examples
  - Error handling
  - Validation logic
  - Testing code

**When to add more docs:**
- Create similar docs for other pallets (Balances, Governance, Staking)
- Follow same structure: types → operations → explanations → usage

### Commands

#### `commands/setup.md`
- **Purpose:** Main workflow orchestration for dApp generation
- **Triggered by:** User typing `/setup` in Claude Code
- **Token cost:** ~500 tokens (workflow instructions)
- **Contains:**
  - Step-by-step workflow
  - Configuration loading
  - Skill/document loading
  - Feature generation logic
  - Validation steps
  - Success reporting
  - Error handling
  - Critical rules and conventions

## Workflow Execution Flow

1. **User completes wizard** (separate web app)
   - Wizard generates `polkadot-config.json`
   - Wizard provides `npx degit` command

2. **User clones template**
   ```bash
   npx degit user/polkadot-dapp-template my-app
   cd my-app
   ```

3. **User opens in Claude Code**
   ```bash
   claude-code .
   ```

4. **User runs setup command**
   ```bash
   /setup
   ```

5. **Claude Code executes workflow**
   - Loads `commands/setup.md`
   - Reads `polkadot-config.json`
   - Validates with `scripts/validate-config.sh`
   - Loads skills from `skills/`
   - Loads reference docs from `docs/`
   - Generates feature code
   - Updates application structure
   - Validates TypeScript
   - Tests connection
   - Reports success

6. **User has working dApp**
   ```bash
   pnpm dev
   ```

## Token Cost Estimates

### Per Workflow Run
- Configuration: ~500 tokens
- Skills (first load): ~5,000 tokens
- Reference documents: ~3,000 tokens
- Workflow instructions: ~500 tokens
- Context: ~2,000 tokens
- Generation output: ~15,000 tokens
- **Total: ~26,000 tokens** (~$0.41)

### Optimized Run (with caching)
- Configuration: ~500 tokens
- Skills (cached): 0 tokens
- Reference documents: ~3,000 tokens
- Workflow instructions: ~500 tokens
- Context: ~2,000 tokens
- Generation output: ~15,000 tokens
- **Total: ~21,000 tokens** (~$0.33)

## Adding New Features

### To Add a New Pallet

1. **Update JSON Schema**
   - Add pallet to `features` in `polkadot-config.schema.json`
   - Define valid operations enum

2. **Document Patterns in Skill**
   - Add operations to `skills/pallet-operations.md`
   - Include type patterns, validation, queries

3. **Create Reference Document**
   - Create `docs/{pallet}-operations-reference.md`
   - Follow structure of `asset-operations-reference.md`
   - Include complete implementation with annotations

4. **Update Slash Command**
   - Add generation logic to `commands/setup.md`
   - Define component structure
   - Specify navigation updates

5. **Update Prompt Templates**
   - Add conditional blocks to `PROMPT_TEMPLATES.md`
   - Define variable substitution

### Example: Adding Governance Pallet

```markdown
# In polkadot-config.schema.json
"governance": {
  "type": "object",
  "properties": {
    "enabled": { "type": "boolean" },
    "operations": {
      "type": "array",
      "items": {
        "enum": ["propose", "vote", "delegate"]
      }
    }
  }
}

# In skills/pallet-operations.md
## Governance Pallet Operations
### Propose
### Vote
### Delegate

# Create docs/governance-operations-reference.md
# ... complete implementation

# In commands/setup.md
{{#if features.governance.enabled}}
#### Governance Pallet
... generation logic
{{/if}}
```

## Validation and Testing

### Validate Configuration
```bash
# Using validation script (uses jq, commonly pre-installed)
./scripts/validate-config.sh polkadot-config.json
```

### Test Connection
```bash
./scripts/test-connection.sh
```

### Test Complete Workflow
```bash
# In Claude Code
/setup

# Then verify
pnpm typecheck
pnpm build
pnpm dev
```

## Troubleshooting

### Common Issues

**"Config validation failed"**
- Check JSON syntax with `jq empty polkadot-config.json`
- Verify required fields present
- Check operation names match valid operations
- Ensure at least one feature enabled

**"Skill not loaded"**
- Check file path: `.claude/skills/{name}.md`
- Verify file is readable
- Check Claude Code can access .claude directory

**"Generation failed"**
- Check TypeScript errors: `pnpm typecheck`
- Verify all imports exist
- Check for syntax errors in generated code
- Review error message in Claude Code output

**"Connection test failed"**
- Check internet connection
- Verify chain URL in `.papi/polkadot-api.json`
- Test manually: `curl https://test.qfnetwork.xyz`
- Install wscat: `npm install -g wscat`

## Resources

- **Phase 1 Architecture:** `../WORKFLOW_ARCHITECTURE.md`
- **Phase 2 Summary:** `../PHASE2_SUMMARY.md`
- **Template Extraction:** `../TEMPLATE_EXTRACTION_PROMPT.md`
- **Initial Prompt:** `../INITIAL_PROMPT.md`
- **PAPI Documentation:** https://papi.how
- **Substrate Docs:** https://docs.substrate.io

## Version

**Assets Version:** 1.0.0 (Phase 2)
**Compatible with:** polkadot-api v1.20.0+
**Schema Version:** 1.0

## Updates

When updating workflow assets:
1. Update version number in this README
2. Document breaking changes
3. Update PHASE2_SUMMARY.md
4. Test complete workflow
5. Update examples if needed

## Support

For issues or questions:
1. Check troubleshooting section above
2. Review Phase 1/2 documentation
3. Check PAPI docs at https://papi.how
4. Open issue in template repository

---

**Last Updated:** October 26, 2025
**Maintained by:** Polkadot dApp Template Team
