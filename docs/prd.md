---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
inputDocuments:
  - docs/analysis/brainstorming-session-2025-12-02.md
  - docs/adr/001-node22-esm-native-apis.md
  - docs/index.md
  - docs/architecture.md
  - docs/project-overview.md
  - semantic-release-gitea-modernization.md
workflowType: 'prd'
lastStep: 11
project_name: 'semantic-release-gitea'
user_name: 'Joseybv'
date: '2025-12-02'
---

# Product Requirements Document - semantic-release-gitea

**Author:** Joseybv
**Date:** 2025-12-02

## Executive Summary

This project modernizes the abandoned `@saithodev/semantic-release-gitea` plugin (v2.1.0, last updated April 2021) into a modern, Forgejo-first release automation tool published as `@pcc/semantic-release-gitea`.

The modernization addresses critical gaps in the original library:
- **Outdated runtime**: Node.js 10/11/12/14+ → Node.js 22+
- **Legacy module system**: CommonJS → ESM
- **Heavy dependencies**: 6 external packages → Native Node.js APIs
- **Platform support**: Gitea-only → Forgejo-first with Gitea compatibility

This is a spiritual successor, not a drop-in replacement. Users of the original package will need to migrate to the new `@pcc` scoped package with updated Node.js requirements.

### What Makes This Special

**Forgejo-First Design**: While maintaining Gitea compatibility, this fork prioritizes Forgejo as the primary platform - supporting both `FORGEJO_TOKEN`/`FORGEJO_URL` and `GITEA_TOKEN`/`GITEA_URL` environment variables.

**Zero-Dependency Core**: By leveraging Node.js 22+ native APIs, we eliminate 6 external dependencies:
- `got` → Native `fetch()`
- `fs-extra` → Native `fs/promises`
- `globby` → Native `fs.glob()`
- `aggregate-error` → Native `AggregateError`
- `url-join` → Native `URL` class
- `querystring` → Native `URLSearchParams`

**Modern JavaScript**: Full ESM adoption with `"type": "module"`, aligning with the JavaScript ecosystem's direction and enabling cleaner, more maintainable code.

## Project Classification

**Technical Type:** Developer Tool (npm library / semantic-release plugin)
**Domain:** General (CI/CD tooling)
**Complexity:** Low (standard software practices)

This is a library that implements the semantic-release plugin contract, providing three lifecycle hooks (`verifyConditions`, `publish`, `addChannel`) to automate release publishing to Forgejo/Gitea instances. The project follows standard open-source practices with no domain-specific regulatory or compliance requirements.

## Success Criteria

### User Success

Developers integrating this plugin into their CI/CD pipelines will consider it successful when:

- **Seamless integration**: Plugin works out of the box with semantic-release v24+ and Forgejo
- **Clear configuration**: Environment variables (`FORGEJO_TOKEN`, `FORGEJO_URL`) and plugin options are intuitive
- **Reliable releases**: Releases are created consistently with correct tags, notes, and assets
- **Actionable errors**: When something fails, error messages clearly explain what went wrong and how to fix it
- **Easy migration**: Developers familiar with `@saithodev/semantic-release-gitea` can migrate with minimal friction

### Business Success

As an internal PCC tool contributed to the Forgejo community:

- **Internal adoption**: Successfully used in PCC's Forgejo-based release workflows
- **Quality over popularity**: A well-tested, maintainable library that solves the problem correctly
- **Community contribution**: Available for Forgejo users who need semantic-release integration
- **Maintenance sustainability**: Minimal dependency footprint reduces long-term maintenance burden

### Technical Success

- **All tests pass**: Comprehensive test coverage for all plugin hooks
- **Native APIs work**: Successfully replace 6 dependencies with Node.js 22+ native APIs
- **ESM migration complete**: Full ES module adoption with no CommonJS remnants
- **CI/CD operational**: GitHub Actions workflow runs tests on Node 22 and 24
- **Published to npm**: Package available as `@pcc/semantic-release-forgejo`

### Measurable Outcomes

| Metric | Target |
|--------|--------|
| Dependencies removed | 6 (got, fs-extra, globby, aggregate-error, url-join, querystring) |
| Node.js versions supported | 22+ |
| Test coverage | Maintain or improve existing coverage |
| Plugin hooks working | 3/3 (verifyConditions, publish, addChannel) |
| Platform support | Forgejo (primary), Gitea (compatible) |

## Product Scope

### MVP - Minimum Viable Product

The MVP delivers a fully functional semantic-release plugin for Forgejo:

1. **Core Plugin Functionality**
   - `verifyConditions`: Validate Forgejo authentication and configuration
   - `publish`: Create releases with release notes and optional asset uploads
   - `addChannel`: Update release prerelease status

2. **Modernization Complete**
   - Node.js 22+ with ESM
   - Native `fetch()` replacing `got`
   - Native `fs.glob()` replacing `globby`
   - Native `AggregateError` replacing `aggregate-error`
   - Native `fs/promises` replacing `fs-extra`
   - Native `URL`/`URLSearchParams` replacing `url-join`/`querystring`

3. **Forgejo Support**
   - `FORGEJO_TOKEN` and `FORGEJO_URL` environment variables
   - Backward-compatible `GITEA_TOKEN` and `GITEA_URL` support
   - Tested against PCC's Forgejo instance

4. **Quality Assurance**
   - Migrated tests using `node:test`
   - Coverage with `c8`
   - CI with GitHub Actions (Node 22, 24)

5. **Published**
   - Available on npm as `@pcc/semantic-release-forgejo`
   - Updated README with Forgejo-first documentation

### Growth Features (Post-MVP)

- TypeScript type declarations (`.d.ts`) for better DX
- Enhanced debug logging
- Additional Forgejo-specific features as the API diverges from Gitea

### Vision (Future)

- Community contributions and feedback integration
- Support for Forgejo-specific release features as they emerge
- Potential collaboration with Forgejo project for official plugin status

## User Journeys

### Journey 1: Carlos - The DevOps Engineer Setting Up Automated Releases

Carlos manages CI/CD for a small team using Forgejo for their self-hosted Git. They've been manually creating releases - tagging, writing release notes, uploading build artifacts. It's tedious and error-prone. He's heard about semantic-release but the only Gitea plugin he found hasn't been updated since 2021 and doesn't mention Forgejo.

One afternoon, while searching "semantic-release forgejo", Carlos discovers `@pcc/semantic-release-forgejo`. The README immediately catches his attention: "Forgejo-first, Node.js 22+, zero dependencies replaced with native APIs." This is exactly what he needs.

He runs `npm install @pcc/semantic-release-forgejo -D`, sets `FORGEJO_TOKEN` and `FORGEJO_URL` in his CI secrets, and adds a simple `.releaserc.json`. On his next commit with `feat:` prefix, the pipeline runs, and within minutes, a beautifully formatted release appears in Forgejo - complete with changelog and his build artifacts attached.

The breakthrough comes when his team realizes they haven't manually created a release in weeks. Their release process is now: write code, write good commit messages, push. That's it. Carlos has automated himself out of release management, and he couldn't be happier.

### Journey 2: Sofia - The Developer Debugging a Failed Release

Sofia is a developer on Carlos's team. One morning, her feature branch gets merged to main, but the release pipeline fails. She checks the CI logs and sees a clear error from `@pcc/semantic-release-forgejo`:

```
ENOFORGEJOPERMISSION: The Forgejo token doesn't have push access to owner/repo
```

The error message includes a link to the documentation and suggests checking token permissions. Sofia realizes she accidentally used a read-only token when rotating credentials last week. She generates a new token with proper permissions, updates the CI secret, and re-runs the pipeline. The release publishes successfully.

Sofia appreciates that the error didn't just say "403 Forbidden" - it told her exactly what was wrong and how to fix it.

### Journey Requirements Summary

These journeys reveal the following capability requirements:

| Capability | Source Journey | Requirement |
|------------|----------------|-------------|
| **Easy Installation** | Carlos | Single npm install, minimal configuration |
| **Clear Documentation** | Carlos | README with quick start, environment variables |
| **Forgejo Environment Variables** | Carlos | `FORGEJO_TOKEN`, `FORGEJO_URL` support |
| **Asset Upload** | Carlos | Attach build artifacts to releases |
| **Automatic Release Notes** | Carlos | Generate changelog from commits |
| **Actionable Error Messages** | Sofia | Clear error codes with fix suggestions |
| **Permission Validation** | Sofia | `verifyConditions` checks token permissions upfront |
| **Discoverability** | Carlos | Package name and description optimized for search |

## Developer Tool Requirements

### Installation & Distribution

| Aspect | Requirement |
|--------|-------------|
| **Package Name** | `@pcc/semantic-release-forgejo` |
| **Registry** | npm (public) |
| **Package Managers** | npm, pnpm, yarn (standard Node.js package) |
| **Installation** | `npm install @pcc/semantic-release-forgejo -D` |

### API Surface

The plugin exports three semantic-release lifecycle hooks:

| Hook | Purpose | Forgejo API Used |
|------|---------|------------------|
| `verifyConditions` | Validate token and permissions | `GET /repos/{owner}/{repo}` |
| `publish` | Create release with optional assets | `POST /repos/{owner}/{repo}/releases`, `POST .../assets` |
| `addChannel` | Update release prerelease flag | `PATCH /repos/{owner}/{repo}/releases/{id}` |

### Configuration Interface

**Environment Variables:**

| Variable | Required | Description |
|----------|----------|-------------|
| `FORGEJO_TOKEN` | Yes* | Forgejo API token with push permissions |
| `FORGEJO_URL` | Yes* | Forgejo instance URL |
| `GITEA_TOKEN` | Yes* | Fallback: Gitea API token |
| `GITEA_URL` | Yes* | Fallback: Gitea instance URL |

*Either FORGEJO_* or GITEA_* variables must be set

**Plugin Options (.releaserc.json):**

```json
{
  "plugins": [
    ["@pcc/semantic-release-forgejo", {
      "giteaUrl": "https://forgejo.example.com",
      "giteaApiPathPrefix": "/api/v1",
      "assets": [
        {"path": "dist/*.js", "label": "JavaScript bundle"}
      ]
    }]
  ]
}
```

### Documentation Requirements

| Document | Content | Priority |
|----------|---------|----------|
| **README.md** | Quick start, installation, configuration, examples | MVP |
| **CHANGELOG.md** | Auto-generated by semantic-release | MVP |
| **Error Codes** | All error codes with fix suggestions | MVP |
| **Migration Guide** | From `@saithodev/semantic-release-gitea` | MVP |

### Code Examples to Include

1. **Basic Configuration** - Minimal `.releaserc.json`
2. **With Asset Upload** - Attaching build artifacts
3. **GitHub Actions Workflow** - CI integration example
4. **Forgejo Actions Workflow** - Native Forgejo CI example

### Compatibility Matrix

| Dependency | Version | Notes |
|------------|---------|-------|
| Node.js | 22+ | Required for native `fs.glob()` |
| semantic-release | 24+ | Plugin API compatibility |
| Forgejo | 1.0+ | API v1 compatible |
| Gitea | 1.17+ | API v1 compatible (fallback) |

## Functional Requirements

### Authentication & Verification

- FR1: Plugin can validate Forgejo/Gitea API token before release
- FR2: Plugin can verify user has push permissions to target repository
- FR3: Plugin can authenticate using `FORGEJO_TOKEN` environment variable
- FR4: Plugin can fallback to `GITEA_TOKEN` if `FORGEJO_TOKEN` is not set
- FR5: Plugin can resolve server URL from `FORGEJO_URL` or `GITEA_URL` environment variables
- FR6: Plugin can accept server URL from plugin configuration options

### Release Management

- FR7: Plugin can create a new release on Forgejo/Gitea
- FR8: Plugin can set release tag name from semantic-release context
- FR9: Plugin can set release name from semantic-release context
- FR10: Plugin can set release body (notes) from semantic-release context
- FR11: Plugin can mark release as prerelease based on branch configuration
- FR12: Plugin can update existing release prerelease status via `addChannel` hook

### Asset Management

- FR13: Plugin can upload file assets to a release
- FR14: Plugin can resolve asset file paths using glob patterns
- FR15: Plugin can set custom names for uploaded assets
- FR16: Plugin can set custom labels for uploaded assets
- FR17: Plugin can handle multiple assets per release

### Error Handling

- FR18: Plugin can return specific error codes for different failure scenarios
- FR19: Plugin can provide actionable error messages with fix suggestions
- FR20: Plugin can aggregate multiple validation errors into single response
- FR21: Plugin can distinguish between authentication, permission, and API errors

### Configuration

- FR22: Plugin can accept custom API path prefix configuration
- FR23: Plugin can merge environment variables with plugin options
- FR24: Plugin can parse repository URL to extract owner and repo name

### Integration

- FR25: Plugin can export `verifyConditions` hook for semantic-release
- FR26: Plugin can export `publish` hook for semantic-release
- FR27: Plugin can export `addChannel` hook for semantic-release
- FR28: Plugin can return release URL after successful publish

## Non-Functional Requirements

### Security

- NFR1: API tokens must never be logged or exposed in output
- NFR2: All API communications must use HTTPS
- NFR3: Token validation must fail securely (no information leakage about why token is invalid)

### Integration

- NFR4: Plugin must implement semantic-release plugin contract v24+
- NFR5: Plugin must use Forgejo/Gitea API v1 endpoints
- NFR6: Plugin must handle API rate limiting gracefully
- NFR7: Plugin must support standard Forgejo/Gitea authentication schemes

### Reliability

- NFR8: Plugin must provide clear error messages for all failure scenarios
- NFR9: Plugin must fail fast during `verifyConditions` rather than during `publish`
- NFR10: Plugin must handle network timeouts gracefully
- NFR11: Asset uploads must not leave partial releases (draft until complete)

### Maintainability

- NFR12: Codebase must use ES modules (ESM) exclusively
- NFR13: All dependencies must be actively maintained or native Node.js APIs
- NFR14: Test coverage must be maintained at current levels or improved
- NFR15: Code must pass XO linting rules

### Compatibility

- NFR16: Plugin must run on Node.js 22 and later LTS versions
- NFR17: Plugin must work with npm, pnpm, and yarn package managers
- NFR18: Plugin must support both Forgejo and Gitea instances
