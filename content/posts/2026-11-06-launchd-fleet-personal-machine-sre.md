---
title: "A Launchd Fleet as Personal Machine SRE"
tags: [macos, launchd, devops, automation]
date: 2026-11-06
slug: launchd-fleet-personal-machine-sre
toc: true
description: "The personal profile loads brew-upgrade and nvim-update. The work profile uses DF_LAUNCHD to load a much larger set from the same launchd directory."
---

My work Mac loads many launchd agents, and the personal Mac loads two. Both machines install from the same clone. `DF_LAUNCHD` in the profile is the list that differs.

## DF_LAUNCHD replaced a module per agent

I listed each agent in `DF_MODULES`, under `modules/launchd-<name>/`. `hooks/postinstall` was the whole module, and every copy called `launchd_install`.

Agents now live under `launchd/`. A profile lists them in `DF_LAUNCHD`, as it lists `DF_MACOS_DEFAULTS`. `install.sh` checks for `launchd/<name>/com.user.<name>.plist` and `launchd/<name>/<name>.sh`. `launchd_apply` in `lib/steps.sh` loads only those names.

```bash
for name in "${DF_LAUNCHD[@]}"; do
  requires_satisfied "$DOTFILES_DIR/launchd/$name/requires" "launchd:$name" || continue
  launchd_install "$DOTFILES_DIR/launchd/$name/com.user.$name.plist"
done
```

An empty `DF_LAUNCHD` loads no agents, and `--no-launchd` skips the step. `--only` and `--skip` filter modules and do not filter agents. The `.sh` file stays in the clone. `launchd_install` replaces `__HOME__` and `__DOTFILES__` in the plist under `~/Library/LaunchAgents`, and it creates `~/.local/state/<name>/` because launchd does not create the log directory. `launchctl bootstrap` runs only if the rendered plist changed or the label is not loaded.

In October 2026 each `launchd/<name>.sh` moved into `launchd/<name>/<name>.sh`. `skill-upstream` and `pi-upstream` keep `package.json` and tests next to the shell script that starts the `.ts` file. If I remove a name from `DF_LAUNCHD`, that agent stays loaded until I run `launchctl bootout` and delete the plist.

## Personal DF_LAUNCHD lists two agents

`profiles/personal/profile.conf` sets the personal list.

```bash
DF_LAUNCHD=(
  brew-upgrade
  nvim-update
)
```

Both plists set `StartInterval` to 3600, then each script checks a stamp so the slow command skips most ticks. `brew-upgrade` runs Homebrew `update`, `upgrade`, cask `--greedy`, and `cleanup` at most once every 24 hours. The stamp file is `~/.local/state/brew-upgrade/last_success`. A failed run leaves the stamp in place, so the next tick retries, and that step calls `alert_failure`. `nvim-update` refreshes LazyVim plugins through lazy.nvim and refreshes treesitter parsers at most once every 4 days, with no `user_present` check. On failure, `nvim-update` calls `banner`.

I use `StartInterval` 3600 because the laptop sleeps. On wake, launchd runs the agent once the stamp is old enough. `qmd-update` remains at `launchd/qmd-update/` and is not listed, so the personal Mac does not run it.

## Work DF_LAUNCHD adds the other agents

`profiles/work/profile.conf` keeps those two names and adds many more. Upkeep, doctor checks, a pull request report, cleanup, and login recovery are the groups.

`caffeine-check` uses `StartInterval` 7200, loads only in an `Aqua` session, and skips with no later rerun when `user_present` fails. Hammerspoon must be open for the `caffeinate` dialog. `firefox-sync` ticks hourly and copies Firefox Sidebery settings only when the stamp is at least 3 days old, on `main`, with clean target files. A rebase or a merge skips the copy. `firefox-sync` does not push.

`claude-doctor` runs on Mondays at 09:30 and writes the HTML report with `render_html`. The dialog can close, copy a command, or show the report. `claude-settings-drift` and `pi-doctor` use `StartInterval` 10800 and skip when I am away. `skill-upstream` runs daily at 10:30 and compares skill SHAs with upstream. `pr-status` runs hourly, calls `user_present`, and lists pull requests that need me or my review in an HTML report. While the queue is not empty, later ticks post the banner again, and the dialog opens that report.

Hourly `janitor` writes a cleanup plan after Sunday at 10:00 when I am present and this week has no stamp. The `janitor` tick does not delete. A delete runs only in a separate approval session. `welcome-back` uses `StartInterval` 60. A lock, 30 minutes idle, or a missed tick in sleep counts as away. `welcome-back` shows banners for VPN, SSO, and vendor logins on the first present tick. I complete those logins.

`where-we-stand` runs hourly from 10:00 through 17:00, Sunday through Thursday, in the machine time zone. A failed `user_present` does not queue that hour. `where-we-stand-server` is the sockets agent for the report. launchd listens on localhost and starts the Python process on the first connection. The process exits after about 30 minutes idle.

## Some agents exit when user_present fails

`launchd/lib.sh` defines `user_present`. `nvim-update` does not call it.

```bash
# True when the screen is unlocked and there was input within IDLE_MAX_MIN.
user_present() {
  if [[ -n "${PRESENT:-}" ]]; then [[ "$PRESENT" == 1 ]]; return; fi
  ioreg -n Root -d1 -a 2>/dev/null | grep -A1 CGSSessionScreenIsLocked | grep -q '<true/>' && return 1
  local idle_ns
  idle_ns="$(ioreg -c IOHIDSystem | awk '/HIDIdleTime/{print $NF; exit}')"
  (( idle_ns / 1000000000 < ${IDLE_MAX_MIN:-10} * 60 ))
}
```

`IDLE_MAX_MIN` defaults to 10 minutes. `PRESENT=1` passes the check. `claude-doctor` and `skill-upstream-new` keep the lock and wait. Four hours is the longest `claude-doctor` waits, so the Monday run still starts after I unlock the screen. `lock_acquire` does not take a lock whose owner process is still alive. The owner stays alive while the dialog is open.

`render_html` runs `bin/quick-html` and writes an HTML report with no remote content under `~/.local/state/<name>/`. `banner` shows a notification. `banner_click` opens an https URL or a file under `~/.local/state/` when `pgrep` finds Hammerspoon, and otherwise calls `banner`. `alert_failure` is an OK dialog with no timeout. `brew-upgrade` calls `alert_failure` while it still holds the lock.

Personal `DF_LAUNCHD` omits `claude-doctor`, `pr-status`, and `welcome-back`. `install.sh` loads an agent only when `DF_LAUNCHD` names it.
