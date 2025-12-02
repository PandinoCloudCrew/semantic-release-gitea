# Development Guide

## Project: @saithodev/semantic-release-gitea

**Generated:** 2025-12-02

---

## Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Node.js | ^10, ^11, ^12, >=14 | See `.nvmrc` for recommended (12.13.1) |
| npm | 6.x+ | Comes with Node.js |
| Git | 2.x+ | For version control |

---

## Quick Start

```bash
# Clone the repository
git clone https://github.com/saitho/semantic-release-gitea.git
cd semantic-release-gitea

# Use correct Node version (if using nvm)
nvm use

# Install dependencies
npm install

# Run tests
npm test

# Run linting
npm run lint
```

---

## Available Scripts

| Command | Description |
|---------|-------------|
| `npm test` | Run test suite with coverage (`nyc ava -v`) |
| `npm run lint` | Run XO linter (ESLint-based) |
| `npm run commit` | Interactive commit with Commitizen |
| `npm run codecov` | Upload coverage to Codecov |
| `npm run semantic-release` | Trigger semantic-release (CI only) |

---

## Environment Setup

### Required for Development

No environment variables required for local development or testing. The test suite uses mocked HTTP calls via `nock`.

### Required for Integration Testing (Optional)

If you want to test against a real Gitea instance:

| Variable | Description |
|----------|-------------|
| `GITEA_URL` | Your Gitea instance URL (e.g., `https://try.gitea.io`) |
| `GITEA_TOKEN` | Personal access token from Gitea |
| `GITEA_PREFIX` | API prefix (default: `/api/v1`) |

---

## Testing

### Test Framework

- **AVA** - Concurrent test runner
- **nock** - HTTP mocking for Gitea API
- **sinon** - Mocking and spying
- **nyc** - Code coverage (Istanbul)

### Running Tests

```bash
# Run all tests with coverage
npm test

# Run specific test file
npx ava test/verify.test.js -v

# Run tests matching pattern
npx ava test/*.test.js -m "Publish a release"

# Watch mode (during development)
npx ava --watch
```

### Test Structure

```
test/
├── verify.test.js        # 18 tests - auth & config validation
├── publish.test.js       # ~10 tests - release creation
├── add-channel.test.js   # ~8 tests - release updates
├── integration.test.js   # 9 tests - end-to-end flows
├── glob-assets.test.js   # ~12 tests - asset glob patterns
├── helpers/
│   └── _mock-gitea.js    # Nock helpers for API mocking
└── fixtures/
    └── files/            # Sample files for upload tests
```

### Coverage

Coverage is configured via `nyc` in `package.json`:

```json
{
  "nyc": {
    "include": ["lib/**/*.js", "index.js"],
    "reporter": ["lcov", "text", "html"],
    "all": true
  }
}
```

View HTML coverage report: `open coverage/index.html`

---

## Code Style

### Linting

This project uses **XO** (opinionated ESLint wrapper) with Prettier:

```json
{
  "xo": {
    "prettier": true,
    "space": true,
    "rules": {
      "camelcase": ["error", {"properties": "never"}]
    }
  }
}
```

```bash
# Check linting
npm run lint

# Fix auto-fixable issues
npx xo --fix
```

### Conventions

- **Indentation**: 2 spaces
- **Quotes**: Single quotes
- **Semicolons**: Yes
- **Trailing commas**: ES5 style
- **camelCase**: Required (except object properties from API)

---

## Commit Guidelines

This project uses **Commitizen** with **Conventional Commits**:

```bash
# Use interactive commit
npm run commit

# Or manually follow format:
# <type>(<scope>): <subject>
#
# Types: feat, fix, docs, style, refactor, test, chore
```

### Setting Up Commit Hook (Recommended)

Create `.git/hooks/prepare-commit-msg`:

```bash
#!/bin/bash
exec < /dev/tty && node_modules/.bin/git-cz --hook || true
```

Then: `chmod +x .git/hooks/prepare-commit-msg`

---

## Debugging

### Debug Logging

The library uses the `debug` package:

```bash
# Enable debug output
DEBUG=saithodev:semantic-release-gitea npm test

# Or when using the plugin
DEBUG=saithodev:semantic-release-gitea npx semantic-release
```

### Common Debug Points

- `lib/publish.js` - Release object creation, asset globbing
- `lib/get-client.js` - HTTP request details

---

## CI/CD Pipeline

### GitHub Actions

**test.yml** - Runs on every push/PR:
- Node.js matrix: 10, 11, 12, 14, 15
- Runs `npm install` and `npm test`
- SonarCloud analysis

**release.yml** - Runs after tests pass on master:
- Triggers semantic-release
- Publishes to npm
- Creates GitHub release

### Local Simulation

```bash
# Dry-run semantic-release
npx semantic-release --dry-run

# Check what would be released
npx semantic-release --dry-run --no-ci
```

---

## Architecture Notes

### Plugin Interface

This is a semantic-release plugin implementing three hooks:

| Hook | When Called | Purpose |
|------|-------------|---------|
| `verifyConditions` | Before release | Validate config & auth |
| `publish` | After tagging | Create Gitea release |
| `addChannel` | Channel update | Update release prerelease flag |

### Module Dependencies

```
index.js (entry)
    ├── lib/verify.js
    ├── lib/publish.js
    └── lib/add-channel.js
            │
            └── lib/get-client.js (GiteaClient)
                    │
                    └── Gitea REST API
```

---

## Troubleshooting

### Tests Failing with Network Errors

All HTTP calls should be mocked. If you see network errors:
1. Check that `nock.cleanAll()` is called in `afterEach`
2. Ensure mock matches the exact URL pattern

### Coverage Not Updating

```bash
# Clear nyc cache
rm -rf .nyc_output coverage
npm test
```

### Commitizen Not Working

```bash
# Reinstall
rm -rf node_modules
npm install
npm run commit
```
