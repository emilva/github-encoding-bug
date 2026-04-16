# GitHub API %2F Encoding Bug

Proof of concept demonstrating that GitHub's REST API no longer decodes `%2F` (percent-encoded forward slash) back to `/` in URL path segments.

## The bug

Around April 8-9 2026, GitHub's API changed how it handles `%2F` in URL paths. Previously, `%2F` was decoded to `/` when matching resources. Now `%2F` is treated as literal characters.

This breaks any client that correctly percent-encodes path parameters per RFC 3986, including:

- **@octokit/rest** - `repos.getContent({ path: '.github/CODEOWNERS' })` sends `.github%2FCODEOWNERS` via RFC 6570 simple expansion `{path}`
- **gh CLI** - `gh api "repos/.../environments/$encoded_name/secrets"` where the name was encoded with `%2F`
- **Any HTTP client** following RFC 3986 URL encoding rules

## Impact

### Contents API

- `getContent({ path: '.github/CODEOWNERS' })` returns 404
- `createOrUpdateFileContents({ path: '.github/CODEOWNERS' })` creates a file literally named `.github%2FCODEOWNERS` in the repo root

### Environments API

- Creating an environment via `%2F`-encoded name creates a duplicate with `%2F` in its name
- Looking up an environment via `%2F`-encoded name returns 404

## Run the PoC

1. Go to [Actions](../../actions) > "Reproduce: GitHub API %2F encoding bug"
2. Click "Run workflow"
3. Check the logs for each test step

## Workaround

### Octokit

Use `github.request()` with RFC 6570 reserved expansion `{+path}` instead of `github.rest.repos.*`:

```javascript
// Broken: {path} encodes / as %2F
await github.rest.repos.getContent({ owner, repo, path: '.github/CODEOWNERS' });

// Working: {+path} preserves /
await github.request('GET /repos/{owner}/{repo}/contents/{+path}', {
  owner, repo, path: '.github/CODEOWNERS'
});
```

### Shell / gh CLI

Don't encode `/` when URL-encoding path parameters:

```bash
# Broken: safe='' encodes / as %2F
encoded=$(python3 -c "import sys, urllib.parse; print(urllib.parse.quote(sys.stdin.read(), safe=''))" <<< "$name")

# Working: safe='/' preserves /
encoded=$(python3 -c "import sys, urllib.parse; print(urllib.parse.quote(sys.stdin.read(), safe='/'))" <<< "$name")
```
