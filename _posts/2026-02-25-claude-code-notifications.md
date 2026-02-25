---
layout: post
title: "Perfect Claude Code Notifications Setup with Tailscale and ntfy"
date: 2026-02-25
description: "Get phone notifications when Claude Code needs your input, using a self-hosted ntfy server over Tailscale."
tags: [claude-code, tailscale, ntfy, docker]
comments: true
---

If you're like me and have been hooked [into running Claude Code on your phone](https://petesena.medium.com/how-to-run-claude-code-from-your-iphone-using-tailscale-termius-and-tmux-2e16d0e5f68b), running several [sessions in parallel like Boris](https://x.com/bcherny/status/2007179833990885678), you may have noticed that it is easy to lose track of what is going on on all those sessions. You may go away for a sec, distracted by [Minecraft parkour videos][minecraft-parkour] and forget that Claude is waiting for your input.

## Idea

[Claude Code][claude-code] comes with a [notification hook][notification-hook]. Some terminals support it natively ([iTerm2][iterm2], [Kitty][kitty], [Ghostty][ghostty]) but most don't, and even when they do, it's a system notification which is easy to miss if you step away.

The idea is to get a phone notification when Claude Code needs your input. I considered a few options, and I ended up choosing [ntfy][ntfy] as the notification provider.

To make sure that everything stays private, I decided to host ntfy on my machine and use [Tailscale][tailscale] as my private network.

I was also tired of dealing with bash scripts. I kept running into compatibility issues between Mac, Linux and Windows, so I built a small tool to solve that (but you can still use bash).

## Requirements

The only thing you need is a Tailscale account and [Docker][docker] for that. If you want to go with bash, it helps to have [`jq`][jq] installed.

## Step 0: Project Structure

Here are the files you'll need:

```text
my-infra/
├── .env
├── compose.yml
└── config/
    └── ntfy.json
```

## Step 1: Configure [Tailscale ACL][acl-tags]

Go to the [ACL editor][acl-editor] and add a `tag:container` tag:

```json
"tagOwners": {
  "tag:container": ["autogroup:admin"]
}
```

## Step 2: Create an [OAuth Credential][oauth-clients]

Go to [Trust & Credentials][ts-credentials] to generate a new OAuth credential.

1. Click **Credential** → **OAuth**
2. Grant `auth_keys` scope with **write** permission
3. Select tag `tag:container`
4. Copy the client secret (`tskey-client-...`)

OAuth works better because the regular auth keys expire in 1–90 days. OAuth client credentials don't expire and the container re-authenticates automatically on restart.

Now add the OAuth key to your `.env`:

```sh
TS_AUTHKEY=...
```

## Step 3: Docker Compose

Your compose will look like below. It uses the [`tailscale/tailscale`][ts-image] and [`binwiederhier/ntfy`][ntfy-image] images and relies on [Tailscale sidecar pattern][ts-docker] where it **exposes your Docker containers as machines** in the tailnet. This is really useful because you can reach the Docker container by name directly, the sidecar will proxy the request, handle HTTPS, etc.

```yaml
name: my-infra

services:
  ts-ntfy:
    image: tailscale/tailscale:latest
    container_name: ts-ntfy
    hostname: ntfy
    restart: unless-stopped
    environment:
      - TS_AUTHKEY=${TS_AUTHKEY}?ephemeral=false
      - TS_EXTRA_ARGS=--advertise-tags=tag:container --reset
      - TS_SERVE_CONFIG=/config/ntfy.json
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_USERSPACE=false
    volumes:
      - ts-ntfy-state:/var/lib/tailscale
      - ./config:/config
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - net_admin

  ntfy:
    image: binwiederhier/ntfy
    container_name: ntfy
    restart: unless-stopped
    command: serve
    environment:
      NTFY_BASE_URL: "https://ntfy.<your-tailnet>.ts.net"
      NTFY_UPSTREAM_BASE_URL: "https://ntfy.sh"
    network_mode: service:ts-ntfy
    depends_on:
      - ts-ntfy

volumes:
  ts-ntfy-state:
```

Note the [**`NTFY_UPSTREAM_BASE_URL`**][ntfy-upstream] setting. This forwards push notifications through ntfy.sh's Firebase/APNs infrastructure for instant mobile delivery. Without it, notifications can be delayed by minutes or hours.

## Step 4: [Tailscale Serve][ts-serve] Config

`config/ntfy.json` — this tells Tailscale to proxy HTTPS to ntfy's port 80:

```json
{
  "TCP": {
    "443": {
      "HTTPS": true
    }
  },
  "Web": {
    "${TS_CERT_DOMAIN}:443": {
      "Handlers": {
        "/": {
          "Proxy": "http://127.0.0.1:80"
        }
      }
    }
  }
}
```

## Step 5: Start It

```bash
docker compose up -d
```

Give it ~15 seconds for the TLS certificate to be provisioned. ntfy is now available at `https://ntfy.<your-tailnet>.ts.net` from any device on your tailnet.

**Tip:** Your tailnet name (the `taila2944f` part) can be changed to something more readable in [DNS settings][ts-dns]. Also make sure that "HTTPS Certificates" are enabled.

## Step 6: Subscribe on Your Phone

You need to install the ntfy app, available on [Google Play][ntfy-android] and the [App Store][ntfy-ios]. Once installed you need to subscribe to a topic with your server URL. For example:

1. Add `claude-code` as the topic
1. Choose the custom server: `https://ntfy.<your-tailnet>.ts.net`

You can make a quick test with:

```bash
curl -s -H "Title: Test" -d "Hello from the terminal!" "https://ntfy.<your-tailnet>.ts.net/claude-code"
```

## Step 7: Claude Code Hook

Now wire up Claude Code to send notifications through ntfy. You have a few options:

### Option 1: claude-notifier

This is the tool I built to solve that: [claude-notifier](https://github.com/felipeelias/claude-notifier). It handles multiple notification channels, sending to ntfy but also to native system notifications (in Mac, via [`terminal-notifier`][terminal-notifier]).

Install it:

```bash
brew install felipeelias/tap/claude-notifier
```

Generate the config:

```bash
claude-notifier init
```

This creates `~/.config/claude-notifier/config.toml`. Point it to your ntfy server:

```toml
[[notifiers.ntfy]]
url = "https://ntfy.<your-tailnet>.ts.net/claude-code"
title = "Claude Code ({% raw %}{{.Project}}{% endraw %})"
```

Then add the hook to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "claude-notifier"
          }
        ]
      }
    ]
  }
}
```

Same binary and config on every machine. Run `claude-notifier test` to verify it works.

### Option 2: Bash script

You can still go with a bash script if you want. Create `~/.claude/hooks/notify.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Convert backslashes for Windows path compatibility
INPUT=$(cat | tr '\\' '/')
PROJECT=$(printf '%s' "$INPUT" | jq -r '.cwd // empty' | xargs basename 2>/dev/null || echo "")
HOOK_TITLE=$(printf '%s' "$INPUT" | jq -r '.title // empty')
MESSAGE=$(printf '%s' "$INPUT" | jq -r '.message // "Done"')

if [ -n "$HOOK_TITLE" ]; then
  TITLE="$HOOK_TITLE"
elif [ -n "$PROJECT" ]; then
  TITLE="Claude Code ($PROJECT)"
else
  TITLE="Claude Code"
fi

curl -s \
  -H "Title: $TITLE" \
  -d "$MESSAGE" \
  "${NTFY_URL}/claude-code" > /dev/null 2>&1 || true
```

Make it executable (`chmod +x ~/.claude/hooks/notify.sh`) and add to `~/.claude/settings.json`:

```json
{
  "env": {
    "NTFY_URL": "https://ntfy.<your-tailnet>.ts.net"
  },
  "hooks": {
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/notify.sh"
          }
        ]
      }
    ]
  }
}
```

This requires `jq` (`brew install jq`, `apt install jq`, or `winget install jqlang.jq`).

## Putting all together

If all is working you should see this:

![ntfy notification on phone](/assets/images/ntfy.webp)

## Troubleshooting

If the notification says "New message", make sure that all devices (including your phone) are on the same Tailscale network. If they are and you're still not getting notifications, you can always ask Claude to help you debug it.

[acl-editor]: https://login.tailscale.com/admin/acls
[acl-tags]: https://tailscale.com/kb/1068/acl-tags
[claude-code]: https://code.claude.com/docs
[docker]: https://www.docker.com
[ghostty]: https://ghostty.org
[iterm2]: https://iterm2.com
[jq]: https://jqlang.org/
[kitty]: https://sw.kovidgoyal.net/kitty/
[minecraft-parkour]: https://www.youtube.com/watch?v=OqPxaKs8xrk
[notification-hook]: https://code.claude.com/docs/en/hooks
[ntfy]: https://ntfy.sh
[ntfy-android]: https://play.google.com/store/apps/details?id=io.heckel.ntfy
[ntfy-image]: https://hub.docker.com/r/binwiederhier/ntfy
[ntfy-ios]: https://apps.apple.com/us/app/ntfy/id1625396347
[ntfy-upstream]: https://docs.ntfy.sh/config/#upstream-ntfysh
[oauth-clients]: https://tailscale.com/kb/1215/oauth-clients
[tailscale]: https://tailscale.com
[terminal-notifier]: https://github.com/julienXX/terminal-notifier
[ts-credentials]: https://login.tailscale.com/admin/settings/trust-credentials
[ts-dns]: https://login.tailscale.com/admin/dns
[ts-docker]: https://tailscale.com/docs/features/containers/docker
[ts-image]: https://hub.docker.com/r/tailscale/tailscale
[ts-serve]: https://tailscale.com/kb/1312/serve
