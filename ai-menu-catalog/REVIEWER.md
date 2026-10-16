# Reviewer notes — AI Menu Catalog

Public listing name: **AI Menu Catalog** (package id: `ai-menu-catalog`).

Everything below was verified live against the production endpoint on
2026-08-10, not inferred from source alone. Re-run any `curl` command here to
reproduce.

## Endpoint and auth

- **MCP endpoint:** `https://catalog-menu.qr.kg/mcp` (Streamable HTTP,
  `POST`, `Content-Type: application/json`, `Accept: application/json,
  text/event-stream`).
- **No authentication of any kind.** No API key, no OAuth, no cookie, no
  session header on `initialize`. Every tool call below succeeded with a
  bare, unauthenticated `curl`. This is intentional — the server is the
  public, anonymous menu catalog described in the plugin's own
  `README.md`; there is no user identity for it to check.
- `initialize` returns `serverInfo.name = "ai-menu-catalog"`,
  `protocolVersion = "2025-11-25"`, and a server `instructions` string that
  restates the price/currency and tool-selection guidance below — safe to
  skip re-reading it here, it does not add anything this document doesn't
  already cover.

```bash
curl -s https://catalog-menu.qr.kg/mcp -X POST \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"reviewer","version":"0"}}}'
```

## The four tools

All four are `readOnlyHint: true`, `openWorldHint: false`,
`idempotentHint: true` per their `tools/list` annotations — none of them
mutate anything, none reach outside this server's own catalog.

| Tool | Required args | What it does |
|---|---|---|
| `search` | `query` (≥2 chars) | Free-text search across venues, menus and dishes. Returns up to 25 `{id, title, url, text}` rows, `text` a short snippet. |
| `list_venues` | none | Browse every published venue with no search term. Returns `{venues: [{id, location_slug, name, address, menus: [...]}]}`. |
| `search_menu_items` | `query` (≥2 chars) | Item-level search with optional `location_slug`, `available_only`, `price_min_kgs`/`price_max_kgs`, `lang`, paging. Returns `{items: [{name, price, currency, ...}]}`. |
| `fetch` | `id` | Given an id from any of the above (`location:<slug>` or `menu:<slug>/<menu-slug>`), returns the full venue or menu as rendered Markdown plus `metadata`. |

## Test cases

### Positive

1. **`list_venues` with no arguments returns the seeded demo catalog.**
   ```bash
   curl -s https://catalog-menu.qr.kg/mcp -X POST \
     -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
     -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"list_venues","arguments":{}}}'
   ```
   Expected: a `venues` array including `location:demo` ("Nomad Kitchen —
   Центр"), `location:demo-ala-too` ("Nomad Kitchen — Ала-Тоо" — same
   `company_name`, a second branch, proving the multi-branch/sibling-location
   feature), and `location:lyudmila`.

2. **`fetch` on the known-good id `location:demo` returns "Nomad Kitchen".**
   ```bash
   curl -s https://catalog-menu.qr.kg/mcp -X POST \
     -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
     -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"fetch","arguments":{"id":"location:demo"}}}'
   ```
   Expected: `result.structuredContent.title = "Nomad Kitchen — Основное
   меню"`, `structuredContent.url = "https://menu.qr.kg/@demo"`,
   `structuredContent.metadata.currency = "KGS"`, and
   `structuredContent.text` is a Markdown menu opening with "# Nomad
   Kitchen", the address/phone/hours block, a "## Основное меню" heading,
   and priced items (e.g. "Сырники с облепихой — 420 сом"). `result.content[0].text`
   carries the **same** data JSON-encoded as a single string (this is the
   dual-shape the skill doc describes for ChatGPT's connector integration —
   don't expect Markdown directly in `content[0].text`, it's
   `json.dumps(structuredContent)`).
   `result._meta["kg.qr.menu/card"]` also carries a structured venue card for
   MCP-Apps-capable hosts (`venue_name: "Nomad Kitchen"`) — present but
   optional to check.

3. **`search` for `"Nomad"` returns both Nomad Kitchen branches.**
   ```bash
   curl -s https://catalog-menu.qr.kg/mcp -X POST \
     -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
     -d '{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"search","arguments":{"query":"Nomad"}}}'
   ```
   Expected: two results, `location:demo` and `location:demo-ala-too`, each
   with a Russian-language snippet and a `fetch`-able `url`.

4. **`search_menu_items` for `"капучино"` finds the same dish at two
   different prices across venues/menus** — a real "cheapest X" query:
   ```bash
   curl -s https://catalog-menu.qr.kg/mcp -X POST \
     -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
     -d '{"jsonrpc":"2.0","id":5,"method":"tools/call","params":{"name":"search_menu_items","arguments":{"query":"капучино"}}}'
   ```
   Expected: 3 rows named "Капучино" — `180.00` KGS on
   `demo-ala-too`'s "Бизнес-ланч" menu, `220.00` KGS on both venues'
   "Основное меню". A client asking "cheapest cappuccino in Bishkek" should
   surface the `180.00` row.

5. **A too-short query is rejected with a specific, non-generic message**
   (this is validation feedback, not an enumeration signal):
   ```bash
   curl -s https://catalog-menu.qr.kg/mcp -X POST \
     -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
     -d '{"jsonrpc":"2.0","id":6,"method":"tools/call","params":{"name":"search_menu_items","arguments":{"query":"a"}}}'
   ```
   Expected: `isError: true`, `content[0].text = "query must be at least 2
   characters"`.

### Negative

6. **`fetch` on an id that does not exist at all.**
   ```bash
   curl -s https://catalog-menu.qr.kg/mcp -X POST \
     -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
     -d '{"jsonrpc":"2.0","id":7,"method":"tools/call","params":{"name":"fetch","arguments":{"id":"location:this-does-not-exist-xyz"}}}'
   ```
   Expected: `isError: true`, `content[0].text = "not found"`. Verified live.

## A deliberate anti-enumeration property, not a bug

**An unknown id and a real-but-unpublished (draft/disabled/deleted) venue or
menu return the exact same `"not found"` string.** There is no way to tell,
from the wire response alone, whether `location:some-slug` doesn't exist or
exists but isn't currently published. This collapse is intentional and
covered by regression tests: the server documents this string as the
single generic message returned for every `fetch` failure mode — malformed
id, unknown slug, draft/disabled menu, and inactive/soft-deleted business
all collapse to it. One test calls `fetch` with three different ids — an
unknown location slug, an unknown menu slug, and a real menu slug that is
currently in draft status — and asserts all three responses are
byte-identical, contain no trace of the underlying reason (no "draft", no
internal id), and carry no `structuredContent`. A second test pins the same
collapse one layer up, over an in-process MCP client. If a probe against
two different ids that you expect to behave differently (one truly
nonexistent, one you know is unpublished) comes back with the same
`"not found"` both times, that is the anti-enumeration design working as
intended — please don't file it as a bug.

We do not have a second demo id in a draft/disabled state to hand you for a
live side-by-side repro against production; the property above is what the
server's own documentation and its regression tests guarantee regardless.

## What this document does not cover

AI Menu Manager (package id: `ai-menu-manager`), the write-capable sibling
plugin, is **not** ready for review. Its OAuth flow itself works — a real
café owner has completed the full sign-in round-trip end-to-end and
performed both a read and a write against production data — but it has
two unresolved blockers that are each independently disqualifying for a
public listing regardless: the authorization server is not yet a
production identity environment, and automatic client registration fails
for a new client attempting to sign in (root cause unresolved). Reviewer
instructions for it would be wrong by the time they're used. This document
is scoped to AI Menu Catalog (`ai-menu-catalog`) only.
