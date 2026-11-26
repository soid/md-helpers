# Markdown Helpers

Setup guides written for [Claude Code](https://claude.com/claude-code) to follow. Each helper is one Markdown file. The top part tells you what it does. The rest is step-by-step instructions that Claude Code works through on your machine: it checks prerequisites, backs up every file before changing it, merges with your existing config, runs tests, and reports what it did.

## Get the helpers

```sh
git clone https://github.com/soid/md-helpers.git ~/md-helpers
```

To get newer versions later:

```sh
git -C ~/md-helpers pull
```

## Helpers

| Helper | What it sets up |
|---|---|
| [iTerm2 tab restore](iterm2-tab-restore-setup.md) | After a restart, every iTerm2 tab comes back with its own zsh history, its tmux session, and its Claude Code conversation |

### iTerm2 tab restore

File: [`iterm2-tab-restore-setup.md`](iterm2-tab-restore-setup.md)

After a reboot, crash or quitting iTerm2, each tab comes back the way it was:

- **Plain shell:** comes back with its own zsh history, so the up arrow shows that tab's commands.
- **Plain shell running Claude Code:** opens in the same folder and runs `claude --resume` with the same conversation.
- **Tab attached to tmux:** reattaches to the same tmux session, with its windows and panes. Panes that were running Claude Code resume their conversations.

New tabs start as plain shells with a copy of the shared history.

**Requirements:** macOS, iTerm2, zsh (the macOS default), Claude Code and Homebrew. tmux is optional.

**Install.** Open Claude Code in an **iTerm2 tab** (not Terminal.app), and give it this prompt:

> Follow the instructions in `~/md-helpers/iterm2-tab-restore-setup.md` to set this up on my machine.

Claude Code asks whether you use tmux, sets everything up, runs the tests, and tells you what to check. macOS may ask whether your terminal can control iTerm2; click **OK**.

**Update.** After `git pull`, if the helper has a newer version, give Claude Code this prompt:

> Apply the updates in `~/md-helpers/iterm2-tab-restore-setup.md` to my machine.

It reads which version your machine has and applies only the missing updates. The helper's Updates section lists what changed in each version.

**Uninstall.** See the Uninstall section at the end of the helper.

## Adding a helper

Each helper follows the same shape:

1. **For people:** what it does, the requirements, and the prompts to install and update it.
2. **For Claude Code:** rules (back up first, merge instead of overwriting, stop and ask on conflicts), numbered steps, and tests that don't touch real data.
3. **Updates:** a version number in the header, plus one section per version with a Check (is this change already on the machine?) and Apply steps. Installed config carries a version line, so an update applies only what's missing.
4. **Everyday use, how it works, limits and uninstall.**

Add the new helper to the table above.
