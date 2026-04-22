# matrix-expert

A Claude Code Skill packed with hard-won knowledge about the Matrix ecosystem:
matrix-rust-sdk, MAS OAuth, E2EE, sliding sync, and the architectural patterns
that make or break real-world Matrix apps.

When loaded, it gives Claude instant context on things like:

- MAS OAuth vs legacy `/login` — token formats, refresh flows
- The single-use refresh-token trap (the #1 source of bugs in multi-client apps)
- Sliding sync (MSC4186) vs `/sync` on large accounts (3000+ rooms)
- E2EE bootstrap, recovery keys, "Unable To Decrypt" diagnostics
- Room version 12 quirks (server-less room IDs)
- Debugging checklists for common failure modes

## Install

### Via plugin marketplace (recommended)

```
/plugin marketplace add yncyrydybyl/matrix-skill
/plugin install matrix-expert@matrix-skill
```

### Manual install

Clone and symlink (or copy) the skill into your user skills directory:

```
git clone https://github.com/yncyrydybyl/matrix-skill.git
ln -s "$PWD/matrix-skill/skills/matrix-expert" ~/.claude/skills/matrix-expert
```

## Usage

The skill loads automatically when you're working on Matrix-related code and
Claude's planner decides it's relevant — e.g. questions mentioning
`matrix-sdk`, `MAS`, `sliding sync`, token refresh bugs, E2EE, or cross-signing.

You can also force-load it by invoking the skill by name.

## What's inside

```
matrix-skill/
├── .claude-plugin/
│   ├── plugin.json           # plugin metadata
│   └── marketplace.json      # marketplace manifest (single-plugin)
├── skills/
│   └── matrix-expert/
│       └── SKILL.md          # the skill itself
├── CONTRIBUTING.md
├── LICENSE
└── README.md
```

## Contributing

Corrections, additions, and new sections are welcome — especially for areas
where the Matrix ecosystem is moving fast (MSC drafts, new room versions, MAS
features). See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE)
