# Contributing

Thanks for considering a contribution. This skill's value comes from being
concrete, terse, and battle-tested — so contributions should match that bar.

## What makes a good addition

- **Hard-won knowledge.** Things you only learn by hitting the bug. "Two
  `Client` instances race on refresh" > "remember to handle tokens carefully".
- **Concrete.** Symbol names, error strings, specific MSC numbers, specific
  SDK methods. Grep-ability matters.
- **Terse.** A bullet list of bullets, not paragraphs. Readers are scanning.
- **Current.** The Matrix ecosystem moves fast — call out SDK versions or
  MSC states when relevant.

## What to avoid

- Generic Matrix intro content (the spec already does that).
- Long prose explanations; prefer bullets with cheat-sheet density.
- Unverified speculation — flag uncertainty explicitly if you include it.

## Workflow

1. Fork and branch.
2. Edit `skills/matrix-expert/SKILL.md`.
3. Keep edits focused — one topic per PR makes review easier.
4. If you're adding a whole new section, mirror the existing structure:
   a one-liner purpose, then bullets, then a code skeleton if relevant.
5. Bump `version` in `.claude-plugin/plugin.json` and
   `.claude-plugin/marketplace.json` using semver:
   - **patch** for fixes/clarifications
   - **minor** for new sections
   - **major** only if you restructure in a way that breaks existing usage
6. Open a PR describing what you changed and — crucially — what real-world
   problem motivated it.

## Style

- Use `**bold**` for rules and diagnostic keywords ("**Fix**:", "**Diagnosis**:").
- Prefer em-dash (—) over double-hyphen.
- Wrap at ~95 columns.
- Code blocks use triple-backtick with language tag (`rust`, `bash`, `json`).

## Testing your change locally

Install the plugin from a local directory while iterating:

```
# From a Claude Code session
/plugin marketplace add /path/to/matrix-skill
/plugin install matrix-expert@matrix-skill
```

After editing, reload Claude Code or re-run `/plugin install` to pick up
changes.

## License

By contributing you agree that your contributions are licensed under the
project's [MIT license](LICENSE).
