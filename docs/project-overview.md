# Project Overview

## @saithodev/semantic-release-gitea

**Generated:** 2025-12-02
**Version:** 2.1.0
**License:** MIT
**Author:** Mario Lubenka

---

## Summary

A [semantic-release](https://github.com/semantic-release/semantic-release) plugin that publishes releases to Gitea instances. This plugin automates the creation of Gitea releases with optional file asset uploads as part of your CI/CD pipeline.

Based on the official [semantic-release GitHub plugin](https://github.com/semantic-release/github).

---

## Quick Reference

| Attribute | Value |
|-----------|-------|
| **Package Name** | `@saithodev/semantic-release-gitea` |
| **Type** | npm Library (semantic-release plugin) |
| **Language** | JavaScript (Node.js) |
| **Node Versions** | ^10, ^11, ^12, >=14 |
| **Entry Point** | `index.js` |
| **Repository** | https://github.com/saitho/semantic-release-gitea |

---

## Features

- **Automated Release Publishing** - Creates Gitea releases with release notes from commits
- **Asset Upload** - Attach build artifacts to releases using glob patterns
- **Prerelease Support** - Automatic prerelease flag based on branch type
- **Channel Management** - Update existing releases via `addChannel` hook
- **Template Support** - Dynamic asset names/labels using Lodash templates
- **Comprehensive Error Handling** - User-friendly error messages with documentation links

---

## Plugin Lifecycle Hooks

| Hook | Description |
|------|-------------|
| `verifyConditions` | Validates Gitea authentication and configuration |
| `publish` | Creates a Gitea release, optionally uploads file assets |
| `addChannel` | Updates a release's prerelease field |

---

## Tech Stack Summary

| Category | Technologies |
|----------|-------------|
| **Runtime** | Node.js |
| **HTTP** | got |
| **Testing** | AVA, nock, sinon, nyc |
| **Linting** | XO (ESLint + Prettier) |
| **CI/CD** | GitHub Actions, semantic-release |
| **Code Quality** | SonarCloud |

---

## Project Structure

```
semantic-release-gitea/
├── index.js              # Entry point (3 exports)
├── lib/                  # Core library (11 modules)
├── test/                 # Test suite (5 test files)
├── .github/workflows/    # CI/CD pipelines
└── docs/                 # Generated documentation
```

---

## Documentation Index

| Document | Description |
|----------|-------------|
| [Architecture](./architecture.md) | System design, data flows, API |
| [Source Tree Analysis](./source-tree-analysis.md) | Directory structure, module map |
| [Development Guide](./development-guide.md) | Setup, testing, contributing |
| [README](../README.md) | Usage guide, configuration |
| [CONTRIBUTING](../CONTRIBUTING.md) | Contribution guidelines |
| [CHANGELOG](../CHANGELOG.md) | Release history |

---

## Getting Started

### Installation

```bash
npm install @saithodev/semantic-release-gitea -D
```

### Configuration

```json
{
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@saithodev/semantic-release-gitea", {
      "giteaUrl": "https://your-gitea.example.com",
      "assets": [
        {"path": "dist/**/*.js", "label": "JavaScript bundle"}
      ]
    }]
  ]
}
```

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `GITEA_URL` | Yes | Gitea instance URL |
| `GITEA_TOKEN` | Yes | Personal access token |
| `GITEA_PREFIX` | No | API prefix (default: `/api/v1`) |

---

## Project Status

| Metric | Status |
|--------|--------|
| **Current Version** | 2.1.0 |
| **Last Release** | April 2021 |
| **Node Support** | ^10, ^11, ^12, >=14 |
| **Test Coverage** | Configured (nyc) |
| **CI Status** | GitHub Actions |

---

## Related Links

- [npm Package](https://www.npmjs.com/package/@saithodev/semantic-release-gitea)
- [GitHub Repository](https://github.com/saitho/semantic-release-gitea)
- [semantic-release Documentation](https://semantic-release.gitbook.io/)
- [Gitea API Documentation](https://docs.gitea.io/en-us/api-usage/)
