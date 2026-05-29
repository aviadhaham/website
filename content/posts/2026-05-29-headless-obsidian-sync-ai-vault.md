---
title: "Headless Obsidian Sync for an AI Vault"
tags: [obsidian, hermes, docker, ansible, devops]
date: 2026-05-29
slug: headless-obsidian-sync-ai-vault
toc: true
description: "How I wired a headless Obsidian Sync container to the same vault Hermes uses on a VPS, with Ansible, Docker Compose, and strict host ownership."
---

I wanted my agent to write into the same Obsidian vault I use on my laptop, without giving the container direct access to my desktop. The setup ended up being pretty simple:

- a host-owned vault directory on the VPS
- a headless Obsidian Sync container
- the Hermes container mounting that same vault path
- a couple of basic permission guards

That keeps Hermes inside the same note-taking flow I already use, while still keeping the host boundary clean.

## The storage model

Hermes mounts the host tree directly into the container:

```yaml
volumes:
  - ${HERMES_HOME_DIR}:/opt/data
```

The vault lives under that tree, so anything Hermes writes to `/opt/data/vaults` lands in the shared vault directory.

## The sync bridge

The actual sync layer is a headless Obsidian Sync container:

```yaml
services:
  obsidian-sync:
    image: ghcr.io/belphemur/obsidian-headless-sync-docker:0.0.8@sha256:...
    restart: unless-stopped
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETUID
      - SETGID
      - DAC_OVERRIDE
    security_opt:
      - no-new-privileges:true
    environment:
      OBSIDIAN_AUTH_TOKEN: ${OBSIDIAN_AUTH_TOKEN}
      VAULT_NAME: ${VAULT_NAME}
      VAULT_PASSWORD: ${VAULT_PASSWORD:-}
      PUID: ${PUID:-1000}
      PGID: ${PGID:-1000}
      VAULT_PATH: ${VAULT_PATH:-/vault}
      DEVICE_NAME: ${DEVICE_NAME:-obsidian-docker}
      CONFLICT_STRATEGY: ${CONFLICT_STRATEGY:-merge}
    volumes:
      - /home/hermes/vaults:/vault
      - ${CONFIG_HOST_PATH:-./config}:/home/obsidian/.config
```

This is the bridge to my Obsidian Sync peers. It mounts the same vault path Hermes uses, so edits move both ways:

- laptop changes sync to the VPS
- Hermes-written notes sync back to the laptop

## A small but important detail

Before the containers start, the host directories are created with the right ownership. That avoids the usual first-run Docker problem where a root-owned directory breaks later writes.

## The security boundary

The important part is that the agent is *not* running as my admin user on the host.

- `hermes` is an unprivileged host user
- vault and config dirs are owned by `hermes`
- the sync container has `no-new-privileges`
- capabilities are trimmed aggressively
- the mount is local to the host, not exposed over a public port

So the note flow stays convenient without making the vault a loose shared filesystem.

## The core idea

In practice, the architecture is:

```text
laptop Obsidian
   ↕ Obsidian Sync
headless sync container on VPS
   ↕ vault directory
Hermes container
```

The point is simple: the agent and the human editor write into the same vault, but through a controlled host-side bridge.

Short version:

> Hermes writes to a host-mounted vault, Obsidian Sync keeps that vault aligned with my laptop, and the host permissions keep it sane.

That’s the pattern I’d recommend if you want an agent to work inside your notes without turning the vault into an unsafe shared filesystem.
