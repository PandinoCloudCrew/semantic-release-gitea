# ADR-001: Target Node.js 22+ with ESM and Native APIs

## Status

**Accepted** - 2025-12-02

## Context

This project is a fork of `@saithodev/semantic-release-gitea` (v2.1.0, last updated April 2021) that we are modernizing to:

1. Support current Node.js LTS versions
2. Prioritize Forgejo compatibility (Gitea compatibility as nice-to-have)
3. Publish under the `@pcc` npm scope
4. Reduce dependency footprint

The original project supports Node.js 10/11/12/14+, uses CommonJS, and relies on several external dependencies for functionality now available natively in Node.js.

### Current Dependencies Under Review

| Dependency | Purpose | Current Version |
|------------|---------|-----------------|
| `got` | HTTP client | 10.x |
| `fs-extra` | File system utilities | 8.x |
| `globby` | Glob pattern matching | 10.x |
| `aggregate-error` | Multi-error aggregation | 3.x |
| `url-join` | URL path joining | 4.x |
| `querystring` | Query string parsing | 0.2.x |

### ESM Migration Concern

Several modern versions of these dependencies (globby 14.x, aggregate-error 5.x, got 14.x) are ESM-only, which initially raised concerns about the migration complexity and change surface.

## Decision

**Target Node.js 22+ and migrate to ESM with native APIs.**

### Node.js Version: 22+

- Node.js 22 "Jod" is in Active LTS (until October 2025, then Maintenance until April 2027)
- Node.js 24 "Krypton" became Active LTS in October 2025
- Node.js 22+ provides all native APIs needed to eliminate external dependencies

### Module System: ESM

- Adopt `"type": "module"` in package.json
- Use ES module syntax (`import`/`export`) throughout
- Aligns with modern JavaScript ecosystem direction

### Native API Replacements

| Dependency | Replacement | Node.js Version |
|------------|-------------|-----------------|
| `got` | Native `fetch()` | 18+ |
| `fs-extra` | Native `fs/promises` | 14+ |
| `globby` | Native `fs.glob()` | 22+ |
| `aggregate-error` | Native `AggregateError` | 15+ |
| `url-join` | Native `URL` class | 10+ |
| `querystring` | Native `URLSearchParams` | 10+ |

## Rationale

### Why Node.js 22+ (not 20 or 24)?

- **Node.js 20**: Would require keeping `globby` dependency (no native `fs.glob()`)
- **Node.js 22**: Sweet spot - has all native APIs, widely adopted, LTS until 2027
- **Node.js 24**: Too new (LTS since October 2025), many CI systems haven't upgraded

### Why ESM now?

Initially we considered a conservative approach (stay CommonJS, update only compatible deps). However, the decision to require Node.js 22+ changes the calculus:

1. **Native APIs eliminate the problem**: The ESM-only dependencies (globby, aggregate-error) are being replaced with native APIs, not upgraded
2. **Cleaner codebase**: Starting fresh with ESM and native APIs is simpler than maintaining legacy patterns
3. **Future-proof**: ESM is the standard direction for the JavaScript ecosystem
4. **Minimal change surface**: Replacing deps with native APIs is actually a smaller change than upgrading deps

### Why prioritize Forgejo?

- Primary deployment target is PCC's Forgejo instance
- Forgejo API is compatible with Gitea API (same endpoints)
- Support both `FORGEJO_TOKEN`/`FORGEJO_URL` and `GITEA_TOKEN`/`GITEA_URL` environment variables

## Consequences

### Positive

- **6 fewer dependencies**: Reduced supply chain risk and maintenance burden
- **Smaller package size**: No bundled HTTP client or glob library
- **Modern codebase**: Easier for contributors familiar with modern JavaScript
- **Native performance**: Node.js native implementations are optimized
- **Simpler testing**: Node.js built-in test runner (`node:test`) + c8 for coverage

### Negative

- **Breaking change**: Users on Node.js < 22 cannot use v3.0.0
- **ESM-only**: Users with CommonJS-only setups need to adapt
- **Learning curve**: Contributors unfamiliar with ESM syntax

### Neutral

- **semantic-release compatibility**: v24 supports both ESM and CJS plugins, no issue
- **Package rename**: `@saithodev/semantic-release-gitea` → `@pcc/semantic-release-gitea` (planned regardless)

## Implementation Plan

### Phase 1: Foundation
- Update `package.json`: Node 22+, `"type": "module"`, new dependencies
- Convert entry point and all lib files to ESM syntax

### Phase 2: Native APIs
- Replace `got` with native `fetch()`
- Replace `fs-extra` with `fs/promises`
- Replace `globby` with `fs.glob()`
- Replace `aggregate-error` with native `AggregateError`
- Replace `url-join` with `URL` class
- Replace `querystring` with `URLSearchParams`

### Phase 3: Forgejo Support
- Add `FORGEJO_TOKEN` / `FORGEJO_URL` environment variable support
- Document Forgejo as primary target, Gitea as compatible

### Phase 4: Testing
- Migrate tests to `node:test`
- Use `c8` for coverage (replaces `nyc`)
- Update test mocks for native `fetch`

### Phase 5: CI/CD & Documentation
- Update GitHub Actions workflow
- Update README with new requirements
- Publish to npm as `@pcc/semantic-release-gitea`

## References

- [Node.js Releases](https://nodejs.org/en/about/previous-releases)
- [Node.js 22 fs.glob() documentation](https://nodejs.org/api/fs.html#fspromisesglobpattern-options)
- [Forgejo API Documentation](https://code.ops.pcc.fyi/swagger.v1.json)
- [Original semantic-release-gitea](https://github.com/saitho/semantic-release-gitea)
