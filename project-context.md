# Project Context: semantic-release-gitea

> **For AI Agents:** Read this file before implementing any code. These rules ensure consistency across all development work.

## Project Overview

**What:** Modernizing `@saithodev/semantic-release-gitea` into `@pcc/semantic-release-forgejo`
**Type:** Node.js library (semantic-release plugin)
**Status:** Modernization in progress

## Critical Rules

### Runtime & Module System

```
Node.js: 22+ REQUIRED (for fs.glob())
Module System: ESM only ("type": "module")
No CommonJS: Never use require() or module.exports
```

### ESM Import Order

Always organize imports in this order:

```javascript
// 1. Node built-ins (with node: prefix)
import { readFile } from 'node:fs/promises';
import { glob } from 'node:fs';

// 2. External packages
import SemanticReleaseError from '@semantic-release/error';
import FormData from 'form-data';
import debug from 'debug';

// 3. Local modules (MUST include .js extension)
import { resolveConfig } from './resolve-config.js';
import { getError } from './get-error.js';
```

### Environment Variable Precedence

**Forgejo-first, always:**

```javascript
// CORRECT - Forgejo takes priority
const token = env.FORGEJO_TOKEN || env.GITEA_TOKEN || pluginConfig.giteaToken;
const url = env.FORGEJO_URL || env.GITEA_URL || pluginConfig.giteaUrl;

// WRONG - Never check Gitea first
const token = env.GITEA_TOKEN || env.FORGEJO_TOKEN; // ❌
```

### Error Handling

**Generic codes, platform-specific messages:**

```javascript
// Error codes are platform-agnostic
const ERROR_CODES = {
  ENOTOKEN: 'Missing authentication token',
  ENOURL: 'Missing server URL',
  ENOPERMISSION: 'Token lacks required permissions',
  EINVALIDTOKEN: 'Token is invalid or expired',
  EMISSINGREPO: 'Repository not found',
  EAPIERROR: 'Generic API error',
  EINVALIDASSETS: 'Invalid assets configuration'
};

// Messages include detected platform
const platform = env.FORGEJO_TOKEN ? 'Forgejo' : 'Gitea';
getError('ENOPERMISSION', { platform });
// → "The Forgejo token doesn't have push access to owner/repo"
```

### fetch() Pattern

**Centralized error detection in get-client.js:**

```javascript
async _makeRequest(method, path, body) {
  const response = await fetch(url, {
    method,
    body: body ? JSON.stringify(body) : undefined,
    headers: {
      'Authorization': `token ${this.token}`,
      'Content-Type': 'application/json',
    },
  });

  if (!response.ok) {
    const error = new Error(`API error: ${response.status}`);
    error.status = response.status;
    error.response = { statusCode: response.status }; // Compatibility shim
    throw error;
  }

  return response.json();
}
```

### fs.glob() Pattern

**Use Array.fromAsync() for async iterators:**

```javascript
// CORRECT
const files = await Array.fromAsync(
  glob(patterns, { cwd, withFileTypes: false })
);

// WRONG - glob returns AsyncIterator, not Promise<Array>
const files = await glob(patterns); // ❌
```

### URL Construction

**Use native URL class with normalized base:**

```javascript
// Normalize base URL once in resolve-config.js
const normalizedUrl = giteaUrl.endsWith('/') ? giteaUrl : `${giteaUrl}/`;

// Construct API URLs in get-client.js
const url = new URL(`repos/${owner}/${repo}`, this.baseUrl);
```

## Dependencies

### Remove (replace with native APIs)

| Package | Replacement |
|---------|-------------|
| `got` | `fetch()` |
| `fs-extra` | `fs/promises` |
| `globby` | `fs.glob()` |
| `aggregate-error` | `AggregateError` |
| `url-join` | `URL` class |
| `querystring` | `URLSearchParams` |
| `lodash` | Native JS |

### Keep

| Package | Reason |
|---------|--------|
| `@semantic-release/error` | Plugin contract |
| `form-data` | File upload reliability |
| `debug` | Standard plugin pattern |

## Testing

```
Framework: node:test (NOT AVA)
Coverage: c8 (NOT nyc)
Mocking: nock 14+ (supports native fetch)
```

**Test file pattern:**
```javascript
import { test, describe } from 'node:test';
import assert from 'node:assert';
import nock from 'nock';

describe('verifyConditions', () => {
  test('validates token permissions', async () => {
    nock('https://forgejo.example.com')
      .get('/api/v1/repos/owner/repo')
      .reply(200, { permissions: { push: true } });

    // ... test implementation
  });
});
```

## Anti-Patterns

```javascript
// ❌ Wrong import order
import { getError } from './get-error.js';
import { readFile } from 'node:fs/promises';

// ❌ Missing .js extension
import { resolveConfig } from './resolve-config';

// ❌ Wrong env var precedence
const token = env.GITEA_TOKEN || env.FORGEJO_TOKEN;

// ❌ Platform-specific error codes
throw getError('EFORGEJOPERMISSION');

// ❌ Not checking response.ok
const data = await fetch(url).then(r => r.json()); // May silently fail

// ❌ Using require()
const fs = require('fs'); // CommonJS not allowed
```

## File Structure

```
index.js           → Plugin entry (exports 3 hooks)
lib/
├── verify.js      → verifyConditions implementation
├── publish.js     → publish implementation
├── add-channel.js → addChannel implementation
├── get-client.js  → GiteaClient (fetch-based) ← MAJOR CHANGE
├── resolve-config.js → Config resolution (Forgejo-first) ← MODERATE CHANGE
├── glob-assets.js → Asset matching (fs.glob) ← MAJOR CHANGE
├── parse-git-url.js → URL parsing
├── is-prerelease.js → Branch detection
├── get-error.js   → Error factory
└── definitions/
    ├── errors.js  → Error definitions ← MODERATE CHANGE
    └── constants.js → Constants
```

## Security

- **Never log tokens** - Not in errors, not in debug output
- **HTTPS only** - All API calls use HTTPS
- **Fail securely** - Don't leak token validity info in errors

---

**Reference Documents:**
- `docs/prd.md` - Product requirements
- `docs/architecture-decision.md` - Full architecture decisions
- `docs/adr/001-node22-esm-native-apis.md` - Platform decision rationale
