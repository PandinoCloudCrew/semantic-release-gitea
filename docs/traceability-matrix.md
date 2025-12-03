# Requirements-to-Tests Traceability Matrix

**Project:** semantic-release-gitea → @pcc/semantic-release-forgejo
**Phase:** System-Level Traceability (Pre-Implementation)
**Date:** 2025-12-02
**Author:** Murat (Test Architect)

---

## Executive Summary

This traceability matrix maps the 28 Functional Requirements and 18 Non-Functional Requirements from the PRD to the existing test suite. The analysis identifies:

- **Current Coverage:** 18/28 FRs have existing tests (64%)
- **Gap Analysis:** 10 FRs require new tests after modernization
- **NFR Coverage:** 5/18 NFRs have indirect test coverage
- **Risk Assessment:** No P0 gaps in existing functionality

---

## Coverage Summary

| Priority | Total Requirements | Covered | Coverage % | Status |
|----------|-------------------|---------|------------|--------|
| **P0 (Critical)** | 6 | 6 | 100% | ✅ PASS |
| **P1 (High)** | 12 | 8 | 67% | ⚠️ GAPS |
| **P2 (Medium)** | 8 | 4 | 50% | ⚠️ GAPS |
| **P3 (Low)** | 2 | 0 | 0% | Acceptable |
| **Total FR** | 28 | 18 | 64% | - |
| **Total NFR** | 18 | 5 | 28% | - |

---

## Functional Requirements Traceability

### Authentication & Verification (FR1-FR6)

| FR ID | Description | Priority | Test Coverage | Test Files | Status |
|-------|-------------|----------|---------------|------------|--------|
| **FR1** | Validate Forgejo/Gitea API token before release | P0 | FULL | `verify.test.js:22-37`, `integration.test.js:31-43` | ✅ |
| **FR2** | Verify user has push permissions | P0 | FULL | `verify.test.js:233-249` | ✅ |
| **FR3** | Authenticate using `FORGEJO_TOKEN` | P1 | **NONE** | - | ❌ NEW |
| **FR4** | Fallback to `GITEA_TOKEN` if `FORGEJO_TOKEN` not set | P1 | **NONE** | - | ❌ NEW |
| **FR5** | Resolve URL from `FORGEJO_URL` or `GITEA_URL` | P1 | PARTIAL | `verify.test.js:60-75` (GITEA only) | ⚠️ |
| **FR6** | Accept server URL from plugin configuration | P2 | PARTIAL | Implicit in tests | ⚠️ |

**Gap Analysis (FR3-FR6):**
- FR3/FR4: Forgejo-first environment variables not yet implemented
- FR5/FR6: Need explicit tests for FORGEJO_URL precedence

---

### Release Management (FR7-FR12)

| FR ID | Description | Priority | Test Coverage | Test Files | Status |
|-------|-------------|----------|---------------|------------|--------|
| **FR7** | Create a new release on Forgejo/Gitea | P0 | FULL | `publish.test.js:25-55`, `integration.test.js:109-154` | ✅ |
| **FR8** | Set release tag name from context | P0 | FULL | `publish.test.js:30` | ✅ |
| **FR9** | Set release name from context | P1 | FULL | `publish.test.js:30` | ✅ |
| **FR10** | Set release body (notes) from context | P1 | FULL | `publish.test.js:30` | ✅ |
| **FR11** | Mark release as prerelease based on branch | P1 | FULL | `publish.test.js:57-119` | ✅ |
| **FR12** | Update existing release prerelease status | P1 | FULL | `add-channel.test.js:23-54`, `integration.test.js:204-234` | ✅ |

**Gap Analysis (FR7-FR12):** All release management requirements have full coverage.

---

### Asset Management (FR13-FR17)

| FR ID | Description | Priority | Test Coverage | Test Files | Status |
|-------|-------------|----------|---------------|------------|--------|
| **FR13** | Upload file assets to a release | P0 | FULL | `publish.test.js:154-198`, `integration.test.js:109-154` | ✅ |
| **FR14** | Resolve asset paths using glob patterns | P1 | FULL | `glob-assets.test.js` (17 tests) | ✅ |
| **FR15** | Set custom names for uploaded assets | P2 | FULL | `publish.test.js:200-244` | ✅ |
| **FR16** | Set custom labels for uploaded assets | P2 | FULL | `publish.test.js:154-198` | ✅ |
| **FR17** | Handle multiple assets per release | P1 | FULL | `integration.test.js:109-154` | ✅ |

**Gap Analysis (FR13-FR17):** All asset management requirements have full coverage.

---

### Error Handling (FR18-FR21)

| FR ID | Description | Priority | Test Coverage | Test Files | Status |
|-------|-------------|----------|---------------|------------|--------|
| **FR18** | Return specific error codes for failures | P1 | FULL | `verify.test.js:191-373` (12 error scenarios) | ✅ |
| **FR19** | Provide actionable error messages | P2 | PARTIAL | Error codes tested, messages not validated | ⚠️ |
| **FR20** | Aggregate multiple validation errors | P1 | FULL | `integration.test.js:86-107` | ✅ |
| **FR21** | Distinguish auth, permission, API errors | P2 | FULL | `verify.test.js:191-281` | ✅ |

**Gap Analysis (FR18-FR21):**
- FR19: Need explicit assertion on error message content (not just code)

---

### Configuration (FR22-FR24)

| FR ID | Description | Priority | Test Coverage | Test Files | Status |
|-------|-------------|----------|---------------|------------|--------|
| **FR22** | Accept custom API path prefix | P2 | PARTIAL | `verify.test.js:60-75` (env only) | ⚠️ |
| **FR23** | Merge env variables with plugin options | P2 | **NONE** | - | ❌ NEW |
| **FR24** | Parse repository URL to extract owner/repo | P1 | FULL | Implicit in all tests | ✅ |

**Gap Analysis (FR22-FR24):**
- FR22: Need test for plugin option `giteaApiPathPrefix`
- FR23: Need explicit test for env + options merge precedence

---

### Integration (FR25-FR28)

| FR ID | Description | Priority | Test Coverage | Test Files | Status |
|-------|-------------|----------|---------------|------------|--------|
| **FR25** | Export `verifyConditions` hook | P0 | FULL | `integration.test.js:31-84` | ✅ |
| **FR26** | Export `publish` hook | P0 | FULL | `integration.test.js:109-202` | ✅ |
| **FR27** | Export `addChannel` hook | P1 | FULL | `integration.test.js:204-315` | ✅ |
| **FR28** | Return release URL after publish | P1 | FULL | `publish.test.js:52`, `integration.test.js:147` | ✅ |

**Gap Analysis (FR25-FR28):** All integration requirements have full coverage.

---

## Non-Functional Requirements Traceability

### Security (NFR1-NFR3)

| NFR ID | Description | Priority | Test Coverage | Status |
|--------|-------------|----------|---------------|--------|
| **NFR1** | API tokens never logged or exposed | P0 | **NONE** | ❌ NEW |
| **NFR2** | All API communications use HTTPS | P1 | IMPLICIT | All test URLs use https:// | ⚠️ |
| **NFR3** | Token validation fails securely | P1 | PARTIAL | 401 tested, not info leakage | ⚠️ |

**Gap Analysis (NFR1-NFR3):**
- NFR1: **CRITICAL** - Need explicit test that token is not in error output
- NFR2: Need test that rejects http:// URLs
- NFR3: Need test that 401 doesn't reveal token validity

---

### Integration (NFR4-NFR7)

| NFR ID | Description | Priority | Test Coverage | Status |
|--------|-------------|----------|---------------|--------|
| **NFR4** | Implement semantic-release plugin contract v24+ | P0 | FULL | `integration.test.js` | ✅ |
| **NFR5** | Use Forgejo/Gitea API v1 endpoints | P1 | FULL | All tests use /api/v1/ | ✅ |
| **NFR6** | Handle API rate limiting gracefully | P2 | **NONE** | ❌ NEW |
| **NFR7** | Support standard auth schemes | P1 | FULL | Token auth tested | ✅ |

**Gap Analysis (NFR4-NFR7):**
- NFR6: Need test for 429 response handling

---

### Reliability (NFR8-NFR11)

| NFR ID | Description | Priority | Test Coverage | Status |
|--------|-------------|----------|---------------|--------|
| **NFR8** | Clear error messages for all failures | P1 | PARTIAL | Codes tested, not messages | ⚠️ |
| **NFR9** | Fail fast during verifyConditions | P0 | FULL | All errors in verify.test.js | ✅ |
| **NFR10** | Handle network timeouts gracefully | P2 | **NONE** | ❌ NEW |
| **NFR11** | Asset uploads use draft workflow | P1 | FULL | `publish.test.js:154-284` | ✅ |

**Gap Analysis (NFR8-NFR11):**
- NFR10: Need test for timeout behavior with nock delays

---

### Maintainability (NFR12-NFR15)

| NFR ID | Description | Priority | Test Coverage | Status |
|--------|-------------|----------|---------------|--------|
| **NFR12** | ESM exclusively | P1 | **PENDING** | Migration required | ⏳ |
| **NFR13** | Native Node.js APIs or maintained deps | P1 | **PENDING** | Migration required | ⏳ |
| **NFR14** | Test coverage maintained or improved | P1 | TRACKING | c8 will validate | ⏳ |
| **NFR15** | Code passes XO linting | P2 | EXISTING | npm run lint | ✅ |

**Gap Analysis (NFR12-NFR15):**
- NFR12-14: Will be validated after migration

---

### Compatibility (NFR16-NFR18)

| NFR ID | Description | Priority | Test Coverage | Status |
|--------|-------------|----------|---------------|--------|
| **NFR16** | Node.js 22+ support | P0 | **PENDING** | CI matrix update | ⏳ |
| **NFR17** | npm, pnpm, yarn support | P3 | N/A | Package manager agnostic | - |
| **NFR18** | Forgejo and Gitea support | P1 | **PARTIAL** | Gitea only currently | ⚠️ |

---

## Gap Analysis Summary

### Critical Gaps (P0 - Blocks Release)

| Gap ID | Requirement | Description | Recommended Test |
|--------|-------------|-------------|------------------|
| **GAP-1** | NFR1 | Token not in error output | Unit test: verify error.message excludes token |

### High Priority Gaps (P1 - Blocks PR)

| Gap ID | Requirement | Description | Recommended Test |
|--------|-------------|-------------|------------------|
| **GAP-2** | FR3 | FORGEJO_TOKEN support | Integration: auth with FORGEJO_TOKEN |
| **GAP-3** | FR4 | FORGEJO→GITEA fallback | Integration: fallback when FORGEJO missing |
| **GAP-4** | FR5 | FORGEJO_URL precedence | Integration: FORGEJO_URL overrides GITEA_URL |
| **GAP-5** | NFR18 | Forgejo compatibility | Integration: test against Forgejo API mock |

### Medium Priority Gaps (P2 - Nightly Tests)

| Gap ID | Requirement | Description | Recommended Test |
|--------|-------------|-------------|------------------|
| **GAP-6** | FR19 | Actionable error messages | Unit: validate message content |
| **GAP-7** | FR22 | Plugin option API prefix | Integration: giteaApiPathPrefix option |
| **GAP-8** | FR23 | Env + options merge | Unit: precedence rules |
| **GAP-9** | NFR6 | Rate limiting (429) | Integration: mock 429, verify retry/message |
| **GAP-10** | NFR10 | Network timeout | Integration: mock timeout, verify error |

### Low Priority Gaps (P3 - Optional)

None identified.

---

## Test Quality Assessment

### Existing Test Suite Quality

| Metric | Value | Target | Status |
|--------|-------|--------|--------|
| **Total Tests** | ~60 tests across 5 files | - | - |
| **Test Files** | 5 | - | - |
| **Assertions per Test** | 2-4 average | >1 | ✅ |
| **Hard Waits** | 0 | 0 | ✅ |
| **Cleanup Pattern** | `nock.cleanAll()` in afterEach | Required | ✅ |
| **Test Isolation** | Serial execution with nock | Required | ✅ |

### Quality Concerns

| Concern | Severity | File | Recommendation |
|---------|----------|------|----------------|
| proxyquire usage | MEDIUM | integration.test.js | Remove for ESM |
| clear-module usage | MEDIUM | integration.test.js | Remove for ESM |
| fs-extra in tests | LOW | glob-assets.test.js | Replace with fs/promises |

---

## Recommendations

### Before Epic 1 (Sprint 0)

1. **Add NFR1 test (Token Security)** - Critical gap, P0 priority
   ```javascript
   test('error messages do not contain token', async () => {
     const token = 'secret_token_value';
     // ... verify token not in any error output
   });
   ```

2. **Create test template** - Establish node:test patterns for migration

### During Epic 1 (Modernization)

1. **Add FR3-FR5 tests** - Forgejo environment variable support
2. **Add NFR6/NFR10 tests** - Rate limiting and timeout handling
3. **Update test infrastructure** - Remove proxyquire, clear-module

### Post-MVP

1. **Add FR19 tests** - Validate error message content
2. **Add FR22-FR23 tests** - Configuration merge precedence

---

## Quality Gate Decision

### Phase 1 (Traceability): ⚠️ CONCERNS

**Summary:** Existing test suite covers core functionality well (64% FR coverage, 100% P0 FR coverage), but gaps exist for Forgejo-first features and NFR security validation.

**Decision Criteria:**

| Criterion | Threshold | Actual | Status |
|-----------|-----------|--------|--------|
| P0 FR Coverage | 100% | 100% (6/6) | ✅ PASS |
| P1 FR Coverage | ≥90% | 67% (8/12) | ⚠️ GAP |
| P0 NFR Coverage | 100% | 50% (1/2)* | ⚠️ GAP |
| Critical Gaps | 0 | 1 (NFR1) | ⚠️ GAP |

*NFR4 (plugin contract) is covered; NFR1 (token security) is not.

**Recommendation:**
- Address GAP-1 (token security) in Sprint 0
- Address GAP-2 through GAP-5 (Forgejo support) in Epic 1
- Proceed with implementation readiness check

---

## Phase 2: Quality Gate Decision

### Gate Type: System-Level (Pre-Implementation)

**Decision:** ⚠️ **CONCERNS** - Proceed with documented gaps

**Rationale:**
1. **P0 functional coverage is 100%** - Core plugin functionality fully tested
2. **Critical security gap (NFR1)** - Must add token exposure test before Epic 1
3. **Forgejo gaps expected** - FR3-FR5 require implementation before testing
4. **Test infrastructure solid** - nock patterns, cleanup, assertions all good

### Action Items

| Priority | Action | Owner | Sprint |
|----------|--------|-------|--------|
| P0 | Add NFR1 token security test | Dev | Sprint 0 |
| P1 | Add FR3-FR5 Forgejo env tests | Dev | Epic 1 |
| P1 | Add NFR18 Forgejo compatibility | Dev | Epic 1 |
| P2 | Add NFR6/NFR10 resilience tests | Dev | Epic 1 |

### Sign-off

- **Traceability Complete:** Yes
- **Gaps Documented:** Yes
- **Gate Decision:** CONCERNS (proceed with action items)
- **Next Step:** implementation-readiness workflow

---

## References

- PRD: `docs/prd.md`
- Test Design: `docs/test-design-system.md`
- Architecture: `docs/architecture-decision.md`
- Existing Tests: `test/*.test.js`

---

**Document Status:** Complete
**Gate Recommendation:** CONCERNS - Proceed with Sprint 0 action items

