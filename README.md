# panagenda plugin marketplace (dev)

Development/test marketplace for the **iDNA Applications Companion** plugin — an AI-assistant
plugin that grounds Notes/Domino estate analysis (usage, modernization, source-code remediation,
application lifecycle) in a curated knowledge wiki, powered by [panagenda iDNA
Applications](https://www.panagenda.com/products/idna-applications/) data.

**Test channel.** This repository carries builds for testing before release. Customers install
from the official repository, [panagenda/idna-companion](https://github.com/panagenda/idna-companion).
Both repositories use the marketplace name `panagenda`; GitHub Copilot and Claude Code register a
marketplace by that name, so they hold only one of the two. To switch, remove this marketplace and
its plugin first, as described under "Switching from an earlier installation" in the official
repository's README.

This repository is a plugin **marketplace** in the Claude plugin format
(`.claude-plugin/marketplace.json`), which is read natively by Cowork, Claude Code, GitHub
Copilot, and OpenAI Codex. One repo serves all of them.

| | Marketplace name | Plugin |
|---|---|---|
| | `panagenda` | `idna-companion` |

## Install

Installing or using the plugin is subject to its license ([plugins/idna-companion/LICENSE](plugins/idna-companion/LICENSE)).

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

### Not supported: claude.ai Chat

The Companion needs its plugin files - the persona and the knowledge wiki - next to its skill.
They are not available to the skill in claude.ai Chat, so the Companion would answer there
without its wiki. Use Cowork, Claude Code, GitHub Copilot or OpenAI Codex.

## Model choice

In GitHub Copilot and OpenAI Codex you choose the AI model per session. The Companion's
restricted-mode discipline - which analysis paths it offers and explains - is strongest on the
most capable models; smaller models follow it less reliably. Organisations that operate under a
restricted posture should standardise on the strongest available model for Companion sessions.
Access to live data is enforced by your iDNA Applications appliance and does not depend on the
model.

## Live data (optional)

The plugin's knowledge wiki works standalone. For live estate data, connect the read-only iDNA
MCP server of your iDNA Applications appliance — setup is documented for your administrators in
the appliance's MCP admin guide (per-user token, HTTPS endpoint). Without the MCP connection the
Companion answers from the wiki in restricted mode and says so.

## Updating

Marketplace-based installs receive updates from this repository: use your client's marketplace
**Update** action (Cowork), `copilot plugin marketplace update panagenda`, then `copilot plugin update idna-companion@panagenda` (Copilot CLI), or reinstall from the refreshed
marketplace.

## Contents

- `.claude-plugin/marketplace.json` — the marketplace manifest
- `plugins/idna-companion/` — the Companion plugin: knowledge wiki (`knowledge/`), the
  `idna-wikilookup` retrieval skill (`skills/`), and the Companion persona
  (`companion-persona.md`)

The plugin tree is generated from the panagenda development repository — do not edit here;
changes land via new releases.
