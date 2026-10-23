# PlacidWay Social Media Content Automation (n8n)

A demo n8n workflow that reads public PlacidWay content every day, asks an LLM (Groq) to write a Facebook post, checks the post against guardrails, and publishes it to a **test** Facebook Page (or sends it for human approval first).

> **Status: working prototype.** Sections marked ✅ were built and run in n8n. Sections marked 🔧 are designed but **not yet built or tested**. Nothing here claims Facebook publishing works until a real test post is confirmed on the Page. See [Status checklist](#13-status-checklist).

---

## 1. Ground rules followed

- Only public PlacidWay pages are used. `robots.txt` was fetched and the official `sitemap.xml` is used as the crawl entry point.
- Request rate must stay at or below **1 request per second** (see [Rate limiting](#rate-limiting)).
- Posting goes to a **test Facebook Page owned by the author**, never a PlacidWay page. Draft/simulation mode (`Test Facebook Post` node) exists for runs without a token.
- Secrets live only in n8n **Credentials**, never in the workflow JSON, this README, or chat.

---

## 2. Architecture

```
Schedule Trigger (daily 09:00) ─┐
                                ├─► Fetch PlacidWay Sitemap (sitemap.xml index)
Manual Trigger (testing) ───────┘            │
                                              ▼
                                  Extract Sitemap URLs  (keep only sitemap_articles.xml)
                                              ▼
                                  Fetch Content Sitemap (articles sitemap)
                                              ▼
                                  Extract Page URLs  (url + image + lastmod)
                                              ▼
                       [Select one NEW url: dedupe vs. Google Sheet log]      🔧
                                              ▼
                                  Fetch Page Content (HTML)
                                              ▼
                                  Extract Page Details (title, summary, image, canonical url)
                                              ▼
                                  Prepare Content for AI
                                              ▼
                                  Generate Facebook Post  ◄── Groq Chat Model
                                              ▼
                       [Validate / guardrails (Code node)]                     🔧
                                              ▼
                       [Config: PUBLISH_MODE  auto | approval]                 🔧
                          │ auto                       │ approval
                          ▼                            ▼
                [Facebook Graph API]            [Send draft to reviewer → wait → approve]   🔧
                          ▼
                Append row in sheet  (Google Sheets log)
                          ▼
                [Error Trigger workflow → alert]                               🔧
```

### Why sitemap parsing

| Option | Verdict |
|---|---|
| **Sitemap (chosen)** | Official, crawler-friendly, listed in `robots.txt`. One request returns hundreds of real article URLs with `lastmod` dates and thumbnail image URLs. No HTML scraping of listing pages. |
| RSS | No feed was identified for articles. |
| HTML listing pages | Fragile, heavier, more requests. |
| Headless browser | Overkill; pages are server-rendered. |

The sitemap index (`https://www.placidway.com/sitemap.xml`) lists 60+ sub-sitemaps (pages, articles, news, packages, doctors, videos, translations, search pages…). The workflow currently targets **`https://www.placidway.com/1/sitemap_articles.xml`**. Each `<url>` there has `<loc>`, `<lastmod>` and one or more `<image:loc>`; the **first `image:loc` is the article thumbnail**, which is used as the featured image.

> Earlier prototype versions only read `sitemap_pages.xml`, which is why they kept selecting the generic `/learning-center` landing page. That is the main content-quality bug the articles sitemap fixes.

---

## 3. Requirements and how each is met

| # | Requirement | Approach | Status |
|---|---|---|---|
| 1 | Daily trigger + manual run | `Schedule Trigger` (every 1 day, 09:00) and a manual trigger feeding the same pipeline | ✅ built; ⚠️ active state and timezone must be verified |
| 2 | Extract title, URL, summary, image | Sitemap → page HTML → `Extract Page Details` (CSS selectors) | ✅ built; articles-sitemap filter 🔧 pending confirmation run |
| 3 | One item per run, never repeat a URL | Compare against Google Sheet log, keep the newest unposted article | 🔧 not built |
| 4 | LLM post (hook, 2–4 lines, CTA, link, 3–5 hashtags, <500 chars before link) | Groq `openai/gpt-oss-20b` via `Generate Facebook Post` | ✅ generates text; prompt tightening 🔧 |
| 5 | Guardrails | Rule-based Code node (section 7) | 🔧 logic tested offline; not yet in n8n |
| 6 | Publish to test Page, include image | Facebook Graph API | 🔧 not built (no Meta token yet) |
| 7 | Auto vs approval mode | `PUBLISH_MODE` flag in a Config node | 🔧 not built |
| 8 | Logging + errors + alerts | Google Sheets log; retries; Error Trigger alert | ✅ log appends; retries/alerts 🔧 |

---

## 4. Setup

### Prerequisites
- n8n (built and tested on **n8n Cloud**; free trial). Version not recorded; use a current release with the AI Agent / Basic LLM Chain nodes.
- A Groq account and API key.
- A Google account (for Google Sheets).
- For publishing: a Meta developer account, a Facebook **test Page you admin**, and a Page access token.

### 4.1 Groq credential
1. Create an API key at the Groq console.
2. In n8n: **Credentials → New → Groq API**, paste the key there.
3. In `Groq Chat Model`, select that credential and model `openai/gpt-oss-20b`.
4. **Never paste the key into chat, the README, or a Set/Code node.** If a key is ever exposed, revoke it immediately and create a new one.

### 4.2 Google Sheets
1. Create a spreadsheet named **`PlacidWay Facebook Post Logs`**.
2. Row 1 headers (exact order):

| Date | Title | URL | Post Text | Image URL | Status | Facebook Post ID |
|---|---|---|---|---|---|---|

3. In n8n: **Credentials → New → Google Sheets OAuth2**, authorize, then select the sheet in `Append row in sheet`.

*Planned extra columns:* `Mode`, `Validation`, `Error`, `Approval` (section 9).

### 4.3 Meta / Facebook 🔧
Not yet configured. Outline (verify against current Meta docs, as they change):
1. Create an app at developers.facebook.com.
2. Create or use a **test Facebook Page** you admin. Do not use a PlacidWay Page.
3. Obtain a **Page access token** with permissions to publish (typically `pages_manage_posts` and `pages_read_engagement`).
4. Store it in an n8n credential (Facebook Graph API / header auth). Never paste it into chat.
5. Publishing calls:
   - **With image:** `POST /{page-id}/photos` with `url` (image URL) and `caption` (post text).
   - **Text + link:** `POST /{page-id}/feed` with `message` (post text).
6. The response `id` is logged as **Facebook Post ID**.

Short-lived tokens expire. A long-lived Page token is needed for a daily job, and token expiry must raise an alert (section 10).

---

## 5. Configuration

| Setting | Where | Values |
|---|---|---|
| `PUBLISH_MODE` | A `Config` **Set node** at the start of the flow 🔧 | `auto` or `approval` |
| Schedule | `Schedule Trigger` | every 1 day, 09:00 |
| Groq model | `Groq Chat Model` | `openai/gpt-oss-20b` |
| Source sitemap | `Extract Sitemap URLs` filter | `sitemap_articles.xml` |

**Why a Set node and not environment variables:** on n8n Cloud, server environment variables are not something you edit freely, so the flag is kept inside the workflow where it is visible and easy to flip during a demo.

### Rate limiting
To respect "≤ 1 request/second", set on each HTTP Request node: **Options → Batching → Items per Batch `1`, Batch Interval `1000` ms**. In the intended flow only about 3–4 PlacidWay requests happen per run (sitemap index, articles sitemap, one article page), so the limit is easily met. 🔧 Batching not yet set.

---

## 6. How to run

**Manual:** open the workflow → **Execute workflow** (starts from the manual trigger).
**Scheduled:** the workflow must be **saved and published/active**. The `Schedule Trigger` uses the workflow timezone. Check **Settings → Timezone** so 09:00 means what you expect. Active state has not been verified yet, and the n8n execution counter showed no scheduled runs.

---

## 7. Post generation and guardrails

### Prompt requirements (for `Generate Facebook Post`)
- Use **only** the supplied title, summary and URL.
- Hook → 2–4 short value lines → CTA → exact URL (once) → 3–5 hashtags.
- Under 500 characters **before** the URL.
- No medical claims, no promised outcomes, no invented prices or facts.
- Output the final post text only.

### Rule-based validator (Code node) 🔧
The prompt alone is not trusted. After generation, a Code node checks:

| Check | Rejects when |
|---|---|
| Link | URL count ≠ 1, or URL ≠ the source URL |
| Length | text before the link > 500 characters |
| Hashtags | fewer than 3 or more than 5 |
| Banned claims | `guarantee(d)`, `cure(s/d)`, `miracle`, `risk-free`, `100%`, `permanent(ly)`, `no side effects`, `painless`, `success rate(s)`, `best clinic/doctor/surgeon` |
| Prices | any `$/€/£` amount not present in the source text |
| CTA | no call-to-action verb (read, learn, see, get, request, explore, discover, compare, check, find) |

A failing post is **not published**; it is logged with status `FAILED_VALIDATION` and the error codes.

**Offline test (run for real):** the same logic was run in Node against 3 real-page sample posts and 7 deliberately bad posts.

```
PASS Dental implants Istanbul | chars before link: 283 | hashtags: 4
PASS IVF Bangkok | chars before link: 259 | hashtags: 4
PASS Hip replacement Puerto Vallarta | chars before link: 313 | hashtags: 4
PASS negative: guarantee claim -> BANNED_PHRASE_guaranteed
PASS negative: invented price -> PRICE_NOT_IN_SOURCE_$99
PASS negative: wrong URL -> URL_MISMATCH
PASS negative: no hashtags -> HASHTAGS_0
PASS negative: too long -> TOO_LONG_525
PASS negative: URL twice -> URL_COUNT_2
PASS negative: cure claim -> BANNED_PHRASE_cures
ALL CHECKS PASSED
```

Script: `validator_test.js`. This tests the logic only. The n8n Code node version is not yet wired in or run inside n8n.

### Limitations of the guardrails
- Keyword and number matching **cannot verify that every statement is supported by the page**. A fluent but wrong claim without a banned word or a price would pass. A second LLM "fact check" pass against the source text is the planned addition.
- The banned-phrase list is small and English-only.
- Price check only looks at currency-symbol amounts, not "four hundred dollars".
- Source pages are medical content. Even faithful summaries should go through **approval mode** for any real brand use.

---

## 8. Duplicate prevention 🔧

Not built yet. The earlier attempt (n8n "Remove Duplicates" + Limit) produced zero items and was removed. Planned design:

1. Read all values in the `URL` column of `PlacidWay Facebook Post Logs`.
2. From `Extract Page URLs`, drop every URL already in that list.
3. Sort the remainder by `lastmod` (newest first) and keep **one**.
4. If nothing is left, stop the run cleanly and log `NO_NEW_CONTENT` (instead of passing zero items to later nodes).

**When is a URL "used"?** Decision: as soon as a row is written with status `PENDING_APPROVAL`, `PUBLISHED`, `SCHEDULED` or `REJECTED`. A URL whose run failed for a **technical** reason (site down, API error) is logged as `FAILED_TECHNICAL` and **may be retried**. This stops a rejected or pending draft from being re-proposed tomorrow while still letting real failures retry.

Dedupe uses the Google Sheet as the persistent store, so it survives restarts and works on n8n Cloud.

---

## 9. Review / approval mode 🔧

`PUBLISH_MODE = auto` → validated post goes straight to Facebook.
`PUBLISH_MODE = approval` → draft (text, image URL, source URL) is sent to a reviewer and the workflow waits for approve/reject before publishing.

The review channel (email, Slack, etc.) is **not yet chosen**. It depends on what the reviewer can access, and it needs its own n8n credential.

---

## 10. Logging, retries and alerts

### Log schema (Google Sheet)
Current ✅: `Date, Title, URL, Post Text, Image URL, Status, Facebook Post ID`
Planned 🔧: `Mode, Validation, Approval, Error`

### Status values
`TEST_READY` (simulation, current) · `FAILED_VALIDATION` · `PENDING_APPROVAL` · `REJECTED` · `PUBLISHED` · `FAILED_TECHNICAL` · `NO_NEW_CONTENT`

### Retries and alerts 🔧
- HTTP Request nodes: enable **Retry On Fail** (e.g. 3 tries, wait several seconds) for transient errors.
- `Fetch Page Content` currently uses **Never Error = ON**, so a 404 does not stop the run. That means the workflow must **check the status code explicitly** and branch, otherwise a 404 error page could be summarized into a post.
- A separate **Error Trigger** workflow will send an alert for persistent failures (site down, Groq/Google credential errors, expired Facebook token, failed publish).

---

## 11. Image handling

- The featured image comes from the first `<image:loc>` of the article in the sitemap, with `og:image` from the page as a cross-check.
- The prototype's earlier image was `.../provider-center/template/img/logo-white.png` (a **logo**, from a landing page). The workflow must never publish that. Rule: if the image URL contains `logo`, is empty, or does not load, publish **text-only**.
- Page `og:image` values are Cloudflare-resized URLs (`/cdn-cgi/image/…`); the raw `placidwaycdn.s3.amazonaws.com/images/article_thumb/...` URL from the sitemap is preferred.

---

## 12. Sample outputs (real PlacidWay pages)

Source pages were fetched on 2026-10-10 and the titles, descriptions and prices below were read from them.

> **Honesty note:** these three posts were written to the workflow's prompt spec and checked with the validator above. They are **not yet outputs of the n8n run** from the articles sitemap. Replace them with the actual `Generate Facebook Post` output from the first successful run before submission.

### Sample 1 — Dental implants in Istanbul
- **Source:** https://www.placidway.com/article/2327/Get-Affordable-Dental-Implants-in-Istanbul-Turkey
- **Page title:** Dental Implants in Istanbul Turkey Patient Guide
- **Supporting fact (from page):** single implant typically $400 to $1,200 USD depending on brand and materials.
- **Image:** `https://placidwaycdn.s3.amazonaws.com/images/article_thumb/1791554390Dental-implants-in-Istanbul%2C-Turkey-%282%29.jpg`

```
Thinking about dental implants abroad? 🦷
One PlacidWay guide puts single implants in Istanbul at roughly $400–$1,200 USD, depending on brand and materials.
It also covers the licences to verify and what an itemized quote should include.
Read the full guide and request a free quote:
https://www.placidway.com/article/2327/Get-Affordable-Dental-Implants-in-Istanbul-Turkey
#DentalImplants #MedicalTourism #Istanbul #DentalTourism
```
283 characters before link · 4 hashtags · validator: PASS

### Sample 2 — IVF in Bangkok
- **Source:** https://www.placidway.com/article/4836/IVF-Support-Services-in-Bangkok-Counseling-Nutrition-and-Stress-Management
- **Page title:** IVF in Bangkok Thailand | Cost Clinics and Patient Guide
- **Supporting facts (from page):** standard cycle typically $6,000 to $9,000 USD; success rates depend heavily on maternal age and underlying health; check Ministry of Public Health Thailand licensing.
- **Image:** `https://placidwaycdn.s3.amazonaws.com/images/article_thumb/1791542168IVF-treatment-in-Bangkok%2C-Thailand-%282%29.jpg`

```
Considering fertility treatment abroad? 🌏
PlacidWay's guide to IVF in Bangkok lists a typical cycle at about $6,000–$9,000 USD.
It also explains which clinic licences to check and notes that outcomes depend on factors like age and health.
Get the full guide:
https://www.placidway.com/article/4836/IVF-Support-Services-in-Bangkok-Counseling-Nutrition-and-Stress-Management
#IVF #FertilityTreatment #MedicalTourism #Bangkok
```
259 characters before link · 4 hashtags · validator: PASS

### Sample 3 — Hip replacement in Puerto Vallarta
- **Source:** https://www.placidway.com/article/3165/Key-Information-on-Hip-Replacement-in-Puerto-Vallarta-Mexico
- **Page title:** Planning Your Hip Replacement in Puerto Vallarta Mexico to Understand Costs and Patient Safety
- **Supporting facts (from page):** estimated $8,500 to $13,500 USD depending on implant type and hospital; verify COFEPRIS licensing and surgeon certification; plan for extended local recovery before flying.
- **Image:** `https://placidwaycdn.s3.amazonaws.com/images/article_thumb/1791549660Hip-replacement-surgery-in-Puerto-Vallarta%2C-Mexico-%282%29.jpg`

```
Planning a hip replacement and weighing your options? 🦴
This PlacidWay guide covers Puerto Vallarta, Mexico, with a cost range of $8,500–$13,500 USD depending on implant and hospital.
It also lists credentials to verify and stresses planning your recovery and flight home.
See what to consider before you decide:
https://www.placidway.com/article/3165/Key-Information-on-Hip-Replacement-in-Puerto-Vallarta-Mexico
#HipReplacement #OrthopedicSurgery #MedicalTourism #PuertoVallarta
```
313 characters before link · 4 hashtags · validator: PASS

---

## 13. Status checklist

**Done and run in n8n ✅**
- [x] Manual trigger and Schedule Trigger node configured (daily 09:00)
- [x] robots.txt and sitemap.xml fetched
- [x] Sitemap URL extraction, page fetch, HTML field extraction (title, summary, image, canonical URL)
- [x] Groq LLM generates a post (`text` field)
- [x] `Test Facebook Post` simulation node (`published: false`)
- [x] Google Sheets append works (row confirmed in the sheet)

**Done outside n8n ✅**
- [x] Real article URLs discovered from the articles sitemap and three pages read
- [x] Validator logic tested offline (3 pass, 7 correctly rejected)

**Pending 🔧**
- [ ] Confirm workflow is saved/active and timezone is correct
- [ ] Filter `Extract Sitemap URLs` to `sitemap_articles.xml` and confirm the run
- [ ] Replace the `learning-center` IF filter with real article selection
- [ ] Duplicate prevention against the Google Sheet
- [ ] Tightened LLM prompt; validator added in n8n; fail path logs `FAILED_VALIDATION`
- [ ] 404 / status-code handling after `Fetch Page Content`
- [ ] Carry source `url`, `title`, `image` through to the log (stop relying on a regex on the post text; stop hardcoding the title)
- [ ] 1 request/second batching on HTTP nodes
- [ ] `PUBLISH_MODE` flag and approval flow (reviewer channel undecided)
- [ ] Meta app, Page token, **real publish to a test Page**, log Facebook Post ID
- [ ] Retries and Error Trigger alert workflow
- [ ] Exported workflow JSON, narrated recording, 3 samples replaced with real workflow output

---

## 14. Known limitations

- **No Facebook post has been published yet.** `published: false` is simulation only.
- Approval mode, duplicate prevention and guardrails are not in the workflow yet (design and offline test only).
- Guardrails are rule-based and cannot fully verify facts (section 7).
- n8n Cloud trial is time-limited; export the workflow JSON before it ends.
- Many PlacidWay articles are marketing/medical-tourism content with price ranges. Posts should be human-reviewed before real use.
- Articles sitemap `lastmod` dates reflect page updates, so "newest" can include old articles that were recently edited.

---

## 15. Security

- Keys and tokens only in n8n Credentials. Exported workflow JSON stores credential **names/IDs, not secrets**; still check the file before sharing.
- A Groq key was once pasted in chat. It must stay revoked and replaced.
- Post only to a Page you own. Revoke the Meta token after the demo.

---

## 16. Export the workflow

1. Open the workflow in n8n.
2. Top-right **`…` menu → Download** (saves a `.json` file).
3. Open the file and confirm no secrets are present before submitting.
4. Submit the `.json` with this README.

(Label it clearly as a prototype if the pending items above are not finished.)

---

## 17. Screen-recording outline (5–10 min, narrated)

| Time | Show |
|---|---|
| 0:00–1:00 | Goal and architecture diagram; why sitemap parsing |
| 1:00–2:00 | Canvas walk-through; Schedule Trigger (daily 09:00) and Manual Trigger; `PUBLISH_MODE` setting |
| 2:00–4:00 | **Live manual run:** sitemap index → articles sitemap → chosen article; show title, summary, URL, image |
| 4:00–5:00 | Duplicate check against the Google Sheet |
| 5:00–6:30 | Groq-generated post; validator result (pass, and one deliberate fail) |
| 6:30–8:00 | Auto or approval routing; **the post appearing on the test Facebook Page**; Facebook Post ID |
| 8:00–9:00 | Google Sheets log row |
| 9:00–10:00 | Retries, alerts, known limitations |

The Facebook segment can only be recorded once the publishing step is built and a real test post exists. Until then it remains pending.


## Links

- **Google Sheet (run log):** [PlacidWay Facebook Post Logs](https://docs.google.com/spreadsheets/d/1cDEh0dBrUbrCGbRwN1e348uwtEvzfMEjOrL7-zV0g3s/edit?usp=sharing)
- **Workflow export:** `placidway-workflow.json` (in this repo)
- **Screen recording:** [Watch the demo](https://www.loom.com/share/fc003b2dfe194317926a1cd7313cbb57)
