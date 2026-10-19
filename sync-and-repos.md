# Sync, add a repo, remove a repo

For `/sky sync`, `add repo <url>` and `remove repo <name>`. Every flow starts with **Ensure the
KB is present** in SKILL.md. Shell variables do not carry over from one command to the next, so
every block sets `KB` on its first line.

## Sync flow

1. **Ensure the KB is present.** If it had to be fetched just now, report the build date from
   `manifest.json` and stop: it is already current.

2. **Detect the method.** The directory says which it is:

   ```bash
   KB="$HOME/sky-kb"
   test -d "$KB/.git" && echo clone || echo download
   ```

3. **Check whether anything changed** before transferring anything.

   clone:

   ```bash
   KB="$HOME/sky-kb"
   git -C "$KB" fetch --depth 1 -q origin main
   git -C "$KB" rev-parse HEAD origin/main
   ```

   Matching revisions: say "Everything up to date." and stop.

   download:

   ```bash
   KB="$HOME/sky-kb"
   cat "$KB/manifest.json"
   curl -fsSL https://raw.githubusercontent.com/nikozalo/sky-kb-content/main/manifest.json
   ```

   Equal `commit` fields: say "Everything up to date." and stop.

4. **Update.**

   clone: `fetch --depth 1` plus `reset --hard` rather than `git pull`, so the checkout stays
   shallow and cannot hit a merge conflict. Only tracked files change, so `content/_custom/`
   survives.

   ```bash
   KB="$HOME/sky-kb"
   git -C "$KB" reset --hard origin/main
   ```

   download: replace the tree wholesale so upstream deletions propagate, carrying `_custom`
   across:

   ```bash
   KB="$HOME/sky-kb"
   rm -rf "$KB.new" && mkdir -p "$KB.new"
   curl -fsSL https://codeload.github.com/nikozalo/sky-kb-content/tar.gz/refs/heads/main | tar xz -C "$KB.new" --strip-components=1
   test -d "$KB/content/_custom" && mv "$KB/content/_custom" "$KB.new/content/_custom"
   rm -rf "$KB.prev" && mv "$KB" "$KB.prev" && mv "$KB.new" "$KB" && rm -rf "$KB.prev"
   ```

5. **Update custom repos**, if any and if git is available:

   ```bash
   KB="$HOME/sky-kb"
   for d in "$KB"/content/_custom/*/; do git -C "$d" pull --ff-only; done
   ```

6. **Summarize concisely**: build date and file count from `manifest.json`, plus any custom repos
   updated. If nothing changed: "Everything up to date."

The mirror is only republished when the upstream corpus actually changed, so "no change" is a
normal result. For anything that may have moved in the last day, prefer the forum, per **Forum
recency** in SKILL.md.

## Add Repo flow

Custom repos are the user's own and live in `<kb>/content/_custom/<name>/`, untracked and
excluded through `.git/info/exclude`, so Sync leaves them alone. Never add a repo anywhere else
under `content/`: the next sync would delete it.

1. **Ensure the KB is present.** If `<kb>/.git` does not exist, the KB was installed without git
   and there is no way to clone an extra repo; say so and stop.
2. **Parse arguments:** the repo URL (required) and a name (optional, taken from the URL). The name
   is one folder name: no `/` and no `..`.
3. **Validate:** `git ls-remote <url> HEAD`.
4. **Check for duplicates.** The name must not be in `_custom/` already, nor match a directory in
   the mirror's `content/`:

   ```bash
   KB="$HOME/sky-kb"
   ls "$KB/content/_custom/" "$KB/content/"
   ```

5. **Clone it:**

   ```bash
   KB="$HOME/sky-kb"
   git clone --depth 1 <url> "$KB/content/_custom/<name>"
   ```

6. **Report** what was added and that it is included in searches from now on.

To add a repo to the shared mirror for everyone, change `presets/sky.json` in the
`sky-kb-content` repository instead.

## Remove Repo flow

1. **Ensure the KB is present.**
2. **Parse the repo name.** It is one folder name: no `/` and no `..`.
3. **Locate it:**
   - `<kb>/content/_custom/<name>`: remove it:

     ```bash
     KB="$HOME/sky-kb"
     rm -rf "$KB/content/_custom/<name>"
     ```

   - `<kb>/content/<name>`: part of the shared mirror. Do not delete it; the next sync would
     restore it. Say it is shared and that removing it for everyone is a change to
     `presets/sky.json` in `sky-kb-content`.
4. **Report** what was removed.
