---
title: "A Launchd Fleet as Personal Machine SRE"
tags: [macos, launchd, devops, automation]
date: 2026-11-06
slug: launchd-fleet-personal-machine-sre
toc: true
description: "How I run a small fleet of launchd agents on my Mac, with upgrades, doctor checks, a PR digest, and login recovery selected per profile."
---

My work Mac has a fleet of tiny launchd agents. One upgrades Homebrew. One writes a doctor report. One leaves a pull-request digest. One asks me to sign back in when a session dies. I do not run all of them on every machine. A profile picks the set, the same way it picks shell modules.

## `launchd/` and `DF_LAUNCHD`

These jobs used to live as `modules/launchd-*`. That was the wrong shelf. A module is something the shell sources. A launchd agent is a plist the installer copies into place and bootstraps with `launchctl`. Keeping them next to shell snippets made the two lists drift, and a personal machine could load a work-only job just because the file was sitting there.

They now live under `launchd/`. The profile names the ones it wants with `DF_LAUNCHD`, and the installer loads that list and ignores the rest.

```bash
# work profile, trimmed to the idea
DF_LAUNCHD=(
  brew-upgrade
  doctor
  pr-digest
  session-recovery
)
```

A personal profile keeps a shorter list. Same tree, different fleet.

## The contract

Every agent follows the same three rules, so a new job does not invent a new operational style.

**Hourly ticks.** launchd starts the script on a `StartInterval` of an hour. The script does its work and exits. Nothing sits in a loop waiting for the world to change. If a run fails, the next tick is the retry.

**Presence gates.** The first thing the script does is decide whether this machine, this user, and this hour are a real time to act. Session locked, a required tool missing, a clean run already recorded today: exit quietly. A closed lid is a successful no-op. The gate is what keeps an hourly job from turning into hourly noise.

**HTML reports and clickable banners.** When there is something to say, the script writes a small HTML file and posts a notification that opens it. The banner is the pointer. The page is the record: what ran, what changed, and what I am supposed to do. A one-line notification never carried the whole story, and a log file I have to go find does not get read.

```bash
# the shape, not a specific agent
if ! presence_ok; then
  exit 0
fi

result="$(do_the_job)" || rc=$?
report="$(write_html_report "$result")"
if [[ -n "$result" ]]; then
  notify_open "$report"
fi
exit "${rc:-0}"
```

## Three patterns

**Upgrade stamp.** The brew agent upgrades what is safe to upgrade and writes down what moved. The page is a stamp: when it ran, which formula, the old version, the new version. If nothing changed, there is no banner. I can answer "what updated this week?" without reconstructing it from Homebrew's own output.

**Doctor.** This one mostly looks. It checks agent exit codes, a stale lock, free disk, and whether the last upgrade stamp is older than it should be. A passing run stays quiet. A failed check gets a banner, and the report already has that check expanded. One page beats bisecting a handful of plist logs.

**Reconnect.** Some sessions die in the background. SSO expires. A vendor login wants a person at the keyboard. The agent does not store a password and does not click through the prompt. It notices the session is gone, opens a dialog, and stops. I handle it when I am back. The next hourly tick sees a live session and stays quiet. The dialog is the whole interface.

## Why the profile matters

The useful part is the list.

The work profile carries the digest and the reconnect agents, because that machine is where review and vendor sessions actually happen. The personal profile keeps upgrades and a lighter doctor. It should not grow a dialog for a login it will never hold.

Adding a job means adding a directory under `launchd/` and naming it from the profiles that should run it. Installing the dotfiles is not the same thing as enlisting every agent I have ever written.

Small jobs, one contract, and a profile that decides who is on call. That is the whole fleet.
