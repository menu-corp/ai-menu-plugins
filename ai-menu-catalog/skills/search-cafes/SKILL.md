---
name: search-cafes
description: Search the public, anonymous catalog of published QR menus for independent cafés and coffee shops in Bishkek, Kyrgyzstan, through the ai-menu-catalog MCP server (search, fetch, search_menu_items, list_venues). Use whenever someone asks about a café, a dish, or a price in Bishkek — no login is required and no company/account context is needed.
when_to_use: Trigger phrases like "where's the cheapest cappuccino in Bishkek", "does X have vegan options", "show me the menu for X", "what cafes are there", "what are X's opening hours", "какое кафе дешевле" — any question an anonymous end user (not the café owner) would ask about published menus.
---

# Searching the public QR-menu catalog

This skill is the operating guide for the `ai-menu-catalog` MCP server's 4
read-only, anonymous tools. It reuses the server's own tool-selection
guidance verbatim rather than re-describing it.

No account, no login, no company context — every tool call is anonymous and
read-only.

## Which tool for which situation

| Situation | Tool |
|---|---|
| Broad question, no specific venue yet ("coffee shop near Chuy", "cafe with a lunch menu") | `search` |
| Specific dish or price point across venues ("cheapest cappuccino in Bishkek", "vegan options") | `search_menu_items` |
| No search term at all — just browsing what's in the catalog | `list_venues` |
| You already have an id (from `search`/`list_venues`, or a link the user pasted) and want the full menu | `fetch` |

`search` returns up to 25 ranked results, each with a short snippet and a
`fetch`-able id — use `fetch` on a result's id to get the complete,
human-readable menu. `fetch` renders opening hours, categories, items, and
prices as Markdown, plus the venue's other locations if it's a multi-branch
brand.

## Reading results

`search` and `fetch` return their payload in **two parallel forms** on
every call: a typed `structuredContent` object and a JSON text content
block with the same data — this is the shape ChatGPT's connector
integration requires, and both fields carry identical information. Read
whichever is more convenient; don't treat them as two different answers.

- `search` result rows: `{id, title, url, text}` — `text` is a ≤200-rune
  snippet, not the full menu. Follow up with `fetch` before answering a
  question that needs full menu detail.
- `fetch` result: `{id, title, text, url, metadata}` — `text` is the
  rendered Markdown menu; `metadata` includes `currency`, `lang`,
  `is_default`, and `updated_at`.
- `search_menu_items` and `list_venues` have their own result shapes for
  item-level and venue-level browsing respectively — use `fetch` afterward
  if the user wants the surrounding menu, not just the matched row.

## Prices are informational only, never a quote

Every price returned is in Kyrgyzstani som (KGS) and reflects the menu **as
published at the moment of the request**. The owner can change price,
availability, or menu contents at any time, and this server has no
draft/publish step — what you see is live. When answering with a price,
say it reflects the current published menu; do not present it as a
guaranteed or contractual offer, and do not promise it will still be that
price later.

## Search is exact/prefix/substring only — no typo tolerance

There is no fuzzy matching, no transliteration, and no synonym expansion.
`latte` will not find a misspelled «латте» or vice versa, and a mixed-script
query like "латте latte" will not match either alphabet. If a search comes
back empty, try the term in the other script or a simpler substring before
concluding the item doesn't exist — don't tell the user "not found anywhere"
after only one script/spelling attempt.

## What's not in this catalog

Only **published** menus appear: a location, menu, or company that is
inactive, disabled, draft, or deleted never shows up in any of these four
tools. If nothing comes back, say the venue may not have a menu published
right now rather than implying it doesn't exist at all.
