---
name: auth
description: Authenticate with AEM Edge Delivery Services. Opens browser for login and captures token. Works for admin.hlx.page and Config Service APIs regardless of content source (Document Authoring, SharePoint, or Google Drive).
license: Apache-2.0
allowed-tools: Read, Write, Edit, Bash, AskUserQuestion
metadata:
  version: "4.0.0"
---

# AEM Edge Delivery Services Authentication

Authenticate to obtain a token for all Edge Delivery Services admin operations. Auto-detects identity provider from org+site — no content source question needed.

## Token Usage

The `auth_token` cookie works for admin APIs:

| API | Header | Usage |
|-----|--------|-------|
| `admin.hlx.page` | `x-auth-token: ${AUTH_TOKEN}` | Preview, publish, status, code sync, jobs, logs, config |
| Config Service | `x-auth-token: ${AUTH_TOKEN}` | Sites, config, secrets, API keys, profiles |

> **Note:** `admin.da.live` uses a separate Adobe IMS token with `Authorization: Bearer` header. See the DA-specific login flow in `ops/resources/da.md`.

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

Tokens are cached at the **user level** (`~/.aem/ims-token.json`), shared across all projects.

```bash
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

### Step 2.5: Resolve Org and Site

The auto-login endpoint `/login/{org}/{site}/main` redirects to the correct identity provider automatically — no need to know the content source.

**Resolve org and site from available sources (project-config, ops-config, git remote):**

```bash
# Try project-config first (handover context)
ORG=$(cat .claude-plugin/project-config.json 2>/dev/null | node -e "
  const d = require('fs').readFileSync(0,'utf8');
  try { process.stdout.write(JSON.parse(d).org || ''); } catch(e) {}
")

# Fallback to ops-config (ops context)
if [ -z "$ORG" ]; then
  ORG=$(node -e "
    const fs = require('fs');
    try {
      const c = JSON.parse(fs.readFileSync(process.env.HOME + '/.aem/ops-config.json', 'utf8'));
      process.stdout.write(c.org || '');
    } catch(e) {}
  ")
fi

# Site: try git remote first, then ops-config
SITE=$(basename -s .git $(git remote get-url origin 2>/dev/null) 2>/dev/null)
if [ -z "$SITE" ]; then
  SITE=$(node -e "
    const fs = require('fs');
    try {
      const c = JSON.parse(fs.readFileSync(process.env.HOME + '/.aem/ops-config.json', 'utf8'));
      process.stdout.write(c.site || '');
    } catch(e) {}
  ")
fi

echo "org=${ORG:-NOT SET} site=${SITE:-NOT SET}"
```

**If `ORG` is empty**, ask the user:

> "I need your organization name to authenticate. You can provide either:
> - The org name (the `{org}` in `https://main--site--{org}.aem.page`)
> - A preview/live URL like `https://main--mysite--myorg.aem.page/`"

**If user provides a URL**, parse org and site from it:

```bash
URL="$USER_INPUT"
if echo "$URL" | grep -q '\.aem\.page\|\.aem\.live'; then
  HOST_PART=$(echo "$URL" | cut -d'/' -f3 | cut -d'.' -f1)
  ORG=$(echo "$HOST_PART" | awk -F'--' '{print $NF}')
  SITE=$(echo "$HOST_PART" | awk -F'--' '{print $(NF-1)}')
  echo "Parsed from URL: org=$ORG site=$SITE"
fi
```

**If `SITE` is still empty** (not in a git repo and no URL provided), ask the user:

> "I also need a site name to auto-detect your login provider. What is your site name? (the `{site}` part of `https://main--{site}--{org}.aem.page`)"

**Do NOT proceed until both org and site are available.**

### Step 3: Capture Token via Playwright

The `/login/{org}/{site}/main` endpoint auto-redirects to the correct identity provider based on the project's configuration. Playwright follows the redirect, user logs in, and the `auth_token` cookie is captured automatically.

```bash
mkdir -p "${HOME}/.aem"

node -e "
const fs = require('fs');
const path = require('path');
const { chromium } = require('playwright');

const TOKEN_PATH = path.join(process.env.HOME, '.aem', 'ims-token.json');
const ORG = '${ORG}';
const SITE = '${SITE}';
const loginUrl = 'https://admin.hlx.page/login/' + ORG + '/' + SITE + '/main';

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
    console.error('Login timed out - no auth_token cookie found after 5 minutes');
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
  "authTokenExpiry": 1780489855
}
```

| Field | Description |
|-------|-------------|
| `authToken` | Token from `auth_token` cookie after login |
| `authTokenExpiry` | Unix timestamp when token expires |

Shared across every project on this machine. File is written with `0600` permissions.

---

## Using the Token

```bash
# Read token from user-level cache
AUTH_TOKEN=$(node -e "
  const fs = require('fs');
  try {
    const t = JSON.parse(fs.readFileSync(process.env.HOME + '/.aem/ims-token.json', 'utf8'));
    process.stdout.write(t.authToken || '');
  } catch (e) {}
")

# admin.hlx.page and Config Service use the same header
curl -H "x-auth-token: ${AUTH_TOKEN}" "https://admin.hlx.page/status/{org}/{site}/main/"
curl -H "x-auth-token: ${AUTH_TOKEN}" "https://admin.hlx.page/config/{org}/sites.json"
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `npx playwright` not found | Run `npm install -g playwright` |
| Browser doesn't open | Run `npx playwright install chromium` |
| Login page doesn't redirect | Verify org and site names are correct |
| Token not captured | Ensure login completed before closing browser |
| 401 after login | Token expired, re-authenticate |
| 403 on API | User lacks permission for that org/site |

---

## Integration

Called by: `ops`, `handover-admin`, `handover-author`, `handover-developer`, `handover`

```
Skill({ skill: "aem-project-management:auth" })
```
