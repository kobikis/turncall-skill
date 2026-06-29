# TurnCall Builder Skill

A [Claude Code](https://claude.com/claude-code) skill for **building apps on top of
[TurnCall](https://github.com/kobikis/turncall)** — the API-only voice-agent platform.

It guides the integration (it doesn't scaffold a stack): which endpoints to call, how to
handle webhooks and call-init, and where *your* logic plugs in. Covers three patterns:

- **Outbound** — leads → call them by your logic (campaign / dialer)
- **Inbound** — incoming call → route by your logic (router)
- **Text** — SMS and the Chat API

## Install

### As a plugin (recommended)

This repo is a Claude Code plugin marketplace. In Claude Code:

```
/plugin marketplace add kobikis/turncall-skill
/plugin install turncall@turncall-skill
```

Update later with `/plugin update`. (The repo is private — you'll need access to it.)

### Or copy it in manually

```bash
# from your app repo
mkdir -p .claude/skills
cp -r /path/to/turncall-skill/skills/turncall .claude/skills/turncall
```

Either way it's **model-invoked**: Claude reaches for it on its own when you're working with
TurnCall (placing outbound calls, routing inbound calls, handling TurnCall webhooks,
configuring call-init). You can also invoke it by name.

## What's inside

```
skills/turncall/
├── SKILL.md            # orient + the logic seam + route to a recipe
├── API.md              # curated builder API: auth, endpoints, webhook/call-init shapes
├── openapi.json        # snapshot of the full spec (field-exact fallback)
└── recipes/
    ├── outbound-campaign.md
    ├── inbound-routing.md
    └── sms-chat.md
.claude-plugin/         # plugin + marketplace manifests
```

## Keeping it current

`API.md` and `openapi.json` are curated/snapshotted from the TurnCall repo's
`docs/openapi.json`. When the TurnCall API changes, refresh both from that authoritative
spec.

## License

MIT.
