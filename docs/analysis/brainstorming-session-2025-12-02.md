# Brainstorming Session: semantic-release-gitea Modernization

**Date:** 2025-12-02
**Facilitator:** Mary (Business Analyst)
**Participant:** Joseybv
**Project:** semantic-release-gitea fork modernization

---

## Session Overview

**Topic:** Complete modernization of the semantic-release-gitea fork - transforming an abandoned library into a modern, Forgejo-first release automation tool.

**Goals:**
- Identify gaps in the existing modernization plan
- Optimize phase sequencing
- Assess technical risks and mitigation strategies

**Input Document:** `semantic-release-gitea-modernization.md`

---

## Part 1: Gap Analysis

### Decisions Made

| Area | Decision | Rationale |
|------|----------|-----------|
| **Backward Compatibility** | Not required | Fresh fork, spiritual successor - not a drop-in replacement |
| **Platform Priority** | Forgejo first, Gitea nice-to-have | Primary deployment target is PCC's Forgejo instance |
| **Edge Cases** | Skip special handling | Library already works; minimize change surface |
| **DX Improvements** | Consider minimally | TypeScript types, debug logging - keep scope small |
| **Security Features** | Defer to future | Not in scope for v3.0.0 |
| **Ecosystem Plugins** | Explore later | Interesting but not MVP |

### Gaps Identified

1. **Forgejo API spec** - Downloaded and saved to `docs/forgejo-api-v1.json`
2. **TypeScript declarations** - Low priority, small effort for DX win (future)
3. **Migration guide** - Not needed (no backward compatibility requirement)

---

## Part 2: Prioritization & Sequencing

### Original Plan Analysis

The original plan had 6 sequential phases. Analysis revealed:

- Phases were logically ordered but didn't account for "fail fast" principles
- Forgejo support was Phase 3, despite being the primary target
- ESM migration was bundled with all other changes

### Key Question: Is ESM Required?

**Initial Concern:** Large change surface with ESM migration

**Analysis:**
- Native `fetch` works in CommonJS
- `fs/promises` works in CommonJS
- BUT: Some dependencies (globby 14.x, aggregate-error 5.x) are ESM-only

### Breakthrough: Dependency Purpose Analysis

| Dependency | Purpose | Native Replacement | Node Version |
|------------|---------|-------------------|--------------|
| `aggregate-error` | Multi-error throwing | `AggregateError` (global) | 15+ |
| `globby` | Glob pattern matching | `fs.glob()` | 22+ |
| `got` | HTTP client | Native `fetch` | 18+ |
| `fs-extra` | File utilities | `fs/promises` | 14+ |
| `url-join` | URL construction | `URL` class | 10+ |
| `querystring` | Query strings | `URLSearchParams` | 10+ |

**Key Insight:** With Node.js 22+, we can eliminate 6 dependencies entirely using native APIs. This makes ESM migration trivial since we're not upgrading ESM-only packages - we're deleting them.

### Final Decision: Node.js 22+ with ESM

See: [ADR-001: Target Node.js 22+ with ESM and Native APIs](../adr/001-node22-esm-native-apis.md)

### Optimized Phase Sequence

```
Phase 1: Foundation
├── Update package.json (Node 22+, "type": "module")
├── Convert to ESM syntax
└── Remove deleted dependencies

Phase 2: Native APIs
├── Replace got → fetch
├── Replace fs-extra → fs/promises
├── Replace globby → fs.glob()
├── Replace aggregate-error → AggregateError
├── Replace url-join → URL
└── Replace querystring → URLSearchParams

Phase 3: Forgejo Support
├── Add FORGEJO_TOKEN / FORGEJO_URL env vars
├── Prioritize Forgejo in documentation
└── Test against PCC Forgejo instance

Phase 4: Testing
├── Migrate to node:test
├── Use c8 for coverage
└── Update mocks for native fetch

Phase 5: CI/CD & Documentation
├── Update GitHub Actions
├── Update README
└── Publish @pcc/semantic-release-gitea
```

---

## Part 3: Risk Analysis

### Risk Assessment Matrix

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Native `fetch` differs from `got` | Medium | Medium | Comprehensive tests, handle edge cases |
| `fs.glob()` API differences | Medium | Low | Test all glob patterns |
| ESM import issues with deps | Low | Medium | lodash-es, debug, form-data support ESM |
| Nock compatibility with fetch | Medium | Medium | Nock 14.x supports fetch; verify |
| Users on Node < 22 | High | Low | Clear docs, v3.0.0 signals breaking |
| CI without Node 22 | Low | Low | Most CI providers support it |
| semantic-release API changes | Low | High | Review v24 plugin API |
| Forgejo API edge cases | Low | Medium | API spec available; test real instance |

### Key Technical Risks

**1. Native `fetch` Error Handling**

Current `got` style:
```javascript
catch (error) {
  if (error.response?.statusCode === 401) { ... }
}
```

New `fetch` style requires manual error construction:
```javascript
if (!response.ok) {
  const error = new Error(`API error: ${response.status}`);
  error.status = response.status;
  throw error;
}
```

**2. `fs.glob()` Return Type**

```javascript
// globby (current) - returns array
const files = await globby(pattern);

// fs.glob() (new) - returns AsyncIterator
const files = await Array.fromAsync(fs.glob(pattern));
```

**3. Form Data for Asset Upload**

The `form-data` package is still needed for multipart file uploads. Native `FormData` exists but file handling compatibility needs verification.

### Recommended Testing Strategy

1. **Unit tests**: Each replaced function in isolation
2. **Integration tests**: Full publish flow against mock Forgejo
3. **E2E test**: Real Forgejo instance (PCC's) before release

---

## Session Outcomes

### Artifacts Produced

1. **ADR-001**: Node.js 22+ with ESM and Native APIs decision
2. **Forgejo API Spec**: Saved to `docs/forgejo-api-v1.json`
3. **This brainstorming document**: Session record and decisions

### Key Decisions Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Node.js version | 22+ | Native `fs.glob()`, active LTS |
| Module system | ESM | Aligned with native APIs, future-proof |
| Platform priority | Forgejo first | PCC deployment target |
| Backward compat | None | Fresh fork, clean break |
| Dependencies | Minimize | 6 deps replaced with native APIs |

### Next Steps

1. Update workflow status with brainstorming completion
2. Proceed to PRD creation with PM agent
3. Use this session as input for detailed requirements

---

## References

- [Original modernization plan](../../semantic-release-gitea-modernization.md)
- [ADR-001: Node.js 22+ Decision](../adr/001-node22-esm-native-apis.md)
- [Forgejo API Spec](../forgejo-api-v1.json)
- [Node.js Releases](https://nodejs.org/en/about/previous-releases)
