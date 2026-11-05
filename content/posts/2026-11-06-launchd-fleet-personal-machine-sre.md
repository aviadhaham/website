---
title: "A Launchd Fleet as Personal Machine SRE"
tags: [macos, launchd, devops, automation]
date: 2026-11-06
slug: launchd-fleet-personal-machine-sre
toc: true
description: "How I select a launchd fleet per profile: two agents on the personal Mac, a much larger work fleet, and a shared contract of ticks, presence gates, and HTML reports."
---

My work Mac runs a fleet of tiny launchd agents. The personal Mac runs two. Same clone, same installer. The only switch is `DF_LAUNCHD` in the profile.

## From `modules/launchd-*` to `launchd/`

Each job used to be a module, `modules/launchd-<name>/`, enabled with `DF_MODULES`. Nothing in that module was for the shell to source. It existed so a profile could turn the job on, and every one of them shipped the same `hooks/postinstall` calling `launchd_install`.

I collapsed that. Jobs live under `launchd/`, and the profile lists names in `DF_LAUNCHD`, the same kind of array as `DF_MACOS_DEFAULTS`. The installer checks that `launchd/<name>/com.user.<name>.plist` and `<name>.sh` exist, then loads only the names in the array. An empty array installs nothing. `--no-launchd` skips the step. `--only` and `--skip` still filter modules, not these agents.

```bash
for name in "${DF_LAUNCHD[@]}"; do
  requires_satisfied "$DOTFILES_DIR/launchd/$name/requires" "launchd:$name" || continue
  launchd_install "$DOTFILES_DIR/launchd/$name/com.user.$name.plist"
done
```

The script stays in the clone and runs in place. `launchd_install` substitutes `__HOME__` and `__DOTFILES__` into a plist under `~/Library/LaunchAgents`, creates `~/.local/state/<name>/` (launchd will not create the log directory), and bootstraps again only when that rendered plist changed or the label is not loaded. A later pass put each job in its own directory, so a TypeScript agent can keep a package and tests next to a thin shell launcher. Removing a name from `DF_LAUNCHD` does not unload a running agent. That still needs `launchctl bootout` and deleting the plist.

## Two lines versus a fleet

Personal is the whole list:

```bash
DF_LAUNCHD=(
  brew-upgrade
  nvim-update
)
```

Both use `StartInterval` of an hour, then a stamp. `brew-upgrade` runs `update`, `upgrade`, a greedy cask pass, and `cleanup` at most once a day. `nvim-update` refreshes plugins and treesitter parsers at most once every four days, and a failure is a banner. A failed brew run leaves the stamp alone, so the next hourly tick retries. I prefer the hourly plist to a daily calendar because this laptop sleeps. After wake, launchd catch-up runs the job once the stamp is old enough, instead of waiting out another full day. `qmd-update` is still in the tree and is not in this list, so it does not run.

The work profile includes those two and keeps going. I think of the rest in groups.

Upkeep is `caffeine-check` (every two hours, GUI session only, and only if I am there) and `firefox-sync`, which pulls Firefox Sidebery settings back into the repo on a multi-day stamp, commits locally, and never pushes. Doctors and drift checks are `claude-doctor` on Mondays at 09:30, `claude-settings-drift`, `pi-doctor`, and `skill-upstream`. `pr-status` rebuilds an HTML report of pull requests that need me, and nags again each hour while that queue is non-empty. `janitor` waits until after Sunday at 10:00, then writes a cleanup plan for listed checkouts. The tick never deletes. `welcome-back` ticks every minute, treats a lock, half an hour idle, or missed ticks as away, and on the first present tick banners VPN, SSO, and vendor logins. I still do the login.

One aside: the standup page is not an always-on process. `where-we-stand-server` is a sockets agent. launchd listens on localhost, starts the server on the first connection, and the process exits after it has been idle.

## Gates, then a page or a dialog

Anything interactive calls `user_present` before it talks to me.

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

Away means skip, and do not queue a catch-up for an empty desk. The idle window defaults to ten minutes. A few weekly jobs wait instead, holding the lock, so the Monday doctor still runs once I am back. The lock will not steal a live holder. A dialog I have not dismissed is not free for the next tick to replace.

A quiet run is a log under `~/.local/state/<name>/`. When there is something to read, `render_html` writes a local page and does not pull in remote content. `banner` is a plain notification. `banner_click` opens an https URL or a file in that state directory through Hammerspoon, and falls back to the plain banner otherwise. `claude-doctor` and `janitor` add a dialog that can copy a command or open the report. `pr-status` opens its report from the dialog. `brew-upgrade` uses `alert_failure` for a failed step: a modal OK with no timeout, called before the lock comes off so the next tick cannot overlap it.

The personal list never includes the doctors, `pr-status`, or `welcome-back`. Those sessions are not on that machine. A new agent is a directory under `launchd/` and a name in the profiles that should run it. The fleet is the list.
