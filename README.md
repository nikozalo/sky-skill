# sky

A Claude skill that answers questions about Sky governance and the Sky protocol, with sources. It
reads a local copy of the Atlas, spells, executive votes, the chainlog and the contract repos, and
also checks the Sky forum.

## Install

### Claude Cowork, the desktop app and claude.ai

**Ask Claude.** In a Cowork task, send:

> Install the skill at https://github.com/nikozalo/sky-skill

Claude shows the skill on a review card. Click **Save** to install it.

**Or upload it yourself:**

1. Download **[sky.zip](https://github.com/nikozalo/sky-skill/releases/latest/download/sky.zip)**.
   Leave it zipped.
2. In Claude, open **Customize → Skills**, upload `sky.zip`, and turn the skill on.

Either way, ask a Sky question and Claude uses the skill, or type `/sky`. You don't need a GitHub
account.

- Skills need **Code execution and file creation**, which is under **Settings → Capabilities**.- The skill downloads the knowledge base (about 12 MB) from `github.com`. On a Team or
  Enterprise plan, the organization's network settings must allow that domain. The Team plan's
  default does.

<details>
<summary>Safari gave you a folder, not a zip</summary>

Safari opens "safe" files after it downloads them, which unpacks the zip. Right-click the folder
in Finder, choose **Compress**, and upload the zip that appears.

</details>

### Claude Code

```bash
git clone https://github.com/nikozalo/sky-skill.git ~/.claude/skills/sky
```

To update, run `git -C ~/.claude/skills/sky pull`.

## Use

```
/sky <question>          # search the knowledge base, then the forum if needed
/sky sync                # update the knowledge base
/sky add repo <url>      # add a repo of your own next to the shared corpus
/sky remove repo <name>
```

Examples: *What does the Atlas say about the Stability Scope?* · *What's the current debt
ceiling on ALLOCATOR-SPARK-A?* · *What changed in the latest executive vote?*

## Updates

- **The knowledge base updates itself.** It is a mirror that CI rebuilds every night. In Cowork,
  the skill fetches a fresh copy in each session. Elsewhere, run `/sky sync`.
- **The skill itself** changes rarely. When it does, download `sky.zip` again and upload it in
  place of the old one. In Claude Code, `git pull`.

## Where the content comes from

The knowledge base is a prebuilt public mirror,
[nikozalo/sky-kb-content](https://github.com/nikozalo/sky-kb-content). It goes in `~/sky-kb`
(about 12 MB) and needs no credentials and no Python.

| Path | What |
|---|---|
| `content/atlas/` | the Atlas, one file per scope, cited by formal ID (`A.2.13.1.1`) |
| `content/laniakea-docs/` | the planned 2026 architecture: **drafts, not live** |
| `content/chainlog-ui/api/mainnet/active.json` | the live chainlog, the ground truth for deployed addresses |
| `content/executive-votes/`, `content/spells-mainnet/` | what governance actually enacted |
| `content/mcd-docs-content/` | MakerDAO-era documentation, for legacy background only |
| `content/_custom/` | repos you added yourself. `sync` leaves them alone |
| `DIRECTORY.md` | index of the mirror, which the skill reads first |

To add a source for everyone, change `presets/sky.json` in `sky-kb-content`. CI picks it up in
the next build. `add repo` only changes your own computer.

## Two things the skill guards against

- **Laniakea is not live.** `laniakea-docs` describes a future design. The chainlog is the
  authority on what is deployed, and any answer that relies on Laniakea is labelled as planned.
- **The newest forum thread wins.** Search ranks by relevance, not date, and years of older
  threads discuss the same parameters. The skill confirms numbers against the newest thread, the
  chainlog or `executive-votes`, and dates every citation.

The forum is searched through a third-party proxy, so queries carry only public Sky terms.

## Files

- `SKILL.md`: the whole skill, in one file so it saves intact when Claude installs it from a chat
- `.github/workflows/release.yml`: on every change to `SKILL.md`, builds `sky.zip` and publishes
  it as the latest release
