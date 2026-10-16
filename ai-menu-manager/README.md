# AI Menu Manager

Manage your café or coffee shop's QR menu from inside Claude — locations,
menus, categories, items, prices, and the stop-list — by talking to it in
plain language ("add a latte for 180 som", "mark the cheesecake out of
stock", "what are my locations?"). This is the plugin for the **business
owner**, not for customers browsing a menu; see
[AI Menu Catalog](../ai-menu-catalog/README.md) for that.

## What it does

Talking to Claude with this plugin enabled, you can:

- see your locations, menus, categories and items, and check onboarding
  status;
- add a new location or menu, edit a location's hours/contacts/description;
- show or hide a menu for guests and choose which one opens first (menus
  are tabs on your location's public page);
- add, edit, and reorder categories and items;
- change a price, mark an item in/out of stock (soft stop-list — the dish
  stays in your data, it just stops showing to customers);
- update your business's name, branding, and status.

Full tool-by-tool guidance lives in
[`skills/manage-menu/SKILL.md`](skills/manage-menu/SKILL.md).

## This plugin writes to your live, public menu

Most of what this plugin does is a **write**: a price change, a new item, or
a stop-list flip goes out to your public menu page immediately, to real
customers — there is no draft/review step beyond hiding a menu through its
own status (a new menu goes live as soon as it has an item, unless you hide
it first). Granting this plugin
access to your account is granting it permission to change what your
customers see and pay, not just to read your data. Claude will confirm the
specific change (which item, old value → new value) before calling a write
tool, but the account-level access itself is all-or-nothing: there's no way
to install this plugin as read-only.

Every price you send or read is **whole Kyrgyzstani som (KGS)**, e.g.
`"180.00"` for 180 som — never smaller units.

## Install

1. Requires a [menu.qr.kg](https://menu.qr.kg) account with at least one
   business/location already set up (or use `onboard_company` through the
   plugin itself on first use).
2. Install from this repository — see the top-level
   [README](../README.md#install) for the marketplace-add command, or load
   it directly by path from a local checkout:
   ```bash
   claude --plugin-dir ./ai-menu-manager
   ```
   Enable it, then the first tool call in a new session
   (`get_account_status`) triggers an **OAuth sign-in** to your menu.qr.kg
   account — this is separate from any browser session you may already have
   open on the console web app.
3. Once signed in, ask Claude about your menu.

## Data and privacy

- Everything this plugin reads or writes belongs to **your own** business —
  it operates in your account's scope only, resolved from your OAuth login,
  never a company/location you don't own.
- There is no delete tool for any resource; "removing" something means
  marking it out of stock (items) or hiding it (menus: `update_menu` with
  status `draft` or `disabled`) — nothing is erased.
- Prices, availability and menu contents reflect what's published *right
  now*; this plugin has no memory of "what it used to say" beyond what you
  ask it to change.
