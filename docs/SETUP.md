# Setup

From an empty machine to the first published video. Allow about an hour.

Never share API keys, app secrets or client secrets, and keep them out of screenshots.

## Prerequisites

| Item | Purpose |
|---|---|
| Docker | Runs n8n |
| Zernio account with one TikTok account connected | Publishing |
| Dropbox account | Video source |
| Google account with access to the tracker spreadsheet | Result log |

## 1. Start n8n

```bash
docker compose up -d
```

Open `http://localhost:5678` and create the owner account.

Set `TZ` and `GENERIC_TIMEZONE` in `docker-compose.yml` to your time zone first: the folder date is taken from the form submission time.

For remote users, put n8n behind HTTPS and add the three variables listed in [DESIGN.md](DESIGN.md#5-deployment).

## 2. Create the Data Tables

*Overview → Data tables → Create Data table.* All columns are of type String; names are case-sensitive.

| Table | Columns |
|---|---|
| `packs` | `packCode`, `folderDate`, `packPath`, `status` |
| `posted` | `name`, `postID`, `dropboxLink`, `dropboxPath`, `packCode`, `status`, `zernioPostID`, `error` |

## 3. Import the workflow

*Create Workflow → ⋯ → Import from File →* `workflow.json` *→ Save.*

## 4. Create the credentials

The OAuth redirect URL shown by n8n is `http://localhost:5678/rest/oauth2-credential/callback`, or the same path on your public address.

### 4.1 Zernio — Header Auth, named `Zernio API`

| Field | Value |
|---|---|
| Name | `Authorization` |
| Value | `Bearer YOUR_API_KEY` |

### 4.2 Dropbox app

1. `https://www.dropbox.com/developers/apps` → *Create app* → **Scoped access** → **Full Dropbox**.
2. *Permissions*: enable the five scopes below, then **Submit**.
   - `files.metadata.read`
   - `files.content.read`
   - `files.content.write`
   - `sharing.read`
   - `sharing.write`
3. *Settings*: add the redirect URL; copy **App key** and **App secret**.

Enable the scopes before connecting. If they change later, disconnect and reconnect both Dropbox credentials.

### 4.3 Dropbox — Dropbox OAuth2 API, named `Dropbox account`

| Field | Value |
|---|---|
| Client ID | App key |
| Client Secret | App secret |
| APP Access Type | Full Dropbox |

### 4.4 Dropbox — OAuth2 API (generic), named `Dropbox sharing`

Required because the built-in credential cannot request `sharing.write`.

| Field | Value |
|---|---|
| Grant Type | Authorization Code |
| Authorization URL | `https://www.dropbox.com/oauth2/authorize` |
| Access Token URL | `https://api.dropboxapi.com/oauth2/token` |
| Client ID / Client Secret | App key / App secret |
| Scope | `files.metadata.read files.content.read files.content.write sharing.read sharing.write` |
| Auth URI Query Parameters | `token_access_type=offline` |
| Authentication | Body |

### 4.5 Google Sheets — Google Sheets OAuth2 API, named `Google Sheets account`

1. Google Cloud Console → new project → enable **Google Sheets API** and **Google Drive API**.
2. OAuth consent screen: External; add yourself as a test user.
3. Create an OAuth client ID of type *Web application* with the redirect URL.
4. Paste Client ID and Client Secret into n8n and sign in.

While the Google app is in *Testing*, tokens expire after 7 days. Publish the app for permanent use.

## 5. Bind the nodes

Imports carry credential names but not their internal IDs, so each node must be pointed at your objects once.

| Nodes | Select |
|---|---|
| `List accounts`, `Publish`, `Check post` | `Zernio API` |
| `Create pack folder`, `Check pack folder`, `List pack files` | `Dropbox account` |
| `Shared link`, `Temp link` | `Dropbox sharing` |
| `Add to sheet` | `Google Sheets account`, your spreadsheet and sheet |
| `Register pack`, `Active packs` | Data table `packs` |
| `Not posted yet`, `Reserve row`, `Save result` | Data table `posted` |

After choosing a Data Table, confirm that the node's condition and column values are still filled in.

In `Add to sheet`, set *Options → Data Location on Sheet → Header Row* to the row holding your headers, and keep exactly three mapped columns: `Name`, `Post ID`, `Dropbox Link`.

## 6. Adjust to your environment

| Setting | Node | Field |
|---|---|---|
| Dropbox base folder | `Pack path` | `packPath` |
| Captions | `Build request` | `CAPTIONS` |
| TikTok account, if several are connected | `Build request` | `TIKTOK_USERNAME` |
| Polling interval | `Every 15 min` | interval |

## 7. First run

1. Open `List accounts` and execute it: your TikTok account should be listed with `isActive: true`.
2. Activate the workflow.
3. Copy the *Production URL* from `Pack form` and open it.
4. Submit a test code, upload one video to the folder shown, and wait up to 20 minutes.

Expected: the video is on TikTok, the `posted` table has a row with `status = published` and a 19-digit `postID`, and the sheet has a new row.

## Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| `redirect_uri mismatch` | Redirect URL differs from the one in the app | Compare character by character |
| `missing_scope` | Scope not enabled, or token issued before it was | Enable, submit, disconnect and reconnect |
| OAuth returns to `localhost` and fails | Public address not configured | Set `WEBHOOK_URL` and `N8N_EDITOR_BASE_URL`, recreate the container |
| Video not picked up | Folder created by hand, not through the form | Register the code in the form |
| Row stays `pending` for more than 10 minutes | Execution stopped mid-loop | Check *Executions*; delete the row to re-queue |
| `Duplicate content detected` | TikTok recognised an earlier upload | Not retryable |
| Sheet writes stop after a week | Google app still in *Testing* | Publish the app and reconnect |
| Video ID ends in zeros in the sheet | Written as a number | Keep the leading apostrophe in the `Post ID` mapping |
