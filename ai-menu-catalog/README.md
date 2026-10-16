# AI Menu Catalog

Ask Claude about cafés and coffee shops in Bishkek — "where's the cheapest
cappuccino near Chuy", "does X have vegan options", "show me Y's menu",
"what are Z's opening hours" — and get an answer sourced from the actual,
currently-published menus on [menu.qr.kg](https://menu.qr.kg). This is the
plugin for **anyone looking for a café**, not for a business managing its
own menu; see [AI Menu Manager](../ai-menu-manager/README.md) for that.

## What it does

Talking to Claude with this plugin enabled, you can:

- search across venues, menus and dishes in Bishkek by name, description or
  price;
- browse the full list of published venues with no search term at all;
- fetch a specific venue's or menu's full details — categories, items,
  prices, opening hours, and its other locations if it's a multi-branch
  brand — rendered as readable Markdown.

Full tool-by-tool guidance lives in
[`skills/search-cafes/SKILL.md`](skills/search-cafes/SKILL.md).

## Read-only, anonymous, no setup

This plugin never writes anything and never asks who you are. There is no
sign-in, no account, and no company/location context — every tool call
answers from the public catalog exactly as an anonymous website visitor
would see it. Nothing you ask through this plugin is tied to a menu.qr.kg
account (there isn't one to tie it to).

Only **published** menus show up. A location, menu, or business that's
inactive, disabled, in draft, or deleted never appears in a search or
fetch result, and neither does an empty menu or a location with no menu
to show — if nothing comes back, the venue may simply not have a menu
published right now.

## Install

See the top-level [README](../README.md#install) for the marketplace-add
command, or load it directly by path from a local checkout, no account or
API key needed:

```bash
claude --plugin-dir ./ai-menu-catalog
```

## Prices are informational, not an offer

Every price is in Kyrgyzstani som (KGS) and reflects the menu **as published
at the moment of the request**. The owner can change a price, an item's
availability, or the whole menu at any time, and this catalog has no
draft/publish delay — what you see is live, but it is not a quote, a
reservation, or a contractual offer. Confirm anything price-sensitive with
the venue directly before relying on it.
