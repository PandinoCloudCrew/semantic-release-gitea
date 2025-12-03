---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
inputDocuments:
  - docs/prd.md
  - docs/adr/001-node22-esm-native-apis.md
  - docs/analysis/brainstorming-session-2025-12-02.md
  - docs/architecture.md
  - docs/index.md
workflowType: 'architecture'
lastStep: 8
status: 'complete'
completedAt: '2025-12-02'
project_name: 'semantic-release-gitea'
user_name: 'Joseybv'
date: '2025-12-02'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Project Context Analysis

### Requirements Overview

**Functional Requirements:**
The PRD defines 28 functional requirements across 6 categories that map directly to the existing module structure:

| Category | FRs | Architectural Component |
|----------|-----|------------------------|
| Authentication & Verification | FR1-FR6 | `lib/verify.js`, `lib/resolve-config.js` |
| Release Management | FR7-FR12 | `lib/publish.js`, `lib/add-channel.js` |
| Asset Management | FR13-FR17 | `lib/publish.js`, `lib/glob-assets.js` |
| Error Handling | FR18-FR21 | `lib/get-error.js`, `lib/definitions/errors.js` |
| Configuration | FR22-FR24 | `lib/resolve-config.js` |
| Integration | FR25-FR28 | `index.js` (exports) |

**Non-Functional Requirements:**
The 18 NFRs drive specific architectural decisions:

| NFR Category | Key Requirements | Architectural Impact |
|--------------|------------------|---------------------|
| Security | NFR1-3: Token handling, HTTPS only | HTTP client must not log tokens |
| Integration | NFR4-7: semantic-release v24+, API v1 | Plugin contract compliance |
| Reliability | NFR8-11: Error messages, fail-fast | Error handling patterns |
| Maintainability | NFR12-15: ESM, native APIs, coverage | Module system, dependency choices |
| Compatibility | NFR16-18: Node 22+, Forgejo/Gitea | Runtime and API compatibility |

**Scale & Complexity:**

- Primary domain: Node.js library / CI tooling
- Complexity level: LOW
- Estimated architectural components: 10 modules (unchanged from legacy)

### Technical Constraints & Dependencies

**Hard Constraints (from ADR-001):**
- Node.js 22+ required (for `fs.glob()`)
- ESM module system (`"type": "module"`)
- semantic-release v24+ plugin contract

**Dependencies Being Removed:**
| Dependency | Native Replacement |
|------------|-------------------|
| `got` | `fetch()` |
| `fs-extra` | `fs/promises` |
| `globby` | `fs.glob()` |
| `aggregate-error` | `AggregateError` |
| `url-join` | `URL` class |
| `querystring` | `URLSearchParams` |

**Dependencies Being Retained:**
- `@semantic-release/error` - semantic-release error contract
- `form-data` - multipart file uploads (native `FormData` file handling unverified)
- `debug` - debug logging (standard pattern)
- `lodash` - may migrate to `lodash-es` or native

### Cross-Cutting Concerns Identified

1. **Environment Variable Precedence**: `FORGEJO_*` must take priority over `GITEA_*` across all config resolution
2. **Error Handling Patterns**: Native `fetch()` doesn't throw on 4xx/5xx - requires wrapper
3. **Async Iterator Handling**: `fs.glob()` returns `AsyncIterator`, not array - requires `Array.fromAsync()`
4. **Token Security**: Must never appear in logs or error messages
5. **API Path Construction**: URL building must handle trailing slashes consistently

## Starter Template Evaluation

### Primary Technology Domain

Node.js library / semantic-release plugin - **existing codebase modernization**

### Starter Template Assessment: Not Applicable

This is a **modernization project** of an existing, working library. The project structure is already established and proven effective:

- **10 modules** with clear separation of concerns
- **~500 LOC** in source, ~700 LOC in tests
- **Working CI/CD** pipeline
- **Established patterns** for plugin hooks, HTTP abstraction, error handling

**Rationale:**
Starter templates are designed for greenfield projects where scaffolding decisions (project structure, tooling, configuration) need to be made. This project already has all these decisions made and validated through 4+ years of production use.

### Modernization Approach Instead

Rather than scaffolding, we will:

1. **Update `package.json`** - Node 22+, `"type": "module"`, updated deps
2. **Convert modules in-place** - `require`→`import`, `module.exports`→`export`
3. **Replace implementations** - Dependencies with native APIs
4. **Migrate tests** - AVA to `node:test`

### Existing Architecture Retained

```
index.js                    # Plugin entry (3 hook exports)
lib/
├── verify.js              # verifyConditions implementation
├── publish.js             # publish implementation
├── add-channel.js         # addChannel implementation
├── get-client.js          # HTTP client abstraction (MAJOR CHANGE: got→fetch)
├── resolve-config.js      # Config resolution (ADD: Forgejo env vars)
├── glob-assets.js         # Asset matching (CHANGE: globby→fs.glob)
├── parse-git-url.js       # URL parsing (minimal change)
├── is-prerelease.js       # Branch detection (minimal change)
├── get-error.js           # Error factory (minimal change)
└── definitions/
    ├── errors.js          # Error definitions (ADD: Forgejo errors)
    └── constants.js       # Constants (minimal change)
```

**Note:** First implementation task is `package.json` update and ESM conversion, not project initialization.

## Core Architectural Decisions

### Decision Priority Analysis

**Critical Decisions (Block Implementation):**
- HTTP client implementation pattern
- Error handling approach for native fetch()
- Test mocking strategy

**Important Decisions (Shape Architecture):**
- Dependency retention/removal choices

**Deferred Decisions (Post-MVP):**
- TypeScript declarations (.d.ts files)
- Enhanced debug logging levels

### HTTP Client Architecture

**Decision:** Response wrapper class with centralized error handling

**Pattern:**
```javascript
class GiteaClient {
  async _makeRequest(method, path, body) {
    const response = await fetch(url, {
      method,
      body: body ? JSON.stringify(body) : undefined,
      headers: { Authorization: `token ${this.token}`, 'Content-Type': 'application/json' }
    });

    if (!response.ok) {
      const error = new Error(`API error: ${response.status}`);
      error.status = response.status;
      error.response = { statusCode: response.status }; // Compatibility shim
      throw error;
    }

    return response.json();
  }
}
```

**Rationale:** Maintains existing class structure, centralizes `!response.ok` detection, preserves try/catch pattern in calling code.

### Dependency Decisions

| Dependency | Decision | Rationale |
|------------|----------|-----------|
| `got` | **Remove** → native `fetch()` | Node 22+ native |
| `fs-extra` | **Remove** → `fs/promises` | Node 14+ native |
| `globby` | **Remove** → `fs.glob()` | Node 22+ native |
| `aggregate-error` | **Remove** → `AggregateError` | Node 15+ native |
| `url-join` | **Remove** → `URL` class | Node 10+ native |
| `querystring` | **Remove** → `URLSearchParams` | Node 10+ native |
| `lodash` | **Remove** → native JS | Minimal usage, native alternatives |
| `form-data` | **Keep** | File upload reliability |
| `debug` | **Keep** | Standard plugin pattern |
| `@semantic-release/error` | **Keep** | Plugin contract requirement |

**Final Dependency Count:** 3 runtime dependencies (down from 9)

### Testing Architecture

**Decision:** nock 14+ for fetch() mocking

**Pattern:**
```javascript
import nock from 'nock';

nock('https://forgejo.example.com')
  .get('/api/v1/repos/owner/repo')
  .reply(200, { permissions: { push: true } });
```

**Rationale:** nock 14+ supports native fetch(), existing test patterns preserved, minimal migration effort.

### Decision Impact Analysis

**Implementation Sequence:**
1. Update `package.json` (Node 22+, ESM, dependency changes)
2. Convert all modules to ESM syntax
3. Implement fetch() wrapper in `get-client.js`
4. Update modules to use native APIs
5. Migrate tests to `node:test` + nock 14+

**Cross-Component Dependencies:**
- All modules depend on `get-client.js` fetch() implementation
- Test migration depends on nock 14+ compatibility verification
- ESM conversion must complete before native API replacements

## Implementation Patterns & Consistency Rules

### Pattern Categories Defined

**Critical Conflict Points Identified:** 5 areas where implementation could vary

### Code Organization Patterns

**ESM Import Structure:**
All files must organize imports in this order:
```javascript
// 1. Node built-ins (with node: prefix)
import { readFile } from 'node:fs/promises';
import { glob } from 'node:fs';

// 2. External packages
import SemanticReleaseError from '@semantic-release/error';
import FormData from 'form-data';
import debug from 'debug';

// 3. Local modules (with .js extension)
import { resolveConfig } from './resolve-config.js';
import { getError } from './get-error.js';
```

**File Extensions:** Always include `.js` extension in local imports (ESM requirement).

### Configuration Patterns

**Environment Variable Precedence:**
Forgejo variables take priority over Gitea variables:
```javascript
const token = env.FORGEJO_TOKEN || env.GITEA_TOKEN || pluginConfig.giteaToken;
const url = env.FORGEJO_URL || env.GITEA_URL || pluginConfig.giteaUrl;
```

**Order:** `FORGEJO_*` → `GITEA_*` → plugin config → default

**Platform Detection:**
```javascript
const platform = env.FORGEJO_TOKEN ? 'Forgejo' : 'Gitea';
```

### Error Handling Patterns

**Error Code Convention:**
Use platform-agnostic error codes with platform-specific messages:

| Code | Meaning |
|------|---------|
| `ENOTOKEN` | Missing authentication token |
| `ENOURL` | Missing server URL |
| `ENOPERMISSION` | Token lacks required permissions |
| `EINVALIDTOKEN` | Token is invalid or expired |
| `EMISSINGREPO` | Repository not found |
| `EAPIERROR` | Generic API error |
| `EINVALIDASSETS` | Invalid assets configuration |

**Error Message Pattern:**
```javascript
// Include detected platform in message
getError('ENOPERMISSION', { platform });
// → "The Forgejo token doesn't have push access to owner/repo"
```

### Native API Patterns

**URL Construction:**
Use native `URL` constructor with normalized base:
```javascript
// In resolve-config.js - normalize base URL once
const normalizedUrl = giteaUrl.endsWith('/') ? giteaUrl : `${giteaUrl}/`;

// In get-client.js - construct API URLs
const url = new URL(`repos/${owner}/${repo}`, this.baseUrl);
```

**Async Iterator Handling:**
Use `Array.fromAsync()` for `fs.glob()` results:
```javascript
// Convert async iterator to array
const files = await Array.fromAsync(
  glob(patterns, { cwd, withFileTypes: false })
);
```

### fetch() Patterns

**Request Pattern:**
```javascript
const response = await fetch(url, {
  method,
  body: body ? JSON.stringify(body) : undefined,
  headers: {
    'Authorization': `token ${this.token}`,
    'Content-Type': 'application/json',
  },
});
```

**Error Detection Pattern:**
```javascript
if (!response.ok) {
  const error = new Error(`API error: ${response.status}`);
  error.status = response.status;
  error.response = { statusCode: response.status };
  throw error;
}
```

### Enforcement Guidelines

**All AI Agents MUST:**
- Follow ESM import grouping order
- Check `FORGEJO_*` before `GITEA_*` environment variables
- Use generic error codes with platform-specific messages
- Use `Array.fromAsync()` for glob results
- Include `.js` extension in all local imports
- Never log or expose tokens in error messages

**Anti-Patterns to Avoid:**
```javascript
// ❌ Wrong import order
import { getError } from './get-error.js';
import { readFile } from 'node:fs/promises';

// ❌ Wrong env var precedence
const token = env.GITEA_TOKEN || env.FORGEJO_TOKEN;

// ❌ Missing .js extension
import { resolveConfig } from './resolve-config';

// ❌ Platform-specific error codes
throw getError('EFORGEJOPERMISSION');
```

## Project Structure & Boundaries

### Complete Project Directory Structure

```
semantic-release-gitea/
├── index.js                      # Plugin entry point (exports 3 hooks)
├── package.json                  # Node 22+, "type": "module", dependencies
├── package-lock.json
├── .nvmrc                        # Node version specification
├── .gitignore
├── .xo-config.json               # XO linting configuration (existing)
├── README.md                     # Forgejo-first documentation
├── CHANGELOG.md                  # Auto-generated by semantic-release
├── LICENSE                       # MIT license
│
├── lib/
│   ├── verify.js                 # verifyConditions hook implementation
│   ├── publish.js                # publish hook implementation
│   ├── add-channel.js            # addChannel hook implementation
│   ├── get-client.js             # GiteaClient class (fetch-based)
│   ├── resolve-config.js         # Config/env resolution (Forgejo-first)
│   ├── glob-assets.js            # Asset file matching (fs.glob)
│   ├── parse-git-url.js          # Repository URL parsing
│   ├── is-prerelease.js          # Branch prerelease detection
│   ├── get-error.js              # Error factory function
│   └── definitions/
│       ├── errors.js             # Error code definitions
│       └── constants.js          # Shared constants
│
├── test/
│   ├── verify.test.js            # verifyConditions tests
│   ├── publish.test.js           # publish tests
│   ├── add-channel.test.js       # addChannel tests
│   ├── get-client.test.js        # GiteaClient tests
│   ├── resolve-config.test.js    # Config resolution tests
│   ├── glob-assets.test.js       # Asset globbing tests
│   ├── parse-git-url.test.js     # URL parsing tests
│   ├── is-prerelease.test.js     # Prerelease detection tests
│   ├── integration.test.js       # End-to-end plugin tests
│   └── helpers/
│       ├── mock-server.js        # nock setup helpers
│       └── fixtures.js           # Test data fixtures
│
├── .github/
│   └── workflows/
│       ├── test.yml              # CI: test on Node 22, 24
│       └── release.yml           # CD: semantic-release publish
│
└── docs/                         # Project documentation (BMAD artifacts)
    ├── prd.md
    ├── architecture-decision.md
    └── ...
```

### Architectural Boundaries

**Plugin Interface Boundary:**
```
semantic-release (caller)
        │
        ▼
   ┌─────────────────────────────────────────┐
   │              index.js                    │
   │  exports: verifyConditions, publish,     │
   │           addChannel                     │
   └─────────────────────────────────────────┘
        │
        ▼
   lib/ modules (internal implementation)
```

**HTTP Client Boundary:**
```
lib/verify.js ──┐
lib/publish.js ─┼──► lib/get-client.js ──► Forgejo/Gitea API
lib/add-channel.js ─┘      (GiteaClient)
```

**Configuration Boundary:**
```
Environment Variables ─┐
                       ├──► lib/resolve-config.js ──► Normalized config
Plugin Options ────────┘
```

### Requirements to Structure Mapping

**FR Category: Authentication & Verification (FR1-6)**
- Primary: `lib/verify.js`
- Config: `lib/resolve-config.js`
- HTTP: `lib/get-client.js`
- Tests: `test/verify.test.js`, `test/resolve-config.test.js`

**FR Category: Release Management (FR7-12)**
- Primary: `lib/publish.js`, `lib/add-channel.js`
- Support: `lib/is-prerelease.js`
- Tests: `test/publish.test.js`, `test/add-channel.test.js`

**FR Category: Asset Management (FR13-17)**
- Primary: `lib/glob-assets.js`
- Integration: `lib/publish.js` (uploads)
- Tests: `test/glob-assets.test.js`

**FR Category: Error Handling (FR18-21)**
- Primary: `lib/get-error.js`
- Definitions: `lib/definitions/errors.js`
- Tests: All test files verify error scenarios

**FR Category: Configuration (FR22-24)**
- Primary: `lib/resolve-config.js`
- Support: `lib/parse-git-url.js`
- Tests: `test/resolve-config.test.js`, `test/parse-git-url.test.js`

**FR Category: Integration (FR25-28)**
- Primary: `index.js`
- Tests: `test/integration.test.js`

### Integration Points

**External Integration: Forgejo/Gitea API**
- Entry: `lib/get-client.js`
- Endpoints used:
  - `GET /repos/{owner}/{repo}` - verify permissions
  - `POST /repos/{owner}/{repo}/releases` - create release
  - `PATCH /repos/{owner}/{repo}/releases/{id}` - update release
  - `POST /repos/{owner}/{repo}/releases/{id}/assets` - upload assets

**External Integration: semantic-release**
- Entry: `index.js`
- Contract: Plugin hooks receive `(pluginConfig, context)`
- Context provides: `env`, `options`, `nextRelease`, `branch`, `logger`

### File Modification Summary

**Files with Major Changes:**
| File | Change Type | Details |
|------|-------------|---------|
| `package.json` | Major | Node 22+, ESM, new deps |
| `lib/get-client.js` | Major | got → fetch() |
| `lib/glob-assets.js` | Major | globby → fs.glob() |
| `lib/resolve-config.js` | Moderate | Add Forgejo env vars |
| `lib/definitions/errors.js` | Moderate | Generic error codes |

**Files with Minor Changes (ESM conversion only):**
| File | Change |
|------|--------|
| `index.js` | `require` → `import`, `module.exports` → `export` |
| `lib/verify.js` | ESM syntax |
| `lib/publish.js` | ESM syntax + AggregateError |
| `lib/add-channel.js` | ESM syntax |
| `lib/parse-git-url.js` | ESM syntax |
| `lib/is-prerelease.js` | ESM syntax |
| `lib/get-error.js` | ESM syntax |
| `lib/definitions/constants.js` | ESM syntax |

**Test Files (migrate to node:test):**
All files in `test/` migrate from AVA to `node:test`

## Architecture Validation Results

### Coherence Validation ✅

**Decision Compatibility:**
All technology choices work together without conflicts:
- Node.js 22+ provides fetch(), fs.glob(), AggregateError natively
- ESM module system aligns with semantic-release v24+ plugin contract
- nock 14+ supports native fetch() for testing
- Retained dependencies (form-data, debug) are ESM-compatible

**Pattern Consistency:**
Implementation patterns support all architectural decisions:
- ESM import grouping applied consistently
- Forgejo-first precedence centralized in resolve-config.js
- Error handling pattern centralized in get-client.js
- Naming conventions are clear and enforceable

**Structure Alignment:**
Project structure supports the architecture:
- Existing proven structure retained
- Clear boundaries between plugin interface, modules, and HTTP client
- All FR categories mapped to specific files

### Requirements Coverage Validation ✅

**Functional Requirements Coverage:**
All 28 FRs have explicit architectural support:
- Authentication (FR1-6): verify.js, resolve-config.js, get-client.js
- Release Management (FR7-12): publish.js, add-channel.js
- Asset Management (FR13-17): glob-assets.js, publish.js
- Error Handling (FR18-21): get-error.js, definitions/errors.js
- Configuration (FR22-24): resolve-config.js, parse-git-url.js
- Integration (FR25-28): index.js

**Non-Functional Requirements Coverage:**
All 18 NFRs are architecturally addressed:
- Security (NFR1-3): Token handling, HTTPS enforcement
- Integration (NFR4-7): Plugin contract compliance
- Reliability (NFR8-11): Centralized error handling
- Maintainability (NFR12-15): ESM, native APIs, testing
- Compatibility (NFR16-18): Node 22+, platform support

### Implementation Readiness Validation ✅

**Decision Completeness:**
- All critical decisions documented with code examples
- Technology versions specified (Node 22+, nock 14+)
- Dependency decisions explicit with rationale

**Structure Completeness:**
- Complete directory tree defined
- All files listed with change classification
- Integration points mapped

**Pattern Completeness:**
- 5 implementation patterns defined
- Anti-patterns documented
- Enforcement guidelines provided

### Gap Analysis Results

**Critical Gaps:** None identified

**Nice-to-Have (Post-MVP):**
- TypeScript declarations (.d.ts)
- Enhanced debug logging levels

### Architecture Completeness Checklist

**✅ Requirements Analysis**
- [x] Project context thoroughly analyzed
- [x] Scale and complexity assessed (LOW)
- [x] Technical constraints identified (Node 22+, ESM, semantic-release v24+)
- [x] Cross-cutting concerns mapped (error handling, config resolution)

**✅ Architectural Decisions**
- [x] Critical decisions documented with versions
- [x] Technology stack fully specified
- [x] Integration patterns defined (fetch wrapper, plugin contract)
- [x] Dependency decisions documented

**✅ Implementation Patterns**
- [x] ESM import conventions established
- [x] Error handling patterns defined
- [x] Configuration patterns specified
- [x] Native API usage patterns documented

**✅ Project Structure**
- [x] Complete directory structure defined
- [x] Component boundaries established
- [x] Requirements to structure mapping complete
- [x] File modification summary provided

### Architecture Readiness Assessment

**Overall Status:** READY FOR IMPLEMENTATION

**Confidence Level:** HIGH
- This is a modernization of a proven, working architecture
- Changes are well-defined and scoped
- Risk analysis completed in brainstorming session

**Key Strengths:**
- Existing architecture is proven (4+ years production use)
- Clear scope: dependency replacement, not redesign
- Comprehensive pattern documentation for consistency
- All decisions traceable to PRD requirements

**Areas for Future Enhancement:**
- TypeScript declarations for better DX
- Enhanced debug logging for troubleshooting

### Implementation Handoff

**AI Agent Guidelines:**
- Follow all architectural decisions exactly as documented
- Use implementation patterns consistently across all modules
- Check Forgejo env vars before Gitea env vars
- Use generic error codes with platform-specific messages
- Include .js extension in all ESM imports

**First Implementation Priority:**
1. Update `package.json` (Node 22+, `"type": "module"`, dependency changes)
2. Convert `index.js` and all `lib/` files to ESM syntax
3. Implement fetch() wrapper in `get-client.js`
4. Update remaining modules to use native APIs
5. Migrate tests to `node:test` + nock 14+

## Architecture Completion Summary

### Workflow Completion

**Architecture Decision Workflow:** COMPLETED ✅
**Total Steps Completed:** 8
**Date Completed:** 2025-12-02
**Document Location:** docs/architecture-decision.md

### Final Architecture Deliverables

**📋 Complete Architecture Document**
- All architectural decisions documented with specific versions
- Implementation patterns ensuring AI agent consistency
- Complete project structure with all files and directories
- Requirements to architecture mapping
- Validation confirming coherence and completeness

**🏗️ Implementation Ready Foundation**
- 6 core architectural decisions made (HTTP client, error handling, dependencies, testing, patterns)
- 5 implementation patterns defined (imports, env vars, errors, URLs, async iterators)
- 10 modules specified with change classification
- 28 functional + 18 non-functional requirements fully supported

**📚 AI Agent Implementation Guide**
- Technology stack: Node.js 22+, ESM, native APIs
- Consistency rules that prevent implementation conflicts
- Project structure with clear boundaries
- Integration patterns (fetch wrapper, plugin contract)

### Implementation Handoff

**For AI Agents:**
This architecture document is your complete guide for implementing semantic-release-gitea modernization. Follow all decisions, patterns, and structures exactly as documented.

**Development Sequence:**
1. Update package.json (Node 22+, ESM, deps)
2. Convert all modules to ESM syntax
3. Implement fetch() wrapper in get-client.js
4. Replace remaining dependencies with native APIs
5. Migrate tests to node:test + nock 14+

### Quality Assurance Checklist

**✅ Architecture Coherence**
- [x] All decisions work together without conflicts
- [x] Technology choices are compatible
- [x] Patterns support the architectural decisions
- [x] Structure aligns with all choices

**✅ Requirements Coverage**
- [x] All 28 functional requirements are supported
- [x] All 18 non-functional requirements are addressed
- [x] Cross-cutting concerns are handled
- [x] Integration points are defined

**✅ Implementation Readiness**
- [x] Decisions are specific and actionable
- [x] Patterns prevent agent conflicts
- [x] Structure is complete and unambiguous
- [x] Examples are provided for clarity

### Project Success Factors

**🎯 Clear Decision Framework**
Every technology choice was made collaboratively with clear rationale, ensuring all stakeholders understand the architectural direction.

**🔧 Consistency Guarantee**
Implementation patterns and rules ensure that multiple AI agents will produce compatible, consistent code that works together seamlessly.

**📋 Complete Coverage**
All project requirements are architecturally supported, with clear mapping from business needs to technical implementation.

**🏗️ Solid Foundation**
The existing proven architecture is retained with focused modernization, following current best practices.

---

**Architecture Status:** READY FOR IMPLEMENTATION ✅

**Next Phase:** Begin implementation using the architectural decisions and patterns documented herein.

**Document Maintenance:** Update this architecture when major technical decisions are made during implementation.
