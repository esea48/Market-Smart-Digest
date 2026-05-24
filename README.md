# Market Smart Digest

An autonomous n8n workflow that delivers a weekly PM-focused industry digest to your inbox and Slack — every Monday at 7 AM, no manual work required.

---

## What it does

1. Reads your subscriber list and industry config from Google Sheets
2. Searches for the 10 most relevant recent articles per industry using Tavily (biased to trusted sources)
3. Sends the raw articles to Claude, which produces a structured PM-focused HTML digest
4. Delivers the full digest to Gmail and a top story teaser to Slack
5. Repeats for every subscriber's industries in one run

Each digest includes:
- Executive summary
- Regulation to watch
- Company headlines + what we can learn
- Trend alerts
- Market gap callout
- Emerging customer pain
- PM question of the week

---

## Cost

| Component | Cost per industry per run |
|---|---|
| Tavily (10 results, advanced) | ~$0.02 |
| Claude Haiku | ~$0.003 |
| n8n Cloud | $0 (included in plan) |
| **Total** | **~$0.023** |

A team of 3 subscribers across 4 industries = ~$0.28/week or $14.50/year.

---

## Prerequisites

Before importing the workflow you need:

- **n8n Cloud** account (any paid plan)
- **Tavily API key** — [app.tavily.com](https://app.tavily.com)
- **Anthropic API key** — [console.anthropic.com](https://console.anthropic.com)
- **Google account** connected to n8n via OAuth (covers both Gmail and Google Sheets)
- **Slack bot** connected to n8n

---

## Setup

### Step 1 — Create two Google Sheets

**Sheet 1: `Market Digest Subscribers`**

| Email | Slack channel | Industries | Active |
|---|---|---|---|
| you@example.com | C0B4SSLQAEN | FinTech, Compliance Tech | Yes |

- `Industries` — comma-separated list
- `Slack channel` — Slack channel ID (e.g. `C0B4SSLQAEN`), leave blank to skip Slack
- `Active` — `Yes` or `No`. Set to `No` to unsubscribe without deleting the row.

**Sheet 2: `Market Digest Config`**

| Industry | Trusted Sources | Regulations | Regulation Title | Companies |
|---|---|---|---|---|
| FinTech | techcrunch.com, bloomberg.com | PSD3, Basel IV | Financial regulation | Stripe, Plaid |

Download the pre-filled config sheet with 14 industries already configured:
👉 [Market_Digest_Config.xlsx](./Market_Digest_Config.xlsx)

Upload it to Google Drive and open with Google Sheets.

---

### Step 2 — Import the workflow

1. In n8n, go to **Workflows → Import**
2. Upload `news-digest_with_database_-native_nodes.json`
3. The workflow opens with all 28 nodes on the canvas

---

### Step 3 — Connect credentials

Open each node marked with a warning icon and connect your credentials:

| Node | Credential needed |
|---|---|
| `Tavily search` | HTTP Header Auth — Name: `Authorization`, Value: `Bearer YOUR_TAVILY_KEY` (or add `api_key` in request body — see note below) |
| `Claude digest` | Anthropic API key |
| `Append or update row`, `Get row(s) in sheet`, `Get watchlist`, `Deactivate subscriber` | Google Sheets OAuth2 |
| `Send a message` (Gmail digest), `Send a message2` (error), `Unsubscribe confirmation` | Gmail OAuth2 |
| `Send a message1` (Slack digest), `Send a message3` (Slack error) | Slack API |

> **Tavily note:** Tavily accepts the API key in the request body as `api_key` rather than as a header. If you get a 401 error with header auth, switch to body auth.

---

### Step 4 — Update hardcoded values

Search the workflow for these and replace with your own:

| Node | Field | Replace with |
|---|---|---|
| `Send a message` (Gmail) | To | Your email address |
| `Send a message2` (error Gmail) | To | Your email address |
| `Claude digest` | Model | `claude-haiku-4-5-20251001` (or your preferred model) |
| `Get row(s) in sheet` | Document | Your `Market Digest Subscribers` sheet |
| `Get watchlist` | Document | Your `Market Digest Config` sheet |

---

### Step 5 — Activate

Flip the **Inactive** toggle at the top of the canvas to **Active**.

The workflow will now run automatically every Monday at 7 AM. To test immediately, use the Webhook trigger URL or submit the subscribe form.

---

## How it works — node by node

```
SUBSCRIBE PATH
On form submission → Prepare subscriber → Append or update row in sheet
                                                    ↓
SCHEDULE PATH                               Split industries
Schedule Trigger → Get row(s) in sheet → Split industries
                                                    ↓
                                          Get watchlist → Merge watchlist
                                                    ↓
                                           Tavily search
                                                    ↓
                                           Bundle results
                                                    ↓
                                            Build prompt
                                                    ↓
                                           Claude digest
                                                    ↓
                                           Extract HTML
                                          ↙            ↘
                               Send Gmail          IF (channel?)
                                                        ↓ yes
                                                   Send Slack

UNSUBSCRIBE PATH
Unsubscribe form → Deactivate subscriber → Unsubscribe confirmation

ERROR PATH
Error Trigger → Send a message2 (Gmail) + Send a message3 (Slack)
```

### Key nodes explained

**`Split industries`** — Takes each subscriber row and expands their comma-separated industry list into one item per industry using `flatMap`. 2 subscribers × 3 industries = 6 items all processed independently.

**`Get watchlist`** — Loads the entire `Market Digest Config` sheet (no filter). Matching is done in JavaScript in the next node.

**`Merge watchlist`** — Matches each industry to its config row using `Array.find()`. If no match (custom industry), falls back to generic trusted sources: `techcrunch.com, reuters.com, bloomberg.com, wired.com`.

**`Build prompt`** — Constructs the full Claude prompt dynamically using JavaScript template literals. The prompt includes topic, watchlist context, companies, regulations, trusted sources, and the 10 search results. This is a Code node — not the Anthropic node — because the prompt is too dynamic to build in a UI text field.

**`Extract HTML`** — Strips code fences Claude occasionally adds (` ```html ``` `), extracts the first article link via regex for the Slack teaser, and generates the `slackSummary` field.

**`IF (channel?)`** — Checks `$json.channel` is not empty before routing to Slack. Subscribers without a Slack channel ID silently skip Slack delivery.

---

## Subscribing and unsubscribing

**Subscribe:** Submit the n8n Form Trigger. You'll receive your first digest immediately, and every Monday from then on.

**Unsubscribe:** Submit the Unsubscribe Form. Your `Active` column in Google Sheets is set to `No`. You receive a confirmation email. The scheduler skips inactive rows.

**Manual management:** You can also directly edit the `Market Digest Subscribers` sheet — add rows, change industries, or set Active to No.

---

## Adding a new industry to the config

1. Open `Market Digest Config` in Google Sheets
2. Add a new row with:
   - `Industry` — exact name (must match what users type in the form)
   - `Trusted Sources` — comma-separated domains (no https://)
   - `Regulations` — comma-separated regulation names, or leave blank
   - `Regulation Title` — label used in the digest (e.g. `Financial regulation`)
   - `Companies` — comma-separated company names to watch
3. No changes needed in n8n — the workflow reads the sheet dynamically

---

## Known limitations

| Limitation | Workaround |
|---|---|
| No per-node retry logic | Enable **Retry On Fail** in Settings tab of `Tavily search` and `Claude digest` nodes (3 retries, 5s wait) |
| OAuth tokens expire | Refresh Google and Gmail credentials in n8n monthly, or use a service account |
| No digest archive | Add a Google Sheets append step after `Extract HTML` to save each digest |
| Scales to ~20-25 industries | Beyond that, execution time increases. For 50+ industries, batch processing is needed |
| Custom industries without config | Falls back to generic sources — no regulations or company watchlist applied |

---

## Troubleshooting

**Workflow stops at `Merge watchlist`**
→ `Get watchlist` returned 0 rows. Check your `Market Digest Config` sheet is connected and has rows with `Yes` in Active.

**Gmail sends but Slack doesn't**
→ Check the subscriber's Slack channel field contains a valid channel ID (format: `C0B4SSLQAEN`). The IF node skips Slack if the field is empty.

**Digest missing the PM question of the week**
→ Max tokens hit. Increase `max_tokens` in `Claude digest` to 6,000.

**401 error on Google Sheets**
→ OAuth token expired. Go to n8n Credentials, find your Google Sheets credential, and reconnect.

**422 error on Tavily**
→ API key issue. Confirm your key is in the request body as `api_key`, not as a header.

---

## File structure

```
news-digest_with_database_-native_nodes.json   ← n8n workflow (import this)
Market_Digest_Config.xlsx                       ← Pre-filled industry config (upload to Google Sheets)
README.md                                       ← This file
```

---

## Built with

- [n8n](https://n8n.io) — workflow automation
- [Tavily](https://tavily.com) — web search API
- [Anthropic Claude](https://anthropic.com) — AI synthesis (Haiku model)
- Google Sheets — subscriber and config database
- Gmail + Slack — delivery

---

*May 2026 · Market Smart Digest · Internal Product Team*
