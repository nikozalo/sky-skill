---
name: sky
description: Answer questions about Sky governance and the Sky protocol from a local knowledge base of the Atlas, spells, executive votes, the chainlog and the contract repos, plus the Sky forum. Use for Atlas rules and scopes, executive votes and spells, rates and parameters, contract addresses, USDS, SKY, stUSDS and Sky Stars, or to sync the knowledge base.
allowed-tools: Grep(*), Read(*), Edit(*), Write(*), Bash(git clone *), Bash(git -C * fetch *), Bash(git -C * reset *), Bash(git -C * rev-parse *), Bash(git -C * pull *), Bash(git -C * remote *), Bash(git ls-remote *), Bash(test *), Bash(ls *), Bash(cat *), Bash(mkdir *), Bash(mv *), Bash(rm -rf "$KB.new"), Bash(rm -rf "$KB.prev"), Bash(echo *), Bash(grep *), Bash(command -v *), Bash(uname *), Bash(curl -fsSL https://codeload.github.com/nikozalo/sky-kb-content/tar.gz/refs/heads/main), Bash(curl -fsSL https://raw.githubusercontent.com/nikozalo/sky-kb-content/main/manifest.json), Bash(tar *), Bash(brew install git), Bash(winget install *), WebFetch(domain:sky-forum-proxy.skynav.workers.dev)
---

# Sky Knowledge Base

The knowledge base is a **prebuilt public mirror**, rebuilt nightly by CI at
`https://github.com/nikozalo/sky-kb-content`. It lives at one fixed path, `$HOME/sky-kb`, written
below as `<kb>`. Shell variables do not carry over from one command to the next, so every block
sets `KB` on its first line. There is no registry to edit and no setup interview: every flow first
makes sure the mirror is present and fetches it if it is not. Fetching is ~12 MB and takes seconds. In
Cowork's cloud sandbox the filesystem does not survive the session, so this can happen once per
session; do not comment on it, just do it.

## Routing

Parse the user's input to determine the subcommand:

- `sync`, `add repo <url>` or `remove repo <name>`: read `sync-and-repos.md` in this skill's
  folder and follow its flow.
- Anything else: the **Search** flow (default).

**Do not announce which flow you are running. Just execute it.**

---

## Ensure the KB is present

Run this first, in every flow.

```bash
KB="$HOME/sky-kb"; test -f "$KB/manifest.json" && echo present || echo missing
```

- **present** → continue with the flow.
- **missing**, `<kb>` does not exist → **Fetch**, below.
- **missing**, `<kb>` exists and holds `.kb_config.json` → an old locally-synced KB: 43 separate
  clones, possibly with repos the user added themselves. Move it aside with
  `KB="$HOME/sky-kb"; mv "$KB" "$KB.old"`, fetch the mirror, then copy every directory under
  `<kb>.old/content/` that is **not** in the mirror's `content/` into
  `<kb>/content/_custom/<name>/`. Tell the user what was carried over and
  that `<kb>.old` can be deleted once they are satisfied. Never delete it yourself.
- **missing**, `<kb>` exists and is anything else → stop and ask. Never fetch into a directory
  you do not understand.

### Fetch

**1. Pick a method.** Nothing here needs a GitHub account; the mirror is public.

```bash
command -v git >/dev/null 2>&1 && git --version
command -v curl >/dev/null 2>&1 && command -v tar >/dev/null 2>&1 && echo "curl+tar ok"
```

| Found | Method | Why |
|---|---|---|
| `git` | **clone** | ~12 MB; later updates transfer only what changed |
| no `git`, but `curl` + `tar` | **download** | ~18.6 MB; updates re-download the whole corpus, fine at this size |
| neither | **forum-only** | no local corpus; answer from the forum and say so |

Both Cowork environments (the local VM and the cloud sandbox) ship git, so **clone** is the
normal path there. Do not ask anyone to install git. If it is genuinely missing and the person
asks, `brew install git` (macOS) or `winget install --id Git.Git -e --source winget` (Windows);
for Linux print the `sudo apt-get install -y git` line for them to run — never run `sudo`
yourself, it will hang on a password prompt.

**2. Fetch.**

clone:

```bash
KB="$HOME/sky-kb"
git clone --depth 1 https://github.com/nikozalo/sky-kb-content.git "$KB"
mkdir -p "$KB/content/_custom"
grep -qxF '/content/_custom/' "$KB/.git/info/exclude" 2>/dev/null \
  || echo '/content/_custom/' >> "$KB/.git/info/exclude"
```

download:

```bash
KB="$HOME/sky-kb"
mkdir -p "$KB.new"
curl -fsSL https://codeload.github.com/nikozalo/sky-kb-content/tar.gz/refs/heads/main | tar xz -C "$KB.new" --strip-components=1
mv "$KB.new" "$KB"
```

Either way the result is the same tree. If the fetch fails because the network is blocked (an
organization egress policy can allow `github.com` but not `codeload.github.com`), say which host
was unreachable and fall back to forum-only.

**3. forum-only.** Say plainly that the Atlas and chainlog are not available, so citations and
contract addresses cannot be confirmed, then continue with the forum steps of the Search flow.

---

## Search flow

1. **Ensure the KB is present** (see above). If it could not be fetched because neither git nor curl+tar are available, work in **forum-only mode**: skip to step 8, answer from the forum, and say plainly that the local corpus (Atlas, chainlog, contract repos) is not available, so Atlas citations and contract addresses cannot be confirmed.

2. **Read `<kb-path>/DIRECTORY.md`** to understand what content exists and where. It indexes
   the shared mirror only. Anything the user added themselves lives in
   `<kb-path>/content/_custom/` and is not listed there, so check that directory too when it is
   non-empty — and when citing from it, say the source is a local addition rather than part of
   the shared corpus.

3. **Target your search** — based on DIRECTORY.md, identify the most relevant subdirectories for the question. Grep within those specific directories rather than all of `content/`.

4. **Read matching files** to find the answer.

5. For Atlas results: also read the parent scope file for broader context.

6. For protocol questions: check whether the question is about the **live system** or the **planned one** — see [Deployed vs. planned](#deployed-vs-planned) below. `laniakea-docs` describes a future architecture and must not be presented as current behaviour. For "how does X work today", ground the answer in the chainlog, the contract repos, and what governance has actually executed (`spells-mainnet`, `spark-spells`, `executive-votes`). Use `mcd-docs-content` for legacy background.

7. For smart contract address lookups: read `<kb-path>/content/chainlog-ui/api/mainnet/active.json` — this is the live chainlog with all current contract addresses.

8. **Search the forum** — if local KB results are incomplete, or the question is about governance discussions/proposals/community sentiment, also search the Sky forum:
   - Use WebFetch to call `https://sky-forum-proxy.skynav.workers.dev/search?q=<query>&max=5` with prompt "Return the raw JSON"
   - If a topic looks highly relevant, fetch full content: `https://sky-forum-proxy.skynav.workers.dev/topic/<id>` with prompt "Return the raw JSON"
   - For "latest", "recent", or "what's new" questions: call `https://sky-forum-proxy.skynav.workers.dev/latest?max=5` first, then fetch full content for the most relevant topics via `/topic/<id>`
   - **Always prefer the newest thread on a subject** — see [Forum recency](#forum-recency) below
   - Skip if local KB already provides a complete, authoritative answer
   - **The forum proxy is a third party.** A query carries only public Sky terms: never an
     internal name, project or amount from the user's organization, and nothing from a
     confidential document or conversation. If the question needs one, search with the public terms only, or
     answer from the local KB and say the forum was not searched.

### Forum recency

Sky's forum has years of threads on the same recurring subjects — debt ceilings, rate changes, allocator parameters, risk assessments — each superseding the last. **The oldest match is usually the wrong answer.** A thread can also be from the MakerDAO era and use retired terminology (MIPs, DAI, Maker branding) for a mechanism that still exists under a different name.

`/search` ranks by relevance, **not** recency, and returns no dates. Recover recency like this:

- **Topic ID is monotonic — higher id means newer.** This is the main signal available on search results. For calibration: id ~22,000 is Maker-era (2023), ~26,100 is early 2025, ~27,600 is January 2026, ~28,160 is August 2026.
- **Many titles carry an explicit date**, e.g. `[Jan 15, 2026] Parameter Changes - Grove Allocator Vault`. Use it when present.
- **Only `/latest` returns real timestamps** (`created_at`, `last_posted_at`, `posts_count`). `/topic/<id>` returns just `title`, `url`, and `posts` with `username` + `content` — no dates. So a single topic fetch cannot tell you how old it is; judge from the id and the title.

Rules:

- Sort candidate topics by id descending and read the newest relevant one **first**. Only reach for older threads for history, or when the newest doesn't cover the question.
- If old and new threads conflict, the newer wins — and say the parameter or policy changed rather than presenting the stale value.
- **Never quote a number** (rate, ceiling, cap, fee) from an old thread as current. Confirm it against the newest thread, the chainlog, or `executive-votes`, which record what was actually enacted.
- Date what you cite: give the thread's date or note it is older discussion, so the reader can judge staleness.

### Deployed vs. planned

**`content/laniakea-docs/` is a forward-looking design corpus, not a description of the live system.** Its own README calls Laniakea "a comprehensive infrastructure overhaul rolling out through 2026" and states "These documents are drafts under active development."

The vocabulary it defines is largely **unbuilt**: Generator, Halo, Folio, Sentinel Network, teleonomes, synomes, LCTS, NFATs, TEJRC / TISRC / srUSDS, the daily settlement cycle. The mainnet chainlog contains **no** `GENERATOR`, `PRIME`, or `HALO` entries. Reading these docs as current behaviour produces confidently wrong answers.

**Exception — the PAU shipped.** `sky-ecosystem/diamond-pau` ("Parallelized Allocation Unit" — Laniakea says "Prime Allocation Unit") went live in the 2026-07-02 spell: a facet-based Controller with a core-owned Beacon (`PAU_BEACON` in the chainlog, PauseProxy admin). As of 2026-08 Grove and Osero run Diamond PAU Controllers; Spark still runs the older `spark-alm-controller` `MainnetController` (v1.10.0 per `spark-address-registry`). Check `spark-address-registry` / `grove-address-registry` and the newest spell threads for who has migrated.

What is actually live for the same functions:

| Laniakea term | Live mechanism today |
|---|---|
| Generator creating USDS | The **Allocation System** — per-agent allocator ilks (`ALLOCATOR-SPARK-A`, `ALLOCATOR-GROVE-A`, …) in the vat |
| Prime PAU | `ALLOCATOR_<NAME>_A_VAULT` + `_BUFFER` (`dss-allocator`) plus an execution layer: **Diamond PAU** (`content/diamond-pau`) for Grove/Osero, legacy `MainnetController` (`spark-alm-controller`) for Spark |
| Prime credit line | The ilk debt ceiling (`line`), managed by DC-IAM / `MCD_IAM_AUTO_LINE` |
| Rate-limited capital flow | `RateLimits` in the Diamond PAU / ALM controller layer (linear recharge; keys double as the integration whitelist), separate from the vat ceiling |
| Daily settlement cycle | The **Monthly** Settlement Cycle, executed by executive spell |

Rules:

- **Never state Laniakea mechanics as how the protocol works now.** If asked how something works today, answer from live sources first; mention Laniakea only as the direction of travel, explicitly labelled as planned.
- **Label which is which.** When both exist, give the live mechanism and name the Laniakea successor separately.
- **Verify a contract exists before asserting it does** — `content/chainlog-ui/api/mainnet/active.json` is ground truth for what is deployed.
- **`content/laniakea-docs/inactive/`** is superseded even within the Laniakea corpus. Treat it as historical drafts, not as a fallback.
- **Atlas** (`content/atlas/`) is ratified governance policy and mostly binds today, but it contains forward-looking articles too. Check whether an article describes an active process before relying on it. Atlas uses "Prime Agent" for what Laniakea calls a "Prime".

### How to answer

- Be specific and cite your sources (Atlas formal ID like `A.1.2.3`, or file path for repo docs)
- Quote relevant passages when helpful
- When citing forum results, include the topic title and link to the discussion
- Forum posts are community discussion, not official protocol policy — note this distinction when relevant
- Prefer the newest forum thread on a subject and date what you cite, per [Forum recency](#forum-recency) above
- Distinguish deployed behaviour from planned architecture, per [Deployed vs. planned](#deployed-vs-planned) above. If an answer rests on `laniakea-docs`, say so and say it is not live yet
- If the content doesn't contain an answer, say so clearly
