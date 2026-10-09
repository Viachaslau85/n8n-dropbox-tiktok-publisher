# Design notes

Decisions behind the workflow, the alternatives that were rejected, and what is still open.

## 1. Architecture

### Two flows joined by a table

| Flow | Trigger | Responsibility |
|---|---|---|
| Pack registration | Form submission | Create the folder, record the pack |
| Publishing | Schedule, every 15 min | Find new videos in recorded packs, publish, record results |

The flows share one Data Table (`packs`) and nothing else.

**Why not one flow.** A single flow started by the form would have to wait for uploads that happen minutes or hours later, and one submission would own one long-running execution. With a registry, registration finishes in seconds and publishing is a stateless poll that can be stopped, restarted or re-run at any time.

**Why a form and not automatic discovery.** Folder names are opaque CRM codes and the day's folder contains packs from other workflows. Only a person knows which pack is theirs, so that one fact is collected explicitly and everything else is derived.

### State in n8n Data Tables, not in the spreadsheet

The first design used the Google Sheet as both the log and the duplicate check. It was replaced because the sheet is edited by people: deleting a row would have re-published a video. The ledger now lives in n8n; the sheet receives a copy of successful results.

| | Data Table | Google Sheet |
|---|---|---|
| Role | Source of truth for idempotency | Human-facing report |
| Written | Before and after every publish | After a successful publish only |
| Editable by the team | No | Yes, without side effects |

## 2. Exactly-once publishing

Posting is irreversible through the API, so the publish step is protected by four independent measures.

| Measure | Failure it covers |
|---|---|
| Anti-join on the ledger before the loop | Normal re-runs: files already handled are skipped |
| `pending` row written before the publish call | A second run starting while the first is still inside the loop; a crash after publishing but before recording |
| Upsert instead of insert for that row | Manual re-execution of a node during debugging creating duplicate rows |
| No automatic retry on the publish request, 180 s timeout | A slow or lost response being retried into a second post |

**Key choice.** The idempotency key is the lower-cased Dropbox path. File names repeat across packs; paths do not. Dropbox paths are case-insensitive, so the lower-cased form is stable.

**Consequence.** A video left in `pending`, `failed` or `publishing` is not retried. Re-queuing means deleting its row. Automatic retry was rejected: a rejection for duplicate content or a daily cap would loop forever, and a false retry is worse than a missed post.

## 3. Handling of external APIs

### Dropbox

| Need | Endpoint | Note |
|---|---|---|
| Create folder | native node | Parent folders are created implicitly |
| List folder | native node | "Always output data" keeps the flow alive on an empty folder |
| Shared link | `sharing/create_shared_link_with_settings` | Needs `sharing.write` |
| Direct link | `files/get_temporary_link` | Valid 4 hours, never stored |

The native credential type cannot request `sharing.write`, which is why the two HTTP Request nodes use a generic OAuth2 credential bound to the same Dropbox app. The app must have all five scopes enabled **before** the account is connected: a token carries only the scopes granted at consent time.

A repeat request for a shared link returns HTTP 409 with the existing link inside the error body. The node is set to never throw, and one expression reads either location:

```
{{ $('Shared link').item.json.url
   ?? $('Shared link').item.json.error?.shared_link_already_exists?.metadata?.url
   ?? '' }}
```

### Zernio / TikTok

Observed behaviour, which differs from a naive reading of the documentation:

| Step | Response |
|---|---|
| `POST /v1/posts` | Immediately `status: "publishing"`; no video ID yet |
| `GET /v1/posts/{id}` after about 3 minutes | `published` with `platformPostId` and `platformPostUrl`, or `failed` with `errorMessage` |
| A post later taken down | Still `published`, plus `removedFromPlatformAt` |

Rules that follow:

- Success is decided from the body, never from the HTTP status.
- `removedFromPlatformAt` maps to a separate `removed` status.
- The wait between publish and poll doubles as spacing between posts.

### Google Sheets

- Header row is set explicitly because the tracker's header is not on row 1.
- Only three columns are mapped. Unmapped columns are not written at all, which protects formulas.
- The video ID is prefixed with an apostrophe so Sheets stores it as text; a 19-digit number would lose precision.

## 4. Portability

| Item | Where it lives |
|---|---|
| API key, OAuth tokens | n8n credentials (encrypted, excluded from export) |
| TikTok account | Resolved at run time from `GET /v1/accounts` |
| Dropbox base folder | `Pack path` node |
| Caption variants | `Build request` node |
| Spreadsheet and sheet | `Add to sheet` node |

The account lookup fails fast when it finds no active TikTok account or more than one, with a message naming the setting to change. Posting to the wrong account silently was considered the worst possible outcome.

## 5. Deployment

The workflow runs on a single n8n container.

| Mode | Use | Limits |
|---|---|---|
| Local (`localhost`) | Development, single operator | Form and OAuth work only on the host machine |
| Local + HTTPS tunnel | Pilot with a remote team | Host must stay on; the editor is exposed and needs a strong password and 2FA |
| Server with a domain | Production | Running cost |

OAuth providers accept only `localhost` or HTTPS callback URLs, so any setup with remote users needs a public HTTPS address and these variables:

```yaml
WEBHOOK_URL: "https://your-address/"
N8N_EDITOR_BASE_URL: "https://your-address/"
N8N_PROXY_HOPS: "1"
```

The official n8n image is sufficient; the workflow uses built-in nodes only.

## 6. Testing approach

- Each node was added and verified on real data before the next one.
- The publish node was pinned to a recorded real response while downstream nodes were built, because executing any node inside a loop re-runs the loop body.
- A final unpinned run on three videos confirmed distinct video IDs, correct links and correct sheet rows.
- A scheduled run across two packs registered on different days confirmed queue order and that late uploads are picked up by the next poll.

## 7. Open items

| Item | Planned approach |
|---|---|
| Packs never expire | Scheduled step setting `closed` after N days, or a close action in the form |
| `publishing` after the first poll | Second flow re-checking unresolved rows |
| Daily posting cap | Count today's `published` rows before the loop and defer the rest |
| No alerts | Error workflow sending a message on failed executions and on `failed` rows |
| Settings spread over two nodes | One configuration node at the start of each flow |
| Second source | Google Drive branch producing the same item shape as `Only videos` |
