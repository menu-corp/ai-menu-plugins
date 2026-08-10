# AI Menu Plugins

Two agent plugins for [menu.qr.kg](https://menu.qr.kg), a QR-menu platform
for independent cafés and coffee shops in Bishkek, Kyrgyzstan.

| Plugin | For | Access |
|---|---|---|
| [AI Menu Catalog](ai-menu-catalog/) | Anyone looking for a café | Anonymous, read-only |
| [AI Menu Manager](ai-menu-manager/) | A café owner managing their own menu | OAuth sign-in, reads and writes |

## What each plugin does

**AI Menu Catalog** lets an AI assistant search published café and
coffee-shop menus and prices in Bishkek — "where's the cheapest cappuccino
near Chuy", "does X have vegan options", "show me Y's menu" — anonymously,
with no account and no sign-in required.

**AI Menu Manager** lets a café owner manage their own QR menu —
locations, menus, categories, items, prices, and the stop-list — by
talking to an assistant instead of filling in a form: "add a latte for
180 som", "mark the cheesecake out of stock", "what are my locations?".

## Install

### Claude Code

```
/plugin marketplace add menu-corp/ai-menu-plugins
/plugin install ai-menu-catalog@ai-menu-plugins
/plugin install ai-menu-manager@ai-menu-plugins
```

### Codex

```
codex plugin marketplace add menu-corp/ai-menu-plugins
```

This registers the marketplace; installing and enabling a plugin from it
happens in the ChatGPT desktop app's Plugins Directory.

Either package can also be loaded directly from a local checkout of this
repository — see that package's own README for the exact command.

## AI Menu Manager writes to your live menu

Most of what AI Menu Manager does is a write: a price change, a new item,
or a stop-list flip made through it goes out to the café's public menu page
immediately — there is no draft or review step beyond a menu's own
optional "draft" status. Installing this plugin and signing in grants it
permission to change what a business's customers see and pay, not just to
read its data. AI Menu Catalog never writes anything; it only reads what a
business has already published.

## AI Menu Manager isn't generally installable yet

Signing in to AI Menu Manager currently requires an OAuth client that was
registered by hand for testing — the automatic registration a new sign-in
attempt relies on does not yet work. In practice, that means anyone trying
to install this plugin today and sign in will hit a sign-in failure rather
than a working install. If you'd like access, open an issue in this
repository and we'll follow up. AI Menu Catalog has no sign-in step at all
and is unaffected.

## Prices

Every price either plugin shows is in Kyrgyzstani som (KGS) and reflects a
café's menu as published at the moment of the request. An owner can change
a price or an item's availability at any time, and neither plugin has a
review delay — what you see is live, but it is not a quote, a reservation,
or a contractual offer.

## More detail

- [`ai-menu-catalog/README.md`](ai-menu-catalog/README.md) and its
  [skill guide](ai-menu-catalog/skills/search-cafes/SKILL.md)
- [`ai-menu-manager/README.md`](ai-menu-manager/README.md) and its
  [skill guide](ai-menu-manager/skills/manage-menu/SKILL.md)

## License

MIT — see [LICENSE](LICENSE). Each package also carries its own copy, since
a package may be copied out and used on its own.
