# semantic-release-gitea Fork Modernization Plan

Repository: https://github.com/PandinoCloudCrew/semantic-release-gitea

## Current State (forked from saitho/semantic-release-gitea v2.1.0)

- **Last upstream release:** April 2021
- **Node.js support:** ^10 || ^11 || ^12 || >=14
- **semantic-release:** v17.x
- **HTTP client:** got@10
- **Test framework:** ava@2

## Modernization Goals

1. Support Node.js 20+ (LTS)
2. Update to semantic-release v24
3. Modernize dependencies
4. Add Forgejo-specific compatibility
5. Maintain backward compatibility with Gitea

---

## Phase 1: Dependency Updates

### package.json Changes

```json
{
  "name": "@pcc/semantic-release-gitea",
  "version": "3.0.0",
  "description": "Publish release notes to Gitea/Forgejo",
  "main": "index.js",
  "type": "module",
  "engines": {
    "node": ">=20"
  },
  "scripts": {
    "lint": "eslint .",
    "test": "node --test",
    "test:coverage": "c8 node --test"
  },
  "dependencies": {
    "@semantic-release/error": "^4.0.0",
    "aggregate-error": "^5.0.0",
    "debug": "^4.3.4",
    "dir-glob": "^3.0.1",
    "form-data": "^4.0.0",
    "globby": "^14.0.0",
    "lodash-es": "^4.17.21"
  },
  "devDependencies": {
    "c8": "^9.0.0",
    "eslint": "^9.0.0",
    "nock": "^14.0.0",
    "semantic-release": "^24.0.0",
    "sinon": "^17.0.0"
  }
}
```

### Dependency Migration Table

| Old Dependency | Version | New Dependency | Version | Notes |
|----------------|---------|----------------|---------|-------|
| `got` | 10.x | Native `fetch` | - | Use Node.js built-in fetch |
| `fs-extra` | 8.x | Native `fs/promises` | - | Use Node.js built-in |
| `globby` | 10.x | `globby` | 14.x | ESM only |
| `aggregate-error` | 3.x | `aggregate-error` | 5.x | ESM only |
| `lodash` | 4.x | `lodash-es` | 4.x | ESM version |
| `url-join` | 4.x | Native `URL` | - | Use Node.js built-in |
| `querystring` | 0.2.x | `URLSearchParams` | - | Use Node.js built-in |
| `ava` | 2.x | `node:test` | - | Use Node.js built-in test runner |
| `nyc` | 15.x | `c8` | 9.x | Native V8 coverage |
| `xo` | 0.38.x | `eslint` | 9.x | Direct ESLint with flat config |

---

## Phase 2: Code Modernization

### 2.1 Convert to ESM

**index.js:**
```javascript
export { verifyConditions } from './lib/verify.js';
export { publish } from './lib/publish.js';
export { addChannel } from './lib/add-channel.js';
```

### 2.2 Replace `got` with Native Fetch

**lib/gitea.js (before):**
```javascript
const got = require('got');

async function createRelease(giteaUrl, token, owner, repo, data) {
  return got.post(`${giteaUrl}/api/v1/repos/${owner}/${repo}/releases`, {
    headers: { Authorization: `token ${token}` },
    json: data,
    responseType: 'json'
  });
}
```

**lib/gitea.js (after):**
```javascript
async function createRelease(giteaUrl, token, owner, repo, data) {
  const url = new URL(`/api/v1/repos/${owner}/${repo}/releases`, giteaUrl);

  const response = await fetch(url, {
    method: 'POST',
    headers: {
      'Authorization': `token ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(data)
  });

  if (!response.ok) {
    throw new Error(`Gitea API error: ${response.status} ${response.statusText}`);
  }

  return response.json();
}

export { createRelease };
```

### 2.3 Replace `fs-extra` with Native fs/promises

**lib/glob-assets.js (before):**
```javascript
const { stat } = require('fs-extra');
```

**lib/glob-assets.js (after):**
```javascript
import { stat } from 'node:fs/promises';
```

### 2.4 Replace `url-join` with Native URL

**Before:**
```javascript
const urlJoin = require('url-join');
const apiUrl = urlJoin(giteaUrl, 'api/v1/repos', owner, repo);
```

**After:**
```javascript
const apiUrl = new URL(`/api/v1/repos/${owner}/${repo}`, giteaUrl).href;
```

---

## Phase 3: Forgejo Compatibility

### 3.1 Add Forgejo Context Support

Forgejo uses `forgejo.*` context variables alongside `github.*` compatibility.

**lib/get-config.js:**
```javascript
export function getConfig(pluginConfig, context) {
  const { env } = context;

  return {
    // Support both Gitea and Forgejo env vars
    giteaToken: env.GITEA_TOKEN || env.FORGEJO_TOKEN,
    giteaUrl: pluginConfig.giteaUrl || env.GITEA_URL || env.FORGEJO_URL,
    giteaApiPathPrefix: pluginConfig.giteaApiPathPrefix || env.GITEA_PREFIX || '/api/v1',
    assets: pluginConfig.assets || []
  };
}
```

### 3.2 Add Forgejo API Compatibility Check

Forgejo API is compatible with Gitea but may have additional endpoints:

```javascript
async function getServerInfo(baseUrl, token) {
  const url = new URL('/api/v1/version', baseUrl);
  const response = await fetch(url, {
    headers: { 'Authorization': `token ${token}` }
  });

  if (!response.ok) return { type: 'unknown', version: null };

  const data = await response.json();

  // Forgejo returns "version": "8.0.0+forgejo"
  const isForgejo = data.version?.includes('forgejo');

  return {
    type: isForgejo ? 'forgejo' : 'gitea',
    version: data.version
  };
}
```

---

## Phase 4: Testing Updates

### 4.1 Migrate to Node.js Test Runner

**test/publish.test.js:**
```javascript
import { describe, it, beforeEach, afterEach, mock } from 'node:test';
import assert from 'node:assert';
import nock from 'nock';
import { publish } from '../lib/publish.js';

describe('publish', () => {
  beforeEach(() => {
    nock.disableNetConnect();
  });

  afterEach(() => {
    nock.cleanAll();
    nock.enableNetConnect();
  });

  it('should create a release on Gitea', async () => {
    const giteaUrl = 'https://gitea.example.com';

    nock(giteaUrl)
      .post('/api/v1/repos/owner/repo/releases')
      .reply(201, { id: 1, tag_name: 'v1.0.0' });

    const result = await publish(
      { giteaUrl },
      {
        env: { GITEA_TOKEN: 'test-token' },
        nextRelease: { version: '1.0.0', gitTag: 'v1.0.0', notes: 'Release notes' },
        options: { repositoryUrl: 'https://gitea.example.com/owner/repo.git' },
        logger: { log: () => {} }
      }
    );

    assert.strictEqual(result.id, 1);
  });

  it('should create a release on Forgejo', async () => {
    const forgejoUrl = 'https://forgejo.example.com';

    nock(forgejoUrl)
      .post('/api/v1/repos/owner/repo/releases')
      .reply(201, { id: 2, tag_name: 'v2.0.0' });

    const result = await publish(
      { giteaUrl: forgejoUrl },
      {
        env: { FORGEJO_TOKEN: 'test-token' },
        nextRelease: { version: '2.0.0', gitTag: 'v2.0.0', notes: 'Release notes' },
        options: { repositoryUrl: 'https://forgejo.example.com/owner/repo.git' },
        logger: { log: () => {} }
      }
    );

    assert.strictEqual(result.id, 2);
  });
});
```

---

## Phase 5: CI/CD Updates

### 5.1 GitHub Actions Workflow

**.github/workflows/ci.yml:**
```yaml
name: CI

on:
  push:
    branches: [main, master]
  pull_request:
    branches: [main, master]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [20, 22]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm run test:coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        if: matrix.node-version == 22

  release:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'

      - run: npm ci
      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## Phase 6: Documentation Updates

### 6.1 README.md Updates

```markdown
# semantic-release-gitea

semantic-release plugin to publish releases to Gitea and Forgejo.

## Installation

```bash
npm install @pcc/semantic-release-gitea -D
```

## Configuration

### Environment Variables

| Variable | Description |
|----------|-------------|
| `GITEA_TOKEN` or `FORGEJO_TOKEN` | API token for authentication |
| `GITEA_URL` or `FORGEJO_URL` | Server URL (optional if in config) |

### Plugin Options

```json
{
  "plugins": [
    ["@pcc/semantic-release-gitea", {
      "giteaUrl": "https://forgejo.example.com",
      "assets": [
        { "path": "dist/*.js", "label": "JavaScript files" }
      ]
    }]
  ]
}
```

## Compatibility

- **Gitea:** 1.17+
- **Forgejo:** 1.0+ (7.0+ recommended)
- **Node.js:** 20+
- **semantic-release:** 24+
```

---

## Migration Checklist

- [ ] Fork repository setup
- [ ] Update package.json with new dependencies
- [ ] Convert to ESM (`"type": "module"`)
- [ ] Replace `got` with native `fetch`
- [ ] Replace `fs-extra` with `fs/promises`
- [ ] Replace `url-join` with native `URL`
- [ ] Replace `querystring` with `URLSearchParams`
- [ ] Add Forgejo environment variable support
- [ ] Migrate tests to Node.js test runner
- [ ] Update CI workflow
- [ ] Update documentation
- [ ] Publish to npm as `@pcc/semantic-release-gitea`
- [ ] Test with PCC Memoria Forgejo workflows

---

## Breaking Changes (v3.0.0)

1. **Node.js 20+ required** - Dropped support for Node.js < 20
2. **ESM only** - No more CommonJS support
3. **Package renamed** - `@saithodev/semantic-release-gitea` → `@pcc/semantic-release-gitea`
4. **semantic-release 24+ required** - Dropped support for v17-v23

## Publishing to npm

### 1. Create npm Account & Login

```bash
# Create account at https://www.npmjs.com/signup
# Then login from CLI
npm login
```

### 2. Configure package.json

Use a scoped package name:

```json
{
  "name": "@pcc/semantic-release-gitea",
  "publishConfig": {
    "access": "public"
  }
}
```

### 3. Get npm Token for CI/CD

1. Go to https://www.npmjs.com/settings/YOUR_USERNAME/tokens
2. Click "Generate New Token" → "Classic Token"
3. Select "Automation" (for CI)
4. Copy the token and add it as `NPM_TOKEN` secret in your repository

### 4. Manual Publishing

```bash
# Check what will be published
npm pack --dry-run

# Publish (first time or after version bump)
npm publish --access public

# Publish beta/prerelease
npm publish --tag beta --access public

# Verify after publishing
npm info @pcc/semantic-release-gitea
```

### 5. Automated Publishing (CI/CD)

Add this job to `.github/workflows/ci.yml`:

```yaml
release:
  needs: test
  runs-on: ubuntu-latest
  if: github.ref == 'refs/heads/main'

  steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - uses: actions/setup-node@v4
      with:
        node-version: 22
        cache: 'npm'

    - run: npm ci

    - name: Semantic Release
      run: npx semantic-release
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

Add `.releaserc.json` to the repository:

```json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    "@semantic-release/npm",
    "@semantic-release/github",
    [
      "@semantic-release/git",
      {
        "assets": ["CHANGELOG.md", "package.json"],
        "message": "chore(release): ${nextRelease.version} [skip ci]"
      }
    ]
  ]
}
```

---

## Sources

- [saitho/semantic-release-gitea](https://github.com/saitho/semantic-release-gitea) - Original repository
- [semantic-release plugins](https://semantic-release.gitbook.io/semantic-release/extending/plugins-list)
- [@semantic-release/github](https://github.com/semantic-release/github) - Reference implementation
- [npm publishing docs](https://docs.npmjs.com/packages-and-modules/contributing-packages-to-the-registry)
