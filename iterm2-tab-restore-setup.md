# iTerm2 tab restore: per-tab zsh history, tmux sessions and Claude Code conversations

After a restart (reboot, crash, or quitting iTerm2), every iTerm2 tab comes back the way it was:

| Tab before the restart | Tab after the restart |
|---|---|
| Plain shell | Plain shell, with **its own** zsh history (up-arrow shows that tab's commands) |
| Plain shell running Claude Code | Same folder, `claude --resume <the same conversation>` |
| Attached to a tmux session | Reattached to **that** session, with its windows and panes; panes that were running Claude Code resume their conversations |

New tabs start as plain shells, with a copy of the shared history. Nothing is put in tmux unless you put it there.

**Requirements:** macOS, iTerm2, zsh (the macOS default), Claude Code, and Homebrew (to install `jq` and, optionally, `tmux`). The tmux part is optional.

**Version:** 2. See [Updates](#updates) for what changed in each version.

## How to use this file

Open Claude Code in an iTerm2 tab and say:

> Follow the instructions in `<path to this file>` to set this up on my machine.

Claude Code does the setup, tests it, and tells you what to check. Everything below the line is written for Claude Code.

If it's already set up and this file has a newer version, say:

> Apply the updates in `<path to this file>` to my machine.

Claude Code then applies only the updates your machine doesn't have yet.

---

## Instructions for Claude Code

Set up the system described above on this machine. Work through the steps in order.

**Already installed?** If `~/.zshrc` already contains `# >>> iterm2-tab-restore >>>`, this is an update, not a new install: follow [Updates](#updates) instead of steps 1–9, unless the user asks for a full reinstall.

Rules:

- **Back up** every file before you change it: `cp <file> <file>.bak-$(date +%Y%m%d-%H%M%S)`.
- **Merge, don't overwrite.** Keep the user's existing `~/.zshrc`, tmux config and `~/.claude/settings.json` content. Add the blocks given here. If a block from an earlier install is already there, replace it.
- **Use absolute paths** where this file says `<HOME>`. Replace it with the real value of `$HOME`.
- **Stop and ask the user** before doing anything this file doesn't describe, or if something on the machine conflicts with it. Conflicts to look out for: a different tool already managing shell history (atuin, zsh-histdb, mcfly), a `HISTFILE` set somewhere other than `~/.zshrc`, or tmux started automatically by another script.
- **Never start a real `claude` process in tests.** The tests below stub it out.
- At the end, report what you changed, which test results passed, and anything you skipped.

### Step 1: Check prerequisites

```sh
sw_vers -productVersion; echo "TERM_PROGRAM=$TERM_PROGRAM ITERM_SESSION_ID=$ITERM_SESSION_ID SHELL=$SHELL"
command -v brew jq tmux claude; ls ~/.tmux/plugins 2>/dev/null
ls -la ~/.zshrc ~/.tmux.conf ~/.config/tmux/tmux.conf ~/.claude/settings.json 2>/dev/null
grep -nE 'HISTFILE|HISTSIZE|SAVEHIST|SHARE_HISTORY|oh-my-zsh|atuin|histdb|tmux' ~/.zshrc ~/.zprofile ~/.zshenv 2>/dev/null
```

- `ITERM_SESSION_ID` must be set. If it isn't, this isn't running inside iTerm2: tell the user to open Claude Code in an iTerm2 tab and stop.
- The login shell must be zsh.
- Install `jq` if it's missing: `brew install jq`.
- **tmux part:** ask the user whether they use tmux. If yes, install it if missing (`brew install tmux`) and do steps 4 and 5. If no, skip steps 4 and 5. The tmux code in `~/.zshrc` is inactive when tmux isn't installed.

### Step 2: Turn on iTerm2 window restoration

```sh
defaults read NSGlobalDomain NSQuitAlwaysKeepsWindows 2>/dev/null
defaults read com.googlecode.iterm2 2>/dev/null | grep -E 'OpenArrangementAtStartup|OpenNoWindowsAtStartup|NoSyncIgnoreSystemWindowRestoration'
```

iTerm2 must restore its windows on launch. Then each tab comes back with the same ID (`$ITERM_SESSION_ID`), which this whole setup depends on.

- `NSQuitAlwaysKeepsWindows` should be `1`. This means macOS's "Close windows when quitting an application" (System Settings → Desktop & Dock) is **off**.
- `OpenArrangementAtStartup` and `OpenNoWindowsAtStartup` should be absent or `0`. In iTerm2 → Settings → General → Startup, *Window restoration policy* should be "Use System Window Restoration Setting".
- If either is wrong, tell the user and ask before changing it. You can fix the macOS one with `defaults write NSGlobalDomain NSQuitAlwaysKeepsWindows -bool true`. The iTerm2 one is best changed in its Settings window.

### Step 3: Create the helper scripts

Create `~/.local/bin` if needed, write these four files, and `chmod +x` them. They run from tmux and Claude Code hooks with a minimal environment, so each sets its own `PATH`.

**`~/.local/bin/claude-pane-record`** (Claude Code `SessionStart` hook):

```sh
#!/bin/sh
# Claude Code SessionStart hook: remember which conversation runs where.
# - In a tmux pane: ~/.zsh_sessions/iterm2/claude/<session>_<window>_<pane>
#   (claude-pane-resume reads it after tmux-resurrect restores the pane).
# - In a plain iTerm2 tab: ~/.zsh_sessions/iterm2/<tab-uuid>.claude
#   (the tab's next shell start resumes it; see ~/.zshrc).
export PATH="$HOME/.local/bin:/opt/homebrew/bin:/usr/local/bin:$PATH"
input=$(cat)
sid=$(printf '%s' "$input" | jq -r '.session_id // empty')
cwd=$(printf '%s' "$input" | jq -r '.cwd // empty')
[ -n "$sid" ] || exit 0
dir="$HOME/.zsh_sessions/iterm2"
mkdir -p "$dir/claude"
if [ -z "$TMUX_PANE" ]; then
  [ -n "$ITERM_SESSION_ID" ] || exit 0
  printf '%s\t%s\n' "$sid" "${cwd:-$PWD}" > "$dir/${ITERM_SESSION_ID#*:}.claude"
  exit 0
fi
key=$(tmux display-message -p -t "$TMUX_PANE" '#{session_name}_#{window_index}_#{pane_index}' 2>/dev/null | tr '/' '_')
[ -n "$key" ] || exit 0
printf '%s\t%s\n' "$sid" "${cwd:-$PWD}" > "$dir/claude/$key"
exit 0
```

**`~/.local/bin/claude-pane-end`** (Claude Code `SessionEnd` hook):

```sh
#!/bin/sh
# Claude Code SessionEnd hook: when the user quits Claude in a plain iTerm2
# tab (/exit, Ctrl-D), forget it so the tab doesn't reopen Claude after a
# restart. A shutdown or kill ends with reason "other" and keeps the record.
export PATH="$HOME/.local/bin:/opt/homebrew/bin:/usr/local/bin:$PATH"
input=$(cat)
reason=$(printf '%s' "$input" | jq -r '.reason // empty')
sid=$(printf '%s' "$input" | jq -r '.session_id // empty')
dir="$HOME/.zsh_sessions/iterm2"
mkdir -p "$dir"
printf '%s %s %s tmux=%s\n' "$(date '+%F %T')" "$reason" "$sid" "${TMUX_PANE:-no}" >> "$dir/claude-end.log"
tail -n 500 "$dir/claude-end.log" > "$dir/claude-end.log.tmp" && mv "$dir/claude-end.log.tmp" "$dir/claude-end.log"
[ -z "$TMUX_PANE" ] && [ -n "$ITERM_SESSION_ID" ] || exit 0
case "$reason" in
  prompt_input_exit|logout)
    rec="$dir/${ITERM_SESSION_ID#*:}.claude"
    [ -f "$rec" ] && [ "$(cut -f1 "$rec")" = "$sid" ] && rm -f "$rec" ;;
esac
exit 0
```

**`~/.local/bin/claude-pane-resume`** (tmux-resurrect runs it in panes where Claude was running):

```zsh
#!/bin/zsh
# Run by tmux-resurrect in panes where Claude was running at the last save.
# Resumes the conversation last recorded for this pane (claude-pane-record);
# falls back to the most recent conversation in the pane's folder.
export PATH="$HOME/.local/bin:$HOME/.claude/local:/opt/homebrew/bin:/usr/local/bin:$PATH"
key=$(tmux display-message -p -t "$TMUX_PANE" '#{session_name}_#{window_index}_#{pane_index}' 2>/dev/null)
rec="$HOME/.zsh_sessions/iterm2/claude/${key//\//_}"
if [[ -n "$key" && -f "$rec" ]]; then
  IFS=$'\t' read -r sid cwd < "$rec"
  [[ -d "$cwd" ]] && cd "$cwd"
  [[ -n "$sid" ]] && exec claude --resume "$sid"
fi
exec claude --continue
```

**`~/.local/bin/tmux-iterm-sync`** (tmux hooks run it; only needed with tmux):

```sh
#!/bin/sh
# Called from tmux hooks. Records which tmux session each iTerm2 tab shows,
# so the tab can reattach to the same session after a restart.
#   tty/<tty>    -> iTerm2 tab UUID (written by the tab's outer zsh)
#   <uuid>.tmux  -> tmux session name
export PATH="/opt/homebrew/bin:/usr/local/bin:$PATH"
dir="$HOME/.zsh_sessions/iterm2"
tmux list-clients -F '#{client_tty} #{session_name}' 2>/dev/null |
while read -r tty name; do
  f="$dir/tty/${tty##*/}"
  [ -f "$f" ] || continue
  uuid=$(cat "$f")
  [ -n "$uuid" ] && printf '%s\n' "$name" > "$dir/$uuid.tmux"
done
exit 0
```

Then check them: `for f in claude-pane-record claude-pane-end tmux-iterm-sync; do sh -n ~/.local/bin/$f; done; zsh -n ~/.local/bin/claude-pane-resume`.

### Step 4 (tmux only): Install TPM, tmux-resurrect and tmux-continuum

Skip any that are already installed.

```sh
[ -d ~/.tmux/plugins/tpm ] || git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```

Plugins are installed in step 5, after the config lists them.

### Step 5 (tmux only): Configure tmux

Use the existing config file: `~/.tmux.conf`, or `~/.config/tmux/tmux.conf` if that's the one in use. Create `~/.tmux.conf` if neither exists. Add this block, replacing `<HOME>`:

```tmux
# >>> iterm2-tab-restore >>>
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-resurrect'
set -g @plugin 'tmux-plugins/tmux-continuum'
# ~/.zshrc restores the tmux-resurrect save when the first tab needs tmux
# after a restart, and makes other tabs wait for it. Continuum only saves.
set -g @continuum-restore 'off'
set -g @continuum-save-interval '5'
# Restart Claude Code in panes where it was running, resuming the same conversation
set -g @resurrect-processes '"~claude-><HOME>/.local/bin/claude-pane-resume"'
# Killing the session you're in switches to another session instead of detaching
set -g detach-on-destroy off
# Status bar: up to 20 characters of the session name, always followed by '] '.
# (The default cuts '[name] ' at 10 characters, so a long name runs into the
# window list: '[138street0:zsh' instead of '[138street_com] 0:zsh'.)
set -g status-left '[#{=20:session_name}] '
set -g status-left-length 25
# Remember which tmux session each iTerm2 tab shows
set-hook -g 'client-attached[90]' 'run-shell -b ~/.local/bin/tmux-iterm-sync'
set-hook -g 'client-session-changed[90]' 'run-shell -b ~/.local/bin/tmux-iterm-sync'
set-hook -g 'session-renamed[90]' 'run-shell -b ~/.local/bin/tmux-iterm-sync'
# <<< iterm2-tab-restore <<<
```

Merging rules:

- Put the block **above** the `run '~/.tmux/plugins/tpm/tpm'` line. That line must stay last in the file; add it if it's missing.
- Remove duplicate `@plugin` lines.
- If the user's config has `@continuum-restore 'on'`, remove it. It would race with the restore in `~/.zshrc`.
- If `@resurrect-processes` is already set, merge the two lists into one quoted list, for example `'"~claude->..." ssh'`.
- If the user's config sets its own `status-left` or `status-left-length`, or uses a theme plugin that styles the status bar, leave the two `status-left` lines out of the block and keep the user's.
- If a tmux server is running, run `tmux source-file <config>`.
- Install the plugins: `~/.tmux/plugins/tpm/bin/install_plugins`.

### Step 6: Add the zsh block

Append this block at the **very end** of `~/.zshrc`. It must come after oh-my-zsh, prezto and anything else that sets history options, because oh-my-zsh turns `SHARE_HISTORY` on. If `~/.zshrc` already sets a larger `HISTSIZE` or `SAVEHIST`, keep the larger value.

```zsh
# >>> iterm2-tab-restore >>>
# iterm2-tab-restore version: 2
# Each iTerm2 tab comes back after a restart the way it was: its own history,
# its tmux session, or its Claude Code conversation.
# Data: ~/.zsh_sessions/iterm2/   Opt out for one shell: NO_TAB_RESTORE=1 zsh

# --- Per-tab history -----------------------------------------------------------
# iTerm2 keeps each tab's ID ($ITERM_SESSION_ID) when it restores windows, so
# the history file is named after it. Inside tmux, $ITERM_SESSION_ID comes from
# whichever tab started tmux, so panes are named after session/window/pane,
# which tmux-resurrect restores.
HISTSIZE=50000
SAVEHIST=50000
setopt INC_APPEND_HISTORY   # write each command right away (survives a crash or reboot)
setopt HIST_IGNORE_DUPS
unsetopt SHARE_HISTORY      # keep tabs separate

_iterm_hist_dir="$HOME/.zsh_sessions/iterm2"
_hist_key=""
if [[ -n "$TMUX" && -n "$TMUX_PANE" ]]; then
  _hist_key="tmux-$(tmux display-message -p -t "$TMUX_PANE" '#{session_name}_#{window_index}_#{pane_index}' 2>/dev/null)"
  _hist_key="${_hist_key//\//_}"
  [[ "$_hist_key" == "tmux-" ]] && _hist_key=""
elif [[ -n "$ITERM_SESSION_ID" ]]; then
  _hist_key="${ITERM_SESSION_ID#*:}"
fi

autoload -Uz add-zsh-hook
if [[ -n "$_hist_key" ]]; then
  [[ -d "$_iterm_hist_dir" ]] || mkdir -p "$_iterm_hist_dir"
  _global_histfile="$HOME/.zsh_history"
  HISTFILE="$_iterm_hist_dir/$_hist_key.history"

  # Brand-new tab or pane: start from the shared history.
  if [[ ! -f "$HISTFILE" && -f "$_global_histfile" ]]; then
    cp "$_global_histfile" "$HISTFILE"
    fc -R "$HISTFILE"
  fi

  # Also append every command to the shared history, so new tabs see all of it.
  _iterm_global_hist() {
    local cmd="${1%%$'\n'}"
    [[ -n "$cmd" ]] && print -r -- "${cmd//$'\n'/\\$'\n'}" >> "$_global_histfile"
    return 0
  }
  add-zsh-hook zshaddhistory _iterm_global_hist

  # Delete history files of tabs not used for 90 days.
  find "$_iterm_hist_dir" -name '*.history' -mtime +90 -delete 2>/dev/null
fi

# --- Restore each tab the way it was ---------------------------------------
# - Tab that was in tmux: reattach to its session (<tab-uuid>.tmux, kept
#   current by tmux hooks). After a reboot, the first such tab restores the
#   tmux-resurrect save; other tabs wait for it.
# - Plain tab that was running Claude Code: resume that conversation in the
#   tab (<tab-uuid>.claude, written by the claude-pane-record hook).
# - Any other tab: plain shell.
_tmux_restore_if_needed() {
  local lock="$_iterm_hist_dir/tmux-restore.lock"
  # Stale lock (crash during a restore)? Remove it after 3 minutes.
  [[ -d "$lock" && -n "$(find "$lock" -maxdepth 0 -mmin +3 2>/dev/null)" ]] && rmdir "$lock" 2>/dev/null
  command tmux has-session 2>/dev/null && [[ ! -d "$lock" ]] && return 0
  [[ -x ~/.tmux/plugins/tmux-resurrect/scripts/restore.sh ]] || return 0
  if mkdir "$lock" 2>/dev/null; then
    if ! command tmux has-session 2>/dev/null; then
      print -P "%F{8}Restoring tmux sessions...%f"
      command tmux new-session -d -s __restoring
      command tmux run-shell ~/.tmux/plugins/tmux-resurrect/scripts/restore.sh
      command tmux kill-session -t '=__restoring' 2>/dev/null
    fi
    rmdir "$lock"
  else
    print -P "%F{8}Waiting for tmux restore...%f"
    local i
    for i in {1..240}; do [[ -d "$lock" ]] || break; sleep 0.5; done
  fi
}

if [[ -o interactive && -z "$TMUX" && -z "$NO_TAB_RESTORE" && -z "$ZSH_EXECUTION_STRING" \
      && "$TERM_PROGRAM" == "iTerm.app" && -n "$ITERM_SESSION_ID" && -t 0 && -t 1 ]]; then
  _iterm_uuid="${ITERM_SESSION_ID#*:}"
  _tab_restored=""

  if (( $+commands[tmux] )); then
    # Lets tmux-iterm-sync tell which tab a tmux client is in.
    mkdir -p "$_iterm_hist_dir/tty"
    print -r -- "$_iterm_uuid" > "$_iterm_hist_dir/tty/${TTY:t}"
    _iterm_tty_cleanup() { rm -f "$_iterm_hist_dir/tty/${TTY:t}" }
    add-zsh-hook zshexit _iterm_tty_cleanup

    # Starting tmux by hand after a reboot restores the save first.
    tmux() { _tmux_restore_if_needed; command tmux "$@" }
    # Make this tab open as a plain shell after restarts.
    tmux-forget() { rm -f "$_iterm_hist_dir/$_iterm_uuid.tmux"; echo "This tab will open as a plain shell after restarts." }

    _tmux_target=""
    [[ -f "$_iterm_hist_dir/$_iterm_uuid.tmux" ]] && _tmux_target="$(<"$_iterm_hist_dir/$_iterm_uuid.tmux")"
    if [[ -n "$_tmux_target" ]]; then
      _tab_restored=1
      _tmux_restore_if_needed
      if command tmux has-session -t "=$_tmux_target" 2>/dev/null; then
        command tmux attach-session -t "=$_tmux_target"
      else
        print -P "%F{8}tmux session '$_tmux_target' for this tab no longer exists (tmux ls to list).%f"
      fi
    fi
    unset _tmux_target
  fi

  if [[ -z "$_tab_restored" && -f "$_iterm_hist_dir/$_iterm_uuid.claude" ]] && (( $+commands[claude] )); then
    IFS=$'\t' read -r _cc_sid _cc_cwd < "$_iterm_hist_dir/$_iterm_uuid.claude"
    rm -f "$_iterm_hist_dir/$_iterm_uuid.claude"   # the resumed Claude records it again
    [[ -d "$_cc_cwd" ]] && cd "$_cc_cwd"
    print -P "%F{8}Resuming Claude session $_cc_sid%f"
    claude --resume "$_cc_sid"
    unset _cc_sid _cc_cwd
  fi
  unset _tab_restored
fi
# <<< iterm2-tab-restore <<<
```

Check it with `zsh -n ~/.zshrc`.

If `claude` is only an alias on this machine (old installs used `alias claude=~/.claude/local/claude`), `$+commands[claude]` is false, so tell the user. The fix is to put `~/.claude/local` on `PATH` in `~/.zshrc`, or to run `claude migrate-installer`.

### Step 7: Register the Claude Code hooks

Add both hooks to `~/.claude/settings.json` and keep everything else. Back the file up first.

```sh
python3 -I - <<'EOF'
import json, os
p = os.path.expanduser('~/.claude/settings.json')
d = json.load(open(p)) if os.path.exists(p) else {}
hooks = d.setdefault('hooks', {})
for event, script in [('SessionStart', 'claude-pane-record'), ('SessionEnd', 'claude-pane-end')]:
    cmd = os.path.expanduser('~/.local/bin/' + script)
    entries = hooks.setdefault(event, [])
    if not any(h.get('command') == cmd for e in entries for h in e.get('hooks', [])):
        entries.append({'hooks': [{'type': 'command', 'command': cmd}]})
os.makedirs(os.path.dirname(p), exist_ok=True)
with open(p, 'w') as f:
    json.dump(d, f, indent=2, ensure_ascii=False)
    f.write('\n')
EOF
jq '.hooks.SessionStart, .hooks.SessionEnd' ~/.claude/settings.json
```

### Step 8: Register what's already open

The setup records tabs and conversations as they start. Things already running were started before it existed, so register them now. This uses AppleScript, so macOS may ask the user to allow the terminal to control iTerm2. Tell them to click **OK**.

```sh
D=~/.zsh_sessions/iterm2; mkdir -p "$D/tty" "$D/claude"
TABS=$(mktemp)
osascript -e 'tell application "iTerm2"
set out to ""
repeat with w in windows
repeat with t in tabs of w
repeat with s in sessions of t
set out to out & (tty of s) & " " & (id of s) & linefeed
end repeat
end repeat
end repeat
return out
end tell' > "$TABS"

# tmux: tab -> session currently shown
if command -v tmux >/dev/null && tmux has-session 2>/dev/null; then
  while read -r tty uuid; do [ -n "$uuid" ] && echo "$uuid" > "$D/tty/${tty##*/}"; done < "$TABS"
  ~/.local/bin/tmux-iterm-sync
  tmux list-panes -a -F '#{pane_pid}	#{session_name}_#{window_index}_#{pane_index}' |
  while IFS=$'\t' read -r ppid key; do
    for pid in $(pgrep -P "$ppid"); do
      f=~/.claude/sessions/$pid.json; [ -f "$f" ] || continue
      printf '%s\t%s\n' "$(jq -r .sessionId "$f")" "$(jq -r .cwd "$f")" > "$D/claude/${key//\//_}"
      echo "tmux pane $key -> Claude $(jq -r .sessionId "$f")"
    done
  done
fi

# Claude running in plain tabs
for f in ~/.claude/sessions/*.json; do
  [ -f "$f" ] || continue
  pid=$(jq -r .pid "$f"); [ "$(jq -r '.tmux // "null"' "$f")" = "null" ] || continue
  tty=$(ps -o tty= -p "$pid" 2>/dev/null | tr -d ' '); [ -n "$tty" ] && [ "$tty" != "??" ] || continue
  uuid=$(awk -v t="/dev/$tty" '$1==t{print $2}' "$TABS"); [ -n "$uuid" ] || continue
  printf '%s\t%s\n' "$(jq -r .sessionId "$f")" "$(jq -r .cwd "$f")" > "$D/$uuid.claude"
  echo "plain tab $tty -> Claude $(jq -r .sessionId "$f")  ($(jq -r .cwd "$f"))"
done
rm -f "$TABS"
grep -H . "$D"/*.tmux 2>/dev/null
```

`~/.claude/sessions/<pid>.json` is Claude Code's internal state, and its format may change. If those files are missing or have no `sessionId` field, skip the Claude part and tell the user. Conversations already running will then come back via `claude --continue` in tmux panes. In plain tabs they won't come back this one time. Everything started from now on is recorded by the hooks.

If a tmux server is running, finish with a save: `tmux run-shell ~/.tmux/plugins/tmux-resurrect/scripts/save.sh`.

### Step 9: Test

Run these tests and report each result. None of them touches the user's real tmux sessions or starts a real `claude`.

```sh
# 1. Per-tab history: a new interactive shell uses this tab's history file
zsh -i -c 'echo "HISTFILE=$HISTFILE"' 2>/dev/null | tail -1
#    expect: ~/.zsh_sessions/iterm2/<this tab's ID>.history

# 2. Hooks: the start hook records, and the end hook keeps on "other" and deletes on a normal quit
U=TEST-$$; echo "{\"session_id\":\"s1\",\"cwd\":\"/tmp\"}" | env -u TMUX_PANE ITERM_SESSION_ID=w0:$U ~/.local/bin/claude-pane-record
cat ~/.zsh_sessions/iterm2/$U.claude                     # expect: s1<TAB>/tmp
echo '{"reason":"other","session_id":"s1"}' | env -u TMUX_PANE ITERM_SESSION_ID=w0:$U ~/.local/bin/claude-pane-end
[ -f ~/.zsh_sessions/iterm2/$U.claude ] && echo "kept on shutdown: OK"
echo '{"reason":"prompt_input_exit","session_id":"s1"}' | env -u TMUX_PANE ITERM_SESSION_ID=w0:$U ~/.local/bin/claude-pane-end
[ -f ~/.zsh_sessions/iterm2/$U.claude ] || echo "deleted on /exit: OK"
sed -i '' '/ s1 tmux=no$/d' ~/.zsh_sessions/iterm2/claude-end.log   # drop the test entries from the log
```

With tmux only:

```sh
# 3. Pane record and resume lookup (claude stubbed out)
S=$(mktemp -d)
tmux new-session -d -s __cctest; P=$(tmux list-panes -t __cctest -F '#{pane_id}')
echo '{"session_id":"test-123","cwd":"/tmp"}' | TMUX_PANE=$P ~/.local/bin/claude-pane-record
sed 's/exec claude/echo would run: claude/' ~/.local/bin/claude-pane-resume > "$S/resume-test"
TMUX_PANE=$P zsh "$S/resume-test" | head -1               # expect: would run: claude --resume test-123
tmux kill-session -t '=__cctest'; rm -rf "$S" ~/.zsh_sessions/iterm2/claude/__cctest_0_0 ~/.zsh_sessions/iterm2/tmux-__cctest_0_0.history

# 4. Restoring the latest save works (separate tmux server; skip if there's no save yet)
T=(tmux -L restoretest -f /dev/null)
$T new-session -d -s __restoring; $T run-shell ~/.tmux/plugins/tmux-resurrect/scripts/restore.sh
$T kill-session -t '=__restoring'; $T ls; $T kill-server   # expect: the saved sessions

# 5. Config loaded
tmux show -g @resurrect-processes; tmux show -g @continuum-restore; tmux show-hooks -g | grep iterm-sync
tmux show -g status-left; tmux show -g status-left-length   # expect: "[#{=20:session_name}] " and 25 (unless the user's own were kept)
```

Run each numbered test as one command. Then tell the user:

1. **Open a new iTerm2 tab** and run `echo $HISTFILE`. It should point to `~/.zsh_sessions/iterm2/<ID>.history`. Tabs that are already open keep using the old setup until they restart; to switch one now, run `exec zsh` in it.
2. **With tmux:** in a tab, run `tmux attach -t <name>` (or `tmux`). The tab remembers that session from then on. Press `prefix Ctrl-s` before a planned restart to save right away.
3. **Restart.** In the macOS Restart dialog, leave "Reopen windows when logging back in" checked.

## Updates

Each version below lists what changed since the one before, a **Check** that shows whether a machine already has the change, and the steps to **Apply** it to an existing install. A new install (steps 1–9) already includes every version.

| Version | Date | Change | Touches |
|---|---|---|---|
| 1 | 2026-11-25 | First release | — |
| 2 | 2026-11-25 | tmux status bar shows the whole session name | tmux config; version line in `~/.zshrc` |

### How to update an existing install

1. **Find the installed version.** No output means version 1, which had no version line.
   ```sh
   sed -n 's/^# iterm2-tab-restore version: //p' ~/.zshrc
   ```
2. **Apply each newer version in order.** Run its Check. If the Check prints `missing`, back up the files listed under Touches and follow its Apply steps. If it prints `applied` or `skip`, leave it. The Checks make it safe to run an update twice.
3. **Set the version line** in the `~/.zshrc` block to the current version. This replaces an existing version line, or adds one directly under `# >>> iterm2-tab-restore >>>`. It writes through `cat` so a symlinked `~/.zshrc` stays a symlink.
   ```sh
   V=2   # the current version
   cp ~/.zshrc ~/.zshrc.bak-$(date +%Y%m%d-%H%M%S)
   tmp=$(mktemp)
   awk -v v="$V" '/^# iterm2-tab-restore version: /{next} {print} /^# >>> iterm2-tab-restore >>>$/{print "# iterm2-tab-restore version: " v}' ~/.zshrc > "$tmp" &&
     cat "$tmp" > ~/.zshrc; rm -f "$tmp"
   zsh -n ~/.zshrc && sed -n 's/^# iterm2-tab-restore version: //p' ~/.zshrc   # expect: the new version, once
   ```
4. **Report** which versions you applied, which you skipped and why, and the result of each Check afterwards.

The same rules apply as for a new install: back up before changing anything, merge instead of overwriting, and stop and ask if something on the machine conflicts.

### Version 2 (2026-11-25): whole session name in the tmux status bar

tmux only; skip this on machines without tmux. tmux's default status bar shows `[session name] ` but cuts it off at 10 characters. A long name then runs into the window list, for example `[138street0:zsh` instead of `[138street_com] 0:zsh`. Version 2 shortens only the name itself (to 20 characters), so the `] ` and space always stay.

**Check.** Set `conf` to the tmux config from step 5.
```sh
conf=~/.tmux.conf   # or ~/.config/tmux/tmux.conf if that's the one in use
if grep -qF "set -g status-left '[#{=20:session_name}] '" "$conf"; then echo applied
elif grep -qE '^[[:space:]]*set(-option)?[[:space:]].*status-left' "$conf"; then echo "skip: the user sets their own status-left"
else echo missing; fi
```
Also skip if the user's config loads a theme plugin that styles the status bar.

**Apply.**
1. Back up the tmux config.
2. In its `# >>> iterm2-tab-restore >>>` block, add these lines directly below `set -g detach-on-destroy off`:
   ```tmux
   # Status bar: up to 20 characters of the session name, always followed by '] '.
   # (The default cuts '[name] ' at 10 characters, so a long name runs into the
   # window list: '[138street0:zsh' instead of '[138street_com] 0:zsh'.)
   set -g status-left '[#{=20:session_name}] '
   set -g status-left-length 25
   ```
3. If a tmux server is running, run `tmux source-file <config>`.
4. Run the Check again (expect `applied`) and the last line of test 5.

### Version 1 (2026-11-25): first release

Steps 1–9 as first published. Installs of this version have no `# iterm2-tab-restore version:` line in `~/.zshrc`.

---

## For users: everyday use

- **New tab:** plain shell, starting with a copy of the shared history.
- **Using tmux in a tab:** run `tmux` or `tmux attach -t name` in it. The tab remembers the session, including when you switch sessions (`prefix s`) or rename one (`prefix $`).
- **Leaving tmux in a tab:** detach with `prefix d`. To make the tab open as a plain shell after restarts, run `tmux-forget` in it.
- **Claude Code:** just use it. If it's running when the computer restarts, it comes back with the same conversation, in tmux panes and plain tabs. If you quit it with `/exit` or Ctrl-D, it doesn't come back.
- **A tab came back on the wrong tmux session:** switch with `prefix s`. The tab remembers the fix.
- **One shell without any of this:** `NO_TAB_RESTORE=1 zsh`.

## How it works

- **Tab IDs:** iTerm2 saves each tab's ID in its window-restoration state and gives it back when it restores the tab. Everything is keyed on that ID. Data lives in `~/.zsh_sessions/iterm2/`.
- **History:** `HISTFILE` is set per tab (per pane inside tmux). Every command is written immediately and also appended to `~/.zsh_history`.
- **tmux:** tmux-resurrect and tmux-continuum save every 5 minutes. tmux hooks record which session each tab shows. After a restart, the first tab that needs tmux runs the restore while a lock makes the other tabs wait, so restores never race.
- **Claude Code:** a `SessionStart` hook records the conversation ID per tab or pane, and updates it on `/clear` or `/resume`. tmux-resurrect restarts `claude` in panes where it was running at the last save, and `claude-pane-resume` picks the recorded conversation. A plain tab resumes its conversation on its next shell start. A `SessionEnd` hook deletes the record when you quit Claude yourself. A shutdown ends sessions with reason `other`, which keeps the record. The log is in `~/.zsh_sessions/iterm2/claude-end.log`.

## Limits

- Claude resumes with default options. Extra command-line flags aren't reused, but `settings.json` still applies.
- In tmux, a Claude started less than 5 minutes before a crash may not be in the last save. Press `prefix Ctrl-s` to save right away.
- Renaming a tmux session starts new history files for its panes, copied from the shared history. In that session, Claude falls back to `claude --continue` until it's restarted.
- Multi-line or non-ASCII commands may not be copied into `~/.zsh_history` exactly. The tab's own history keeps them correctly.

## Uninstall

1. Delete the `# >>> iterm2-tab-restore >>>` … `# <<< iterm2-tab-restore <<<` blocks from `~/.zshrc` and the tmux config.
2. Remove the two `~/.local/bin/claude-pane-*` hook entries from `~/.claude/settings.json`.
3. Delete `~/.local/bin/{claude-pane-record,claude-pane-end,claude-pane-resume,tmux-iterm-sync}`.
4. Delete `~/.zsh_sessions/iterm2/` (this deletes the per-tab histories; your shared `~/.zsh_history` stays).
