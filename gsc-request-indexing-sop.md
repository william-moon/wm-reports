# Google Search Console — Request-Indexing SOP

**Purpose:** How to check a page's index status and request indexing reliably and honestly, using William's shared setup. Written for any William chat that handles request-indexing.

**Last verified:** 2026-07-16 (Command Center chat)

---

## 0. The one rule that matters most

**Never fake or guess indexing status. Real GSC data only.** If a request fails, log it as a
failure — never as "requested" or "indexed." Honesty over optimistic reporting. This is
non-negotiable.

---

## 1. The big picture (what actually works vs. what doesn't)

There are three ways to touch Google's index. Only some are reliable:

| Path | What it does | Reliable? | Use it? |
|------|-------------|-----------|---------|
| **URL Inspection API** | READ a page's live index status | ✅ Yes, permanent | **Yes — this is our source of truth for status** |
| **Sitemaps** | NOTIFY Google of all URLs | ✅ Yes, durable | **Yes — best hands-off signal** |
| **Browser "Request Indexing" button** | Manually push one URL | ⚠️ Google blocks our bot | Fallback only; William's manual clicks work, our headless does not |
| **Indexing API** (indexing.googleapis.com) | POST a URL for crawl | ❌ **Not allowed for us** | **No — see §5** |

**Key truth:** Nobody — no tool, not even William clicking — can *force* Google to index a page.
You can only reliably *notify* Google and let it decide. What we can make fully hands-off and
trustworthy is the notifying + honest status reporting.

---

## 2. Reading index status (works, permanent)

Use the central shared CLI. cwd doesn't matter; William's Mac does NOT need to be on:

```
node /home/william/projects/-1003918414144/scripts/gsc.mjs list                 # all mapped GSC properties + account
node /home/william/projects/-1003918414144/scripts/gsc.mjs status <url|domain>   # live index verdict (PASS = indexed)
```

- Backed by the **URL Inspection API** — permanent, no anti-bot issues.
- Covers all 20 GSC properties across the william / charlene / clients Google accounts;
  auto-routes each domain to its owning account.
- `PASS` = indexed. Anything else = not yet indexed.

---

## 3. Requesting indexing (the honest browser path)

```
node /home/william/projects/-1003918414144/scripts/gsc.mjs request-index <url> --confirm
```

(Dry-run without `--confirm`.)

**What to expect — proven live this session (3 runs, 2 sites):**
- Google returns **"Oops! Something went wrong… try again later"** for our headless submit.
- Reading works; only the *submit* is rejected for automation.
- **William's own manual clicks in a real browser succeed** (his 11 FAP URLs indexed fine).
- So Google accepts a human in a real browser but blocks our headless bot.

**The trust bug we fixed (do not reintroduce):** the old driver scanned the page for the words
"Indexing requested" — text Google leaves on the page from *any past request* — so it logged
REQUESTED even when the submit had just errored. It recorded failures as successes.
The fix: only ever report success on Google's real confirmation modal; gate on the live-test
spinner + modal dialog only; stop after 3 Google errors (`GOOGLE_ERROR_RETRY`) instead of hammering.

Driver: `scripts/gsc-request-indexing.mjs`. Rotation cron: `scripts/gsc-indexing-rotate.mjs`
(PER_SITE cap = 10/account/run; aborts after 3 Google errors). A live browser request ≈ 70s each.

**Google's daily cap** is ~10–12 URLs per property. Don't exceed it — bot requests + William's
manual requests share the same cap.

---

## 4. The durable hands-off path: sitemaps

The most reliable legitimate signal Google honors without any per-URL action is a **complete,
fresh sitemap** submitted in GSC. Keep every tracked URL in the sitemap and keep it current.
This is what to lean on for a genuinely hands-off baseline.

---

## 5. Why we do NOT use the Indexing API (important)

Tempting, but **off the table.** Google's Indexing API officially supports **only two page types**:
- Pages with **JobPosting** structured data
- Pages with **BroadcastEvent** (livestream video) markup

Service pages, blog posts, and general pages are **not eligible**. Google's docs still state the API
"can only be used to crawl pages with either JobPosting or BroadcastEvent." A May 2025 policy
clarification tightened enforcement — using it on ineligible pages violates their terms and risks
**getting the service account's API access revoked**. That account is Owner on all of William's
GCP projects, so a ban would be costly. **Do not use the Indexing API for service pages.**
(Verified against Google's official docs 2026-07-16.)

Sources:
- https://developers.google.com/search/apis/indexing-api/v3/quickstart
- https://developers.google.com/search/apis/indexing-api/v3/using-api

---

## 6. Dashboard behavior

- Requested URLs must stay **VISIBLE** on the dashboard with their request date + live status,
  **even after they index** (don't drop them the moment they turn Indexed).
- Generator: `scripts/gsc-dashboard.mjs` (**gitignored**, box-only, holds tokens — apply on the
  box, never push).
- Live dashboard: https://william-moon.github.io/wm-reports/gsc-dashboard.html

---

## 7. Where everything lives

**Scripts** (in `/home/william/projects/-1003918414144/scripts/`):
- `gsc.mjs` — on-demand CLI: `list` / `status <url>` / `request-index <url> --confirm`
- `gsc-request-indexing.mjs` — browser driver (honest; gates on modal; never fakes success)
- `gsc-indexing-rotate.mjs` — rotation cron (3-error abort; 10/account/run cap)
- `gsc-dashboard.mjs` — dashboard generator (gitignored, box-only)
- `index-status-check.mjs` — status sweep

**Data** (in `data/`): `tracked-pages.json`, `tracked-pages-indexed.json`,
`indexing-rotation-log.json`, `indexing-requests.json`, `index-status-latest.json`.
Screenshots: `data/gsc-req-*.png`.

**Sessions:** `secure/_shared/gsc-ui-state-{william,charlene,clients}.json` — cookies good to
2027-07, refreshed nightly by the session-harvest cron. **FAP = the clients account**
(clientspleasellc@gmail.com).

**Config:** `config/gsc-properties.json` (e.g. FAP → `sc-domain:financial-advisorpro.com`,
account "clients").

**Service account** (for reads/API path): `command-center-automation@wm-command-center.iam.gserviceaccount.com`
(Owner on all projects). GCP wrapper: `scripts/gcloud.sh`.

**Crons:** session-harvest `*/30`; index-status-check `12:10`; gsc-indexing-rotate `12:40`;
gsc-dashboard `13:00`.

---

## 8. Gotchas

- Gitignored (box-only, never push): `gsc-dashboard.mjs`, `gsc-ui-state-*.json`, `gsc-oauth.json`.
- Clients Please (CP) is walled — never give it the central tokens. It manages its own.
- Leading-dash project dirs → prefix paths with `./`.
- A live browser request ≈ 70s.
- Google's manual button is Google-side anti-bot flaky; it can't be made 100% reliable via a
  headless browser. That's a Google limitation, not a bug in our code.
