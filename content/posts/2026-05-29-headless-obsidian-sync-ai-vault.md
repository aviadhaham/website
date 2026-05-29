---
title: "Headless Obsidian Sync for an AI Vault"
tags: [obsidian, hermes, docker, ansible, devops]
date: 2026-05-29
slug: headless-obsidian-sync-ai-vault
toc: true
description: "How I wired a headless Obsidian Sync container to the same vault Hermes uses on a VPS, with Ansible, Docker Compose, and strict host ownership."
---

I wanted my agent runtime to write into the same Obsidian vault I use on my laptop, without giving the container direct access to my desktop. The result is a simple pattern:

- a host-owned vault directory on the VPS
- a headless Obsidian Sync container
- the Hermes container mounting the same vault path
- strict ownership and capability limits

That lets Hermes write notes into the same vault ecosystem I use personally, while keeping the agent isolated from the rest of the host.

## The storage model

Hermes mounts the host tree directly into the container:

```yaml
# templates/stacks/hermes/compose.yml
volumes:
  - ${HERMES_HOME_DIR}:/opt/data
```

On the host, that tree includes the vault directory:

```text
/home/hermes/vaults  ->  /opt/data/vaults
```

So anything Hermes writes under `/opt/data/vaults` lands in the host vault directory.

## The sync bridge

The actual sync layer is a headless Obsidian Sync container:

```yaml
# templates/stacks/obsidian-sync/compose.yml
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

This container is the bridge to my Obsidian Sync peers. It mounts the same vault path that Hermes uses, so edits propagate both ways:

- laptop changes sync to the VPS
- Hermes-written notes sync back to the laptop

## Why Ansible matters

Before the containers start, Ansible pre-creates the host directories:

```yaml
# tasks/shared.yml
- name: Ensure hermes vaults directory
  ansible.builtin.file:
    path: "/home/{{ hermes_user }}/vaults"
    state: directory
    owner: "{{ hermes_user }}"
    group: "{{ hermes_user }}"
    mode: "0750"
```

and the Obsidian Sync config dir:

```yaml
- name: Ensure obsidian-sync config directory
  ansible.builtin.file:
    path: "/home/{{ hermes_user }}/obsidian-sync-config"
    state: directory
    owner: "{{ hermes_user }}"
    group: "{{ hermes_user }}"
    mode: "0700"
```

That avoids the common Docker-first-start problem where a root-owned directory breaks later writes.

## The security boundary

The important part is that the agent is *not* running as my admin user on the host.

- `hermes` is an unprivileged host user
- vault and config dirs are owned by `hermes`
- the sync container has `no-new-privileges`
- capabilities are trimmed aggressively
- the mount is local to the host, not exposed over a public port

So the note flow is convenient, but the trust boundary stays intact.

## The core idea

In practice, the architecture is:

```text
laptop Obsidian
   ↕ Obsidian Sync
headless sync container on VPS
   ↕ /home/hermes/vaults
Hermes container
```

The useful trick is that the AI agent and the human editor are not separate silos. They write into the same vault, but through a controlled host-side bridge.

If you want the shortest possible summary:

> Hermes writes to a host-mounted vault, Obsidian Sync keeps that vault aligned with my laptop, and Ansible makes the whole thing reproducible.

That is the pattern I’d recommend if you want an agent to operate inside your notes without turning the vault into an unsafe shared filesystem.
