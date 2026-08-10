---
name: manage-menu
description: Manage a Bishkek café or coffee shop's QR menu (locations, menus, categories, items, prices, availability) through the ai-menu-manager MCP server. Use when the caller is the café owner or their staff and wants to view or edit their own business data — list locations/menus, add a dish, change a price, mark an item out of stock, reorder categories or items, create a new location, or check onboarding status. Requires an OAuth login to the owner MCP server (separate from any console-web session); always call get_account_status first in a new session before any other tool.
when_to_use: Trigger phrases like "покажи мои точки/меню", "добавь позицию", "поменяй цену", "отметь как нет в наличии", "создай новую локацию", "list my locations/menus", "add a menu item", "change the price of X", "mark X out of stock", "reorder categories".
---

# Managing a QR menu with the owner MCP server

This skill is the operating guide for the `ai-menu-manager` MCP server's 19
tools (6 read, 13 write). It reuses the server's own tool descriptions and
account-status guidance verbatim rather than re-describing them, so this
file matches what the server itself tells a client.

## 1. Always call `get_account_status` first

Every new session starts by calling `get_account_status` — it works even
before onboarding and returns whether the caller's account is linked to a
company yet, which `company_id`/role/scopes apply, and a `next_action` hint.
If the account is not yet linked, call `onboard_company` before anything
else. Every other tool call in this session needs the `company_id` this
call returns.

The resource hierarchy is `company -> location -> menu -> category -> item`.
List tools return the ids you need to go one level deeper (e.g. `list_menus`
needs the `location_id` from `list_locations`).

## 2. Which tool for which situation

| Situation | Tool | R/W | Scope |
|---|---|---|---|
| Onboarding check (call first, every session) | `get_account_status` | R | none |
| First-time setup, no company yet | `onboard_company` | W | none |
| "Show my locations" | `list_locations` | R | `locations:read` |
| "What menus does location X have" | `list_menus` | R | `menus:read` |
| "What categories are in this menu" | `list_categories` | R | `categories:read` |
| "List the items in this category" (name/price/stock only) | `list_items` | R | `menu_items:read` |
| "Show full details of this item" (description, photos) | `get_item` | R | `menu_items:read` |
| Update branding, name, status, languages | `update_company` | W | `companies:write` |
| Add a new venue/point | `create_location` | W | `locations:write` |
| Edit a location's description/hours/contacts | `update_location` | W | `locations:write` |
| Add an extra menu (e.g. "Lunch", "Drinks") | `create_menu` | W | `menus:write` |
| Edit a menu's name/description | `update_menu` | W | `menus:write` |
| Make a menu the location's default | `set_default_menu` | W | `menus:write` |
| Add a category (e.g. "Coffee", "Desserts") | `create_category` | W | `categories:write` |
| Edit a category's name/description | `update_category` | W | `categories:write` |
| Add a dish/drink | `create_item` | W | `menu_items:write` |
| Edit an item's fields | `update_item` | W | `menu_items:write` |
| Mark in-stock / out-of-stock | `set_item_availability` | W | `menu_items:write` |
| Change display order of menus/categories/items | `reorder` | W | scope matches the `entity` argument |

There is **no delete tool** for any of these resources — every write tool
in this server declares `destructiveHint:false`. Removing a dish from view
is `set_item_availability` with
`out_of_stock: true` (a soft stop-list flag, not a delete). If the user asks
to "delete" or "remove" something, use the out-of-stock/disable path and say
so — don't imply data was erased.

`create_location` atomically creates a default menu for the new location in
the same call — mention this so the user isn't surprised a "Основное меню"
already exists.

## 3. Prices: whole KGS as a decimal string, not tyiyn

`create_item`'s and `update_item`'s `price` field is documented on the wire
exactly like this:

> Price in WHOLE Kyrgyzstani som (KGS) as a decimal string, e.g. "120.50"
> means 120 som 50 tyiyn — this is som, NOT tyiyn/minor units. Backed by
> numeric(12,2): at most 2 digits after the decimal point, at most 10 digits
> before it, non-negative, no scientific notation.

**Failure mode to actively avoid:** if a user says "a latte for 180 som",
send `"price": "180"` or `"180.00"` — never `"18000"` (that would be
180 *tyiyn*, i.e. 1.80 som on a public menu people pay with). There is no
minor-unit convention anywhere in this API; every price you send or receive
is already in whole som.

## 4. Localization: `{ru, ky, en}`, `ru` is required

Names and descriptions for locations, categories, and items are localized
objects with `ru`/`ky`/`en` keys. `ru` is always
required and non-empty; `ky` and `en` are optional. On an `update_*` call
the localized field **replaces the whole object** — if you only want to fix
the Kyrgyz translation, still resend `ru` (and `en` if set) with their
current values, or you will blank them out.

`create_category` and `create_item` refuse a duplicate `name.ru` (case- and
whitespace-insensitive) in the same parent by default; pass
`allow_duplicate_name: true` only when the user explicitly wants two items
with the same Russian name.

## 5. Everything you write is live — confirm before calling a write tool

There is no draft/publish step beyond a menu's own optional `status: draft`.
A price change, a new item, or an availability flip through this server is
visible on the location's public menu page immediately, to real customers
(see §6 below for how to get that page's exact link — never construct one
yourself). Before calling any `W` tool from the table above:

1. State back to the user exactly what you are about to change (which
   location/menu/category/item, old value → new value, and the price in
   plain "N som" form) and get an explicit go-ahead, unless the user's
   instruction was already that specific and unambiguous.
2. Never guess an id. Resolve `company_id`/`location_id`/`menu_id`/
   `category_id`/`item_id` from `get_account_status` and the `list_*` tools
   in this same conversation — don't reuse an id from a much earlier
   session without re-listing, since it may have changed.
3. `set_default_menu` and `set_item_availability` are idempotent — safe to
   retry if a call times out. `create_*` tools are not; don't blindly retry
   a timed-out create without checking with `list_*` whether it already
   landed.

## 6. Reading results

- List tools (`list_locations`, `list_menus`, `list_categories`,
  `list_items`) return `{items: [...], offset, limit, has_more}`. `has_more`
  is a heuristic (`true` when the page came back full) — keep paging with
  an increasing `offset` until a response comes back short. Default page
  size is 25, max 100.
- `list_items` returns a **compact projection only**: item_id, name, price,
  out_of_stock, sort_order, photo_count — no description or photo URLs.
  Call `get_item` for the full record before describing a dish in detail to
  the user.
- `list_locations` and `list_menus` (and the `create_location`/
  `update_location`/`create_menu`/`update_menu`/`set_default_menu` results,
  which share the same shape) each carry a `public_url` field — the direct
  link to that location's or menu's public page. Return it to the user
  as-is. Never build, edit, or guess this URL yourself, even when the
  pattern looks obvious from one example: if a situation needs a link that
  hasn't come back from a tool yet, call `list_locations`/`list_menus`
  (or the relevant write tool) again rather than assembling one.
- Every id in a request/response is a UUID string. Get them from
  `get_account_status` (company_id) or the corresponding `list_*` call —
  never invent one.

## 7. Auth note

This server requires a separate OAuth login through your MCP client — a
console-web session cookie is not accepted here by design. If
a tool call fails with an auth error mid-conversation, tell the user to
re-authenticate via their client's MCP connection settings rather than
retrying the same call repeatedly.
