# GitHub API %2F Encoding Bug

Proof of concept demonstrating that GitHub's REST API no longer decodes `%2F` (percent-encoded forward slash) back to `/` in URL path segments.

## The bug

Around April 8-9 2026, GitHub's API changed how it handles `%2F` in URL paths. Previously, `%2F` was decoded to `/` when matching resources. Now `%2F` is treated as literal characters.

This breaks any client that correctly percent-encodes path parameters per RFC 3986, including:

- **@octokit/rest** - `repos.getContent({ path: '.github/CODEOWNERS' })` sends `.github%2FCODEOWNERS` via RFC 6570 simple expansion `{path}`
- **gh CLI** - `gh api "repos/.../contents/.github%2FCODEOWNERS"` when the path is percent-encoded
- **Any HTTP client** following RFC 3986 URL encoding rules

## Impact on Contents API

- `getContent({ path: '.github/CODEOWNERS' })` returns **404** because the server looks for a file literally named `.github%2FCODEOWNERS`
- `createOrUpdateFileContents({ path: '.github/CODEOWNERS' })` creates a **root-level file** named `.github%2FCODEOWNERS` instead of `.github/CODEOWNERS`
- Any Octokit REST method using `{path}` expansion with paths containing `/` is affected

## Run the PoC

1. Go to [Actions](../../actions) > "Reproduce: GitHub API %2F encoding bug"
2. Click "Run workflow"
3. Check the logs for each test step

The workflow runs two jobs:

| Job | What it tests |
|-----|---------------|
| **Octokit** | `repos.getContent()` and `createOrUpdateFileContents()` with `{path}` vs `{+path}` |
| **gh CLI** | `gh api` with `%2F`-encoded vs unencoded paths |

## Workaround

Use `github.request()` with RFC 6570 reserved expansion `{+path}` instead of `github.rest.repos.*`:

```javascript
// Broken: {path} encodes / as %2F
await github.rest.repos.getContent({ owner, repo, path: '.github/CODEOWNERS' });

// Working: {+path} preserves /
await github.request('GET /repos/{owner}/{repo}/contents/{+path}', {
  owner, repo, path: '.github/CODEOWNERS'
});
```
