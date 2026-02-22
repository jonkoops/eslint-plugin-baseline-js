# Add ESLint v10 support & clean up defensive code

## Summary

Adds ESLint v10 to the supported version range while maintaining backward compatibility with ESLint v8.57+ and v9. Also removes unnecessary defensive code that was guarding against APIs guaranteed to exist since well before our minimum supported version (8.57.0).

## Changes

### ESLint v10 support

**package.json**
- Peer dependency: `>=8.57.0 <10` → `>=8.57.0 <11`
- Dev dependency: `eslint@^9.39.2` → `eslint@^10.0.1`

**src/orchestrator/use-baseline.ts** — delegate context forwarding

ESLint v10 removes several deprecated `context.*` methods (`getSourceCode`, `getCwd`, `getFilename`, `getPhysicalFilename`, `getScope`, `getAncestors`, `getDeclaredVariables`, `markVariableAsUsed`). This plugin creates proxy context objects for delegate rules, so the forwarding logic was updated:

- **Method forwarding** now uses `typeof v === "function"` guards — methods are forwarded only when they still exist on the real context (v8/v9), silently omitted on v10 where they've been removed.
- **Property forwarding** now includes `cwd`, `filename`, and `physicalFilename` — the v9+/v10 property replacements for the deprecated `getCwd()`, `getFilename()`, and `getPhysicalFilename()` methods.
- **`sourceCode` access** simplified from `ctx.getSourceCode?.()` fallback chain to direct `ctx.sourceCode.ast` access — `context.sourceCode` has existed since ESLint 8.40.0 (PR [#17107](https://github.com/eslint/eslint/pull/17107)), 17 minor versions before our 8.57.0 floor.

**src/orchestrator/use-baseline.ts** — feature-usage descriptor context

Removed a `getSourceCode()` fallback that populated `delegateCtx.sourceCode` when missing. With `sourceCode` already forwarded as a property (guaranteed present since 8.40.0), this code was dead.

**.github/workflows/ci.yml**
- Test matrix now runs against both ESLint v9 and v10 (`eslint-version: [9, 10]`) with `fail-fast: false`.
- Typecheck step conditioned to `eslint-version == 10` (types match the latest major).

**README.md**
- Updated: "ESLint >= 8.57 (Flat Config) — tested on ESLint v9 and v10"

---

### Defensive code cleanup

With the minimum supported version at **8.57.0**, several defensive patterns were guarding against APIs that have existed for 17+ minor versions. These guards added unnecessary complexity and made the code harder to follow.

#### Key ESLint version facts

| API | Introduced | Min floor gap |
|-----|-----------|---------------|
| `context.sourceCode` | 8.40.0 ([#17107](https://github.com/eslint/eslint/pull/17107)) | 17 minors |
| `sourceCode.getScope()` | 8.37.0 ([#17004](https://github.com/eslint/eslint/pull/17004)) | 20 minors |
| `sourceCode.scopeManager` | v4+ | ~50+ minors |

**src/util/ast.ts** — `isUnboundIdentifier`

- Removed outdated comment ("In ESLint v9, context.getScope() is not available") — misleading since the real reason for using `scopeManager` is broader.
- Changed `(context as unknown as { sourceCode?: … }).sourceCode?.scopeManager` to `context.sourceCode.scopeManager` — eliminates a type cast and optional chaining on a property guaranteed to exist since ESLint 8.40.0.

**src/util/ast.ts** — `isGlobalNotShadowed`

- Removed `sourceCode?.getScope` null guard and early return (`if (!sourceCode?.getScope) return true`). `sourceCode.getScope()` has existed since ESLint 8.37.0, making this branch dead code for all supported versions.
- Changed type assertion from `{ getScope?: … }` to `{ getScope: … }` to reflect the API is always present.

**src/orchestrator/use-baseline.ts** — `CtxLike` type

- Changed `sourceCode?: { parserServices?: … }` to `sourceCode: { parserServices?: … }` (non-optional).
- Changed `cx.sourceCode?.parserServices` to `cx.sourceCode.parserServices` — `sourceCode` is always present.

## Testing

- **108/108 tests pass** across 22 test files
- Build succeeds (tsdown v0.18.4)
- Typecheck clean (`tsc --noEmit`)
- CI matrix validates against both ESLint v9 and v10
