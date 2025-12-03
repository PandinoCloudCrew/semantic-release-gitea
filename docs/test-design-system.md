# System-Level Test Design

**Project:** semantic-release-gitea → @pcc/semantic-release-forgejo
**Phase:** 3 (Solutioning - Testability Review)
**Date:** 2025-12-02
**Author:** Murat (Test Architect)

---

## Testability Assessment

### Controllability: ✅ PASS

The architecture supports full test controllability:

| Aspect | Assessment | Evidence |
|--------|------------|----------|
| **State Control** | Stateless functions | All modules are pure or use injected dependencies |
| **External Dependencies** | Mockable | Single HTTP client (`get-client.js`) enables nock interception |
| **Error Injection** | Supported | nock can simulate 401, 403, 404, 500, 429 responses |
| **Configuration** | Injectable | All config flows through `resolveConfig()` with env/options merge |

**Key Control Points:**
- `GiteaClient` class centralizes all HTTP calls → single mock point
- Environment variables injectable via context object
- Plugin config injectable via first argument to hooks

### Observability: ✅ PASS

Test results are deterministic and inspectable:

| Aspect | Assessment | Evidence |
|--------|------------|----------|
| **API Responses** | Predictable | JSON responses with clear structure |
| **Error Messages** | Structured | SemanticReleaseError with code, message, details |
| **Debug Output** | Available | `debug` package for verbose logging |
| **Return Values** | Explicit | Hooks return release URL, ID, name |

**Observable Outputs:**
- `verifyConditions`: throws AggregateError or resolves silently
- `publish`: returns `{ url, name, id }`
- `addChannel`: returns `{ url, name, id }`

### Reliability: ✅ PASS

Tests can run reliably in isolation and parallel:

| Aspect | Assessment | Evidence |
|--------|------------|----------|
| **Isolation** | Stateless | No shared mutable state between calls |
| **Parallel Safety** | Safe | Each test can use unique nock scopes |
| **Determinism** | High | No random data, timestamps, or race conditions |
| **Cleanup** | Automatic | nock.cleanAll() in afterEach |

**Existing Test Patterns (to preserve):**
```javascript
test.afterEach.always(() => {
  nock.cleanAll();
});
```

---

## Architecturally Significant Requirements (ASRs)

### ASR-1: Token Security (NFR1)

**Requirement:** API tokens must never be logged or exposed in output

**Quality Scenario:**
- Given: A release fails with authentication error
- When: Error message is displayed to user
- Then: Token value MUST NOT appear in message or stack trace

**Testability Approach:**
```javascript
test('error messages do not contain token', async () => {
  const token = 'secret_token_value';
  const env = { FORGEJO_TOKEN: token, FORGEJO_URL: 'https://forgejo.example.com' };

  nock('https://forgejo.example.com')
    .get('/api/v1/repos/owner/repo')
    .reply(401);

  const error = await assertRejects(() => verify({}, { env, options, logger }));

  assert.ok(!error.message.includes(token));
  assert.ok(!error.details?.includes(token));
  assert.ok(!JSON.stringify(error).includes(token));
});
```

**Risk Score:** Probability=2, Impact=3 → **Score 6 (HIGH)**

**Mitigation:** Explicit test coverage for all error paths

---

### ASR-2: Fail-Fast Verification (NFR9)

**Requirement:** Plugin must fail during verifyConditions, not during publish

**Quality Scenario:**
- Given: Invalid configuration (missing token, wrong URL, no permissions)
- When: semantic-release calls verifyConditions
- Then: All errors aggregated and thrown immediately

**Testability Approach:**
```javascript
test('aggregates multiple validation errors', async () => {
  const env = {}; // Missing token AND URL

  const error = await assertRejects(() => verify({}, { env, options, logger }));

  assert.ok(error instanceof AggregateError);
  assert.ok(error.errors.length >= 2);
  assert.ok(error.errors.some(e => e.code === 'ENOTOKEN'));
  assert.ok(error.errors.some(e => e.code === 'ENOURL'));
});
```

**Risk Score:** Probability=2, Impact=2 → **Score 4 (MEDIUM)**

---

### ASR-3: Forgejo-First Configuration (FR3-5)

**Requirement:** FORGEJO_* env vars take precedence over GITEA_*

**Quality Scenario:**
- Given: Both FORGEJO_TOKEN and GITEA_TOKEN are set
- When: Plugin resolves configuration
- Then: FORGEJO_TOKEN is used

**Testability Approach:**
```javascript
test('prioritizes FORGEJO_TOKEN over GITEA_TOKEN', () => {
  const env = {
    FORGEJO_TOKEN: 'forgejo-token',
    GITEA_TOKEN: 'gitea-token',
    FORGEJO_URL: 'https://forgejo.example.com'
  };

  const config = resolveConfig({}, { env });

  assert.strictEqual(config.giteaToken, 'forgejo-token');
  assert.strictEqual(config.platform, 'Forgejo');
});
```

**Risk Score:** Probability=1, Impact=2 → **Score 2 (LOW)**

---

### ASR-4: Asset Upload Atomicity (NFR11)

**Requirement:** Releases with assets must not be visible until all assets uploaded

**Quality Scenario:**
- Given: Release configured with 3 assets
- When: Second asset upload fails
- Then: Release remains as draft (not published)

**Testability Approach:**
```javascript
test('release stays draft when asset upload fails', async () => {
  nock('https://forgejo.example.com')
    .post('/api/v1/repos/owner/repo/releases')
    .reply(201, { id: 1, draft: true })
    .post('/api/v1/repos/owner/repo/releases/1/assets')
    .reply(201) // First asset succeeds
    .post('/api/v1/repos/owner/repo/releases/1/assets')
    .reply(500); // Second asset fails

  await assertRejects(() => publish(config, context));

  // Verify no PATCH to undraft was called
  assert.ok(!nock.isDone()); // Undraft endpoint never hit
});
```

**Risk Score:** Probability=2, Impact=2 → **Score 4 (MEDIUM)**

---

## Test Levels Strategy

### Recommended Test Pyramid

```
        ╱╲
       ╱  ╲     E2E (5%)
      ╱────╲    - Full plugin flow with semantic-release context
     ╱      ╲
    ╱────────╲  Integration (35%)
   ╱          ╲ - API contract tests (verify, publish, add-channel)
  ╱────────────╲- HTTP client with nock mocking
 ╱              ╲
╱────────────────╲ Unit (60%)
                   - Pure functions (is-prerelease, parse-git-url)
                   - Error factory, config resolution
                   - Asset globbing
```

### Test Level Assignments

| Module | Test Level | Rationale |
|--------|------------|-----------|
| `is-prerelease.js` | Unit | Pure function, no dependencies |
| `parse-git-url.js` | Unit | Pure function, string manipulation |
| `get-error.js` | Unit | Factory function, template replacement |
| `resolve-config.js` | Unit | Object merging, no external calls |
| `glob-assets.js` | Unit + Integration | Pure logic + filesystem interaction |
| `get-client.js` | Integration | HTTP client, requires nock |
| `verify.js` | Integration | Orchestrates client + config |
| `publish.js` | Integration | Orchestrates client + assets |
| `add-channel.js` | Integration | Orchestrates client |
| `index.js` (exports) | E2E | Full plugin contract |

### Coverage Targets

| Priority | Pass Rate | Coverage Target |
|----------|-----------|-----------------|
| P0 (Critical) | 100% | Token security, error handling, publish flow |
| P1 (High) | ≥95% | Configuration resolution, asset upload |
| P2 (Medium) | ≥90% | Edge cases, glob patterns |
| Overall | - | ≥80% line coverage |

---

## NFR Testing Approach

### Security (NFR1-3)

| Test | Type | Description |
|------|------|-------------|
| Token not in errors | Unit | Verify SemanticReleaseError excludes token |
| Token not logged | Unit | Verify debug output excludes token |
| HTTPS enforcement | Integration | Verify URL normalization adds https |

**Tools:** node:test, sinon (spy on logger)

### Reliability (NFR8-11)

| Test | Type | Description |
|------|------|-------------|
| Clear error messages | Unit | Each error code has actionable message |
| Error aggregation | Integration | Multiple errors collected before throw |
| Timeout handling | Integration | nock timeout simulation |
| Draft workflow | Integration | Assets fail → release stays draft |

**Tools:** node:test, nock (timeout, error simulation)

### Maintainability (NFR12-15)

| Metric | Target | Tool |
|--------|--------|------|
| Code coverage | ≥80% | c8 |
| ESM compliance | 100% | xo (unicorn/prefer-module) |
| No CommonJS | 0 require() | xo lint |

**CI Integration:**
```yaml
- name: Test with coverage
  run: c8 node --test test/**/*.test.js

- name: Check coverage threshold
  run: c8 check-coverage --lines 80
```

---

## Test Environment Requirements

### Local Development

```bash
# Node.js 22+ required
node --version  # v22.x.x

# Run tests
npm test        # node --test test/**/*.test.js

# Run with coverage
npm run test:coverage  # c8 node --test test/**/*.test.js
```

### CI Environment (GitHub Actions)

```yaml
strategy:
  matrix:
    node-version: ['22.x', '24.x']

steps:
  - uses: actions/setup-node@v4
    with:
      node-version: ${{ matrix.node-version }}

  - run: npm ci
  - run: npm test
  - run: npm run lint
```

### Mock Server Configuration

```javascript
// test/helpers/mock-server.js
import nock from 'nock';

export function mockForgejo(baseUrl = 'https://forgejo.example.com') {
  return nock(baseUrl, {
    reqheaders: {
      'Authorization': /^token .+$/
    }
  }).persist();
}

export function mockRepoAccess(scope, owner, repo, permissions = { push: true }) {
  return scope
    .get(`/api/v1/repos/${owner}/${repo}`)
    .reply(200, { permissions });
}

export function mockReleaseCreate(scope, owner, repo, releaseData) {
  return scope
    .post(`/api/v1/repos/${owner}/${repo}/releases`)
    .reply(201, { id: 1, html_url: `${scope.basePath}/releases/1`, ...releaseData });
}
```

---

## Testability Concerns

### Resolved Concerns

| Concern | Resolution |
|---------|------------|
| nock 11 → 14 migration | nock 14+ supports native fetch() - verified in documentation |
| proxyquire ESM incompatible | Remove - use test fixtures and dependency injection |
| clear-module ESM incompatible | Remove - ESM module caching handled differently |

### No Blocking Concerns

All identified testability issues have clear mitigations. Architecture supports:
- ✅ Full HTTP mocking via nock
- ✅ Stateless function testing
- ✅ Error scenario simulation
- ✅ Parallel test execution

---

## Recommendations for Sprint 0

### Before Epic 1 Implementation

1. **Spike: nock 14 + fetch() verification**
   ```javascript
   // Verify nock intercepts native fetch
   import nock from 'nock';

   nock('https://example.com')
     .get('/test')
     .reply(200, { ok: true });

   const response = await fetch('https://example.com/test');
   const data = await response.json();
   console.log(data); // { ok: true }
   ```

2. **Remove ESM-incompatible devDependencies**
   - `proxyquire` - not needed with ESM
   - `clear-module` - not needed with ESM

3. **Create test template**
   ```javascript
   // test/example.test.js
   import { describe, it, before, after, beforeEach, afterEach } from 'node:test';
   import assert from 'node:assert';
   import nock from 'nock';

   describe('Module', () => {
     beforeEach(() => {
       nock.disableNetConnect();
     });

     afterEach(() => {
       nock.cleanAll();
       nock.enableNetConnect();
     });

     it('should do something', async () => {
       // Arrange
       nock('https://example.com').get('/').reply(200);

       // Act
       const result = await someFunction();

       // Assert
       assert.strictEqual(result, expected);
     });
   });
   ```

---

## Quality Gate Criteria

### Gate Decision: ✅ PASS (Testability)

| Criterion | Status | Notes |
|-----------|--------|-------|
| Controllability | ✅ PASS | All external deps mockable |
| Observability | ✅ PASS | Clear outputs, structured errors |
| Reliability | ✅ PASS | Stateless, parallel-safe |
| ASRs identified | ✅ PASS | 4 ASRs with test approaches |
| Test strategy defined | ✅ PASS | 60/35/5 pyramid |
| No blocking concerns | ✅ PASS | All concerns have mitigations |

### Next Steps

1. Run `*framework` workflow to scaffold test infrastructure
2. Run `*ci` workflow to configure GitHub Actions
3. Proceed to `implementation-readiness` gate check
4. Begin Epic 1 implementation

---

**Document Status:** Complete
**Gate Recommendation:** PROCEED to implementation-readiness

