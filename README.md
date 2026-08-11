# panagenda plugin marketplace (dev)

Development/test marketplace for the **iDNA Applications Companion** plugin — an AI-assistant
plugin that grounds Notes/Domino estate analysis (usage, modernization, source-code remediation,
application lifecycle) in a curated knowledge wiki, powered by [panagenda iDNA
Applications](https://www.panagenda.com/products/idna-applications/) data.

This repository is a plugin **marketplace** in the Claude plugin format
(`.claude-plugin/marketplace.json`), which is read natively by Cowork, Claude Code, GitHub
Copilot, and OpenAI Codex. One repo serves all of them.

| | Marketplace name | Plugin |
|---|---|---|
| | `panagenda` | `idna-companion` |

## Install

### Cowork

**Customize → Plugins → Add marketplace** → enter `panfwalder/idna-companion-dev`
(or the full URL `https://github.com/panfwalder/idna-companion-dev`), then install
**iDNA Applications Companion** from the marketplace list.

### GitHub Copilot App

**Settings → Plugins → Install ▾ → Add marketplace** → enter
`https://github.com/panfwalder/idna-companion-dev`, then
**Install ▾ → Install plugin** → `idna-companion@panagenda`.

### GitHub Copilot CLI

```sh
copilot plugin marketplace add panfwalder/idna-companion-dev
copilot plugin install idna-companion@panagenda
```

### Claude Code

```
/plugin marketplace add panfwalder/idna-companion-dev
/plugin install idna-companion@panagenda
```

### OpenAI Codex

```sh
codex plugin marketplace add panfwalder/idna-companion-dev
```

then install `idna-companion` from the marketplace (`/plugins` in the TUI).

## Live data (optional)

The plugin's knowledge wiki works standalone. For live estate data, connect the read-only iDNA
MCP server of your iDNA Applications appliance — setup is documented for your administrators in
the appliance's MCP admin guide (per-user token, HTTPS endpoint). Without the MCP connection the
Companion answers from the wiki in restricted mode and says so.

## Updating

Marketplace-based installs receive updates from this repository: use your client's marketplace
**Update** action (Cowork), `copilot plugin update` (Copilot), or reinstall from the refreshed
marketplace.

## Contents

- `.claude-plugin/marketplace.json` — the marketplace manifest
- `plugins/idna-companion/` — the Companion plugin: knowledge wiki (`knowledge/`), the
  `idna-wikilookup` retrieval skill (`skills/`), and the Companion persona
  (`companion-persona.md`)

The plugin tree is generated from the panagenda development repository — do not edit here;
changes land via new releases.
