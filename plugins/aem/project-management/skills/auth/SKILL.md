---
name: auth
description: Authenticate with AEM Edge Delivery Services. Opens browser for login and captures token. Works for admin.hlx.page, admin.da.live, and Config Service APIs regardless of content source (Document Authoring, SharePoint, or Google Drive).
license: Apache-2.0
allowed-tools: Read, Write, Edit, Bash, AskUserQuestion
metadata:
  version: "3.0.0"
---

# AEM Edge Delivery Services Authentication

Authenticate to obtain a Bearer token for all Edge Delivery Services admin operations. Works for all content sources (Document Authoring, SharePoint, Google Drive).

## Token Usage

The token works for all admin APIs:

| API | Usage |
|-----|-------|
| `admin.hlx.page` | Preview, publish, status, code sync, jobs, logs |
| `admin.da.live` | DA content operations (list, source, copy, move) |
| Config Service | Sites, config, secrets, API keys, profiles |

**admin.hlx.page:** `-H "x-auth-token: ${AUTH_TOKEN}"`
**admin.da.live:** `-H "Authorization: Bearer ${IMS_TOKEN}"`

## When to Use This Skill

- API returns 401 Unauthorized
- User says "login", "authenticate", "auth"
- Before any admin operation when token is missing/expired
- Before generating guides that need API access

## Prerequisites

- Node.js installed
- Playwright installed (`npx playwright install chromium`)

---

## Authentication Flow

### Step 1: Check Existing Token

Tokens are cached at the **user level** (`~/.aem/ims-token.json`), not per-project — one auth covers every project on this machine that uses Edge Delivery Services. Only `org` / `site` remain per-project at `.claude-plugin/project-config.json`.

```bash
TOKEN_FILE="${HOME}/.aem/ims-token.json"
mkdir -p "${HOME}/.aem"

AUTH_TOKEN=$(node -e "
  const fs = require('fs');
  try {
    const t = JSON.parse(fs.readFileSync(process.env.HOME + '/.aem/ims-token.json', 'utf8'));
    if (t.authToken && t.authTokenExpiry > Math.floor(Date.now()/1000) + 60) {
      process.stdout.write(t.authToken);
    }
  } catch (e) {}
")

if [ -n "$AUTH_TOKEN" ]; then
  echo "Token valid"
  exit 0
fi

echo "Token missing or expired. Starting login..."
```

### Step 2: Install Playwright (if needed)

```bash
npx playwright --version 2>/dev/null || npm install -g playwright
npx playwright install chromium 2>/dev/null || true
```

### Step 2.5: Resolve Organization and Site

The login endpoint requires org and site to route to the correct identity provider. Check saved config:

```bash
ORG=$(node -e "
  const fs = require('fs');
  try {
    const c = JSON.parse(fs.readFileSync(process.env.HOME + '/.aem/ops-config.json', 'utf8'));
    process.stdout.write(c.org || '');
  } catch(e) {}
")
SITE=$(node -e "
  const fs = require('fs');
  try {
    const c = JSON.parse(fs.readFileSync(process.env.HOME + '/.aem/ops-config.json', 'utf8'));
    process.stdout.write(c.site || '');
  } catch(e) {}
")
# Fallback: check project-level config
if [ -z "$ORG" ]; then
  ORG=$(cat .claude-plugin/project-config.json 2>/dev/null | node -e "
    const d = require('fs').readFileSync(0,'utf8');
    try { process.stdout.write(JSON.parse(d).org || ''); } catch(e) {}
  ")
fi
echo "org=${ORG:-NOT SET} site=${SITE:-NOT SET}"
```

**If `ORG` or `SITE` is empty**, ask the user:

> "I need your organization and site name to authenticate. What is your org (the `{org}` in `https://main--site--{org}.aem.page`) and site name?"

**Do NOT proceed until both org and site are available.**

### Step 3: Capture Token via Playwright

Playwright opens browser to `admin.hlx.page/login/{org}/{site}` which routes to the correct identity provider based on the project's content source. After login completes, the token is stored as the `auth_token` cookie on `admin.hlx.page`. Playwright reads this cookie, saves the token to disk, then closes the browser automatically.

```bash
mkdir -p "${HOME}/.aem"

node -e "
const fs = require('fs');
const path = require('path');
const { chromium } = require('playwright');

const TOKEN_PATH = path.join(process.env.HOME, '.aem', 'ims-token.json');
const ORG = '${ORG}';
const SITE = '${SITE}';

const loginUrl = 'https://admin.hlx.page/login/' + ORG + '/' + SITE;

(async () => {
  console.log('Opening browser for login...');
  console.log('URL: ' + loginUrl);
  const browser = await chromium.launch({ headless: false });
  const context = await browser.newContext();
  const page = await context.newPage();
  await page.goto(loginUrl);

  // Poll for auth_token cookie after login completes
  let token = null;
  for (let i = 0; i < 60; i++) {
    await page.waitForTimeout(5000);
    const cookies = await context.cookies('https://admin.hlx.page');
    const authCookie = cookies.find(c => c.name === 'auth_token');
    if (authCookie && authCookie.value) {
      token = authCookie.value;
      break;
    }
  }

  if (token) {
    const expiresAt = Math.floor(Date.now() / 1000) + 86400;
    fs.mkdirSync(path.dirname(TOKEN_PATH), { recursive: true });
    // Merge with existing file to preserve imsToken (DA) if present
    let existing = {};
    try { existing = JSON.parse(fs.readFileSync(TOKEN_PATH, 'utf8')); } catch (e) {}
    existing.authToken = token;
    existing.authTokenExpiry = expiresAt;
    fs.writeFileSync(TOKEN_PATH, JSON.stringify(existing, null, 2));
    try { fs.chmodSync(TOKEN_PATH, 0o600); } catch (e) {}
    console.log('Authentication successful');
    console.log('Token cached at: ' + TOKEN_PATH);
    console.log('Expires: ' + new Date(expiresAt * 1000).toISOString());
  } else {
    console.error('Login timed out - no auth_token cookie found');
  }

  await browser.close();
  process.exit(token ? 0 : 1);
})();
"
```

---

## Token Storage

**User-level token cache** — `~/.aem/ims-token.json`:

```json
{
  "authToken": "eyJ...",
  "authTokenExpiry": 1780489855,
  "imsToken": "eyJ...",
  "imsTokenExpiry": 1777891272
}
```

| Field | Description | Used by |
|-------|-------------|---------|
| `authToken` | Token from `admin.hlx.page/login` | All `admin.hlx.page` operations (cookie auth) |
| `authTokenExpiry` | Unix timestamp when authToken expires | |
| `imsToken` | Adobe IMS OAuth token (DA only) | `admin.da.live` operations (Bearer auth) |
| `imsTokenExpiry` | Unix timestamp when imsToken expires | |

Shared across every project on this machine. File is written with `0600` permissions.

**Project-level config** — `.claude-plugin/project-config.json` (per-project, gitignored):

```json
{
  "org": "myorg",
  "site": "mysite"
}
```

Holds only project context. **No token fields.**

---

## Using the Token

```bash
# For admin.hlx.page operations (all content sources)
AUTH_TOKEN=$(node -e "
  const fs = require('fs');
  try {
    const t = JSON.parse(fs.readFileSync(process.env.HOME + '/.aem/ims-token.json', 'utf8'));
    process.stdout.write(t.authToken || '');
  } catch (e) {}
")
curl -H "x-auth-token: ${AUTH_TOKEN}" "https://admin.hlx.page/status/{org}/{site}/main/"
curl -H "x-auth-token: ${AUTH_TOKEN}" "https://admin.hlx.page/config/{org}/sites.json"

# For admin.da.live operations (DA content only, requires separate IMS login)
IMS_TOKEN=$(node -e "
  const fs = require('fs');
  try {
    const t = JSON.parse(fs.readFileSync(process.env.HOME + '/.aem/ims-token.json', 'utf8'));
    process.stdout.write(t.imsToken || '');
  } catch (e) {}
")
curl -H "Authorization: Bearer ${IMS_TOKEN}" "https://admin.da.live/list/{org}/{site}"
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `npx playwright` not found | Run `npm install -g playwright` |
| Browser doesn't open | Run `npx playwright install chromium` |
| Login page doesn't load | Check network connectivity |
| Token not captured | Ensure login completed before closing browser |
| 401 after login | Token expired, re-authenticate |
| 403 on API | User lacks permission for that org/site |

---

## Integration

Called by: `ops`, `handover-admin`, `handover-author`, `handover-developer`, `handover`

```
Skill({ skill: "project-management:auth" })
```
