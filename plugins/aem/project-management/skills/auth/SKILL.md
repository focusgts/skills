---
name: auth
description: Authenticate with AEM Edge Delivery Services. Opens browser for login and captures token. Works for admin.hlx.page, admin.da.live, and Config Service APIs regardless of content source (Document Authoring, SharePoint, or Google Drive).
license: Apache-2.0
allowed-tools: Read, Write, Edit, Bash, AskUserQuestion
metadata:
  version: "3.0.0"
---

# AEM Edge Delivery Services Authentication

Authenticate to obtain a token for all Edge Delivery Services admin operations. Supports all content sources: SharePoint (Microsoft), Google Drive (Google), Document Authoring (Adobe), and Crosswalk (Adobe).

## Token Usage

The `auth_token` cookie works for all admin APIs:

| API | Header | Usage |
|-----|--------|-------|
| `admin.hlx.page` | `x-auth-token: ${AUTH_TOKEN}` | Preview, publish, status, code sync, jobs, logs, config |
| `admin.da.live` | `x-auth-token: ${AUTH_TOKEN}` | DA content operations (list, source, copy, move) |
| Config Service | `x-auth-token: ${AUTH_TOKEN}` | Sites, config, secrets, API keys, profiles |

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

Tokens are stored at the **project level** in `.claude-plugin/project-config.json`.

```bash
mkdir -p .claude-plugin
grep -qxF '.claude-plugin/' .gitignore 2>/dev/null || echo '.claude-plugin/' >> .gitignore

AUTH_TOKEN=$(cat .claude-plugin/project-config.json 2>/dev/null | node -e "
  const d = require('fs').readFileSync(0,'utf8');
  try {
    const c = JSON.parse(d);
    const now = Math.floor(Date.now()/1000);
    if (c.authToken && c.authTokenExpiry > now + 60) process.stdout.write(c.authToken);
  } catch(e) {}
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

### Step 2.5: Resolve Organization and Auth Provider

The login requires the org name and identity provider. Check saved config:

```bash
ORG=$(cat .claude-plugin/project-config.json 2>/dev/null | node -e "
  const d = require('fs').readFileSync(0,'utf8');
  try { process.stdout.write(JSON.parse(d).org || ''); } catch(e) {}
")

AUTH_PROVIDER=$(cat .claude-plugin/project-config.json 2>/dev/null | node -e "
  const d = require('fs').readFileSync(0,'utf8');
  try { process.stdout.write(JSON.parse(d).authProvider || ''); } catch(e) {}
")

echo "org=${ORG:-NOT SET}"
echo "authProvider=${AUTH_PROVIDER:-NOT SET}"
```

**If `ORG` is empty**, ask the user:

> "I need your organization name to authenticate. You can provide either:
> - The org name (the `{org}` in `https://main--site--{org}.aem.page`)
> - A preview/live URL like `https://main--site--org.aem.page/`"

**If user provides a URL**, parse org from it:

```bash
URL="$USER_INPUT"
if echo "$URL" | grep -q '\.aem\.page\|\.aem\.live'; then
  HOST_PART=$(echo "$URL" | cut -d'/' -f3 | cut -d'.' -f1)
  ORG=$(echo "$HOST_PART" | awk -F'--' '{print $NF}')
  echo "Parsed from URL: org=$ORG"
fi
```

**If `AUTH_PROVIDER` is empty**, ask the user for their content source:

> "What is your project's content source?
> 1. SharePoint
> 2. Google Drive
> 3. Document Authoring (DA)
> 4. Crosswalk
>
> Please enter 1/2/3/4."

Map the response:
- 1 (SharePoint) → `AUTH_PROVIDER=microsoft`
- 2 (Google Drive) → `AUTH_PROVIDER=google`
- 3 (DA) → `AUTH_PROVIDER=adobe`
- 4 (Crosswalk) → `AUTH_PROVIDER=adobe`

Save both values to config:

```bash
node -e "
  const fs = require('fs');
  let c = {};
  try { c = JSON.parse(fs.readFileSync('.claude-plugin/project-config.json', 'utf8')); } catch(e) {}
  c.org = '${ORG}';
  c.authProvider = '${AUTH_PROVIDER}';
  fs.writeFileSync('.claude-plugin/project-config.json', JSON.stringify(c, null, 2));
"
```

**Do NOT proceed until both org and auth provider are available.**

### Step 3: Capture Token via Playwright

The `/login/{org}` endpoint does NOT auto-redirect — it returns a JSON with multiple provider links. Instead, navigate directly to `https://admin.hlx.page/auth/{provider}` which redirects to the correct identity provider login page.

| Content Source | Auth Provider | Login URL |
|---|---|---|
| SharePoint | microsoft | `https://admin.hlx.page/auth/microsoft` |
| Google Drive | google | `https://admin.hlx.page/auth/google` |
| DA | adobe | `https://admin.hlx.page/auth/adobe` |
| Crosswalk | adobe | `https://admin.hlx.page/auth/adobe` |

After login completes, the token is stored as the `auth_token` cookie on `admin.hlx.page`. Playwright reads this cookie, saves it to the project config, then closes the browser automatically.

```bash
node -e "
const fs = require('fs');
const { chromium } = require('playwright');

const CONFIG_PATH = '.claude-plugin/project-config.json';
const AUTH_PROVIDER = '${AUTH_PROVIDER}';
const loginUrl = 'https://admin.hlx.page/auth/' + AUTH_PROVIDER;

(async () => {
  console.log('Opening browser for login...');
  console.log('Identity provider: ' + AUTH_PROVIDER);
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
    let config = {};
    try { config = JSON.parse(fs.readFileSync(CONFIG_PATH, 'utf8')); } catch (e) {}
    config.authToken = token;
    config.authTokenExpiry = expiresAt;
    fs.writeFileSync(CONFIG_PATH, JSON.stringify(config, null, 2));
    console.log('Authentication successful');
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

Stored in `.claude-plugin/project-config.json` (project-level, gitignored):

```json
{
  "org": "myorg",
  "contentSource": "sharepoint",
  "authProvider": "microsoft",
  "authToken": "eyJ...",
  "authTokenExpiry": 1780489855
}
```

| Field | Description |
|-------|-------------|
| `org` | Config Service organization name |
| `contentSource` | Content source: `sharepoint`, `google`, `da`, `crosswalk` |
| `authProvider` | Identity provider: `microsoft`, `google`, `adobe` |
| `authToken` | Token from `auth_token` cookie after login |
| `authTokenExpiry` | Unix timestamp when token expires |

---

## Using the Token

```bash
# Read token from project config
AUTH_TOKEN=$(cat .claude-plugin/project-config.json | node -e "
  const d = require('fs').readFileSync(0,'utf8');
  process.stdout.write(JSON.parse(d).authToken || '');
")

# All APIs use the same header
curl -H "x-auth-token: ${AUTH_TOKEN}" "https://admin.hlx.page/status/{org}/{site}/main/"
curl -H "x-auth-token: ${AUTH_TOKEN}" "https://admin.hlx.page/config/{org}/sites.json"
curl -H "x-auth-token: ${AUTH_TOKEN}" "https://admin.da.live/list/{org}/{site}"
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `npx playwright` not found | Run `npm install -g playwright` |
| Browser doesn't open | Run `npx playwright install chromium` |
| Login page doesn't redirect | Check `authProvider` is correct for your content source |
| Token not captured | Ensure login completed before closing browser |
| 401 after login | Token expired, re-authenticate |
| 403 on API | User lacks permission for that org/site |

---

## Integration

Called by: `ops`, `handover-admin`, `handover-author`, `handover-developer`, `handover`

```
Skill({ skill: "aem-project-management:auth" })
```
