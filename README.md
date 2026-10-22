# Phone Drop

Send photos, files, and text from any phone's browser straight into an inbox folder on your Mac — over your trusted LAN or Tailscale, with nothing to install on the phone.

> **Status:** stable · low-frequency maintenance · last verified 2026-10 (v0.3.0). Renamed from `iphone-sensor-bridge-poc` in October 2026; old GitHub links redirect here.
>
> **The phone page is in Chinese** (Simplified). There is no English localization yet.

[简体中文](README_CN.md)

<img src="docs/assets/phone-page.png" width="300" alt="The phone page after connecting and sending a short note: the note lands in the Mac inbox as a .txt file, and the page offers a ready-to-paste instruction pointing an AI assistant at that inbox.">

## Three steps

1. **Make a token** (once): `openssl rand -hex 32 > ~/.config/phone-drop/token` inside a mode-`700` directory, file mode `600` (full commands under [LAN or Tailscale run](#lan-or-tailscale-run)).
2. **Start it on your Mac's Tailscale or LAN address**: `HOSTS=<that address> BRIDGE_TOKEN_FILE=~/.config/phone-drop/token npm start`, or install it as a LaunchAgent with `BRIDGE_LISTEN_HOSTS=<that address> npm run launchd:install`.
3. **Open `http://<that address>:8765` on the phone**, paste the token once, then take a photo, pick files, or send a block of text. Everything lands in `~/Desktop/phone-drop-inbox`.

After each upload the page shows a short note (the inbox path plus the new file names) that you can copy and paste into an AI assistant running on the Mac, such as Claude Code, so it picks the files up.

## How it compares

| | Works from | Needs an app on the phone | Mac can be elsewhere | Sends plain text |
|---|---|---|---|---|
| AirDrop | iPhone / Apple devices only | No (built in) | No, must be nearby | Yes |
| [Taildrop](https://tailscale.com/kb/1106/taildrop) | Any device in your tailnet | Tailscale app | Yes | No, files only |
| [LocalSend](https://localsend.org) | Most platforms | LocalSend app | No, same network | Yes |
| **Phone Drop** | Any phone with a browser | No (browser only; Tailscale app only if the Mac is remote) | Yes, over Tailscale | Yes |

Pick Phone Drop when the phone is not an Apple device or not yours to install apps on, when you want every drop to land in one fixed inbox folder on the Mac, or when you want to hand those files straight to an AI assistant. Pick one of the others if you want end-to-end encryption on a plain LAN (Phone Drop over LAN is plain HTTP) or transfers in both directions.

## Security model

- `GET /` is an unauthenticated application shell. It contains neither the server token nor protected inbox data.
- `GET /api/health` is public and returns only service status and version.
- All other `/api/*` routes require `Authorization: Bearer <token>` when a token is configured.
- The former `x-bridge-token` header is no longer accepted.
- If any listen address is not loopback, startup is refused unless a Bearer-safe token of at least 32 characters is supplied through `BRIDGE_TOKEN` or `BRIDGE_TOKEN_FILE`. There is no tracked or predictable default token.
- The browser asks the user for the token and keeps it in the page's `localStorage`, so it persists across tabs and restarts until cleared with the page's "Forget Token" button. A link with `?token=...` appended can be opened directly — the value is saved immediately and then stripped from the address bar. The server's access log records only the request path, never the query string.
- `X-Forwarded-For` is ignored by default. Set `TRUST_PROXY=true` only behind a trusted reverse proxy that either overwrites the header with its verified client address or appends the directly observed client address. The rate limiter uses the rightmost valid address, so standard append behavior cannot be bypassed with a spoofed leftmost value. A proxy that passes the client header through unchanged is unsafe and must not use this option.

Tailscale and the Bearer token cover different boundaries: Tailscale controls which devices can reach the HTTP service, while the token controls which requests can read inbox metadata or deliver content. Do not expose this server directly to the public internet. A direct LAN connection is plain HTTP, so use only a trusted LAN or a Tailscale route.

## Requirements

- macOS for the LaunchAgent installer; the server itself is plain Node.js
- Node.js 18 or newer
- No external npm dependencies

## Local loopback run

Loopback-only mode may run without a token:

```sh
npm start
```

The defaults are `127.0.0.1:8765` and `~/Desktop/phone-drop-inbox`.

## LAN or Tailscale run

Generate a high-entropy token locally and store it outside Git in a private file:

```sh
mkdir -p "$HOME/.config/phone-drop"
chmod 700 "$HOME/.config/phone-drop"
umask 077
openssl rand -hex 32 > "$HOME/.config/phone-drop/token"
chmod 600 "$HOME/.config/phone-drop/token"

cp .env.example .env
chmod 600 .env
```

Edit `.env`, set `HOSTS` to the required non-loopback address, and set `BRIDGE_TOKEN_FILE=$HOME/.config/phone-drop/token`. Prefer the Mac's exact Tailscale address (`tailscale ip -4`); use `0.0.0.0` only when you intentionally want every network interface to reach the service. Load the local environment before starting:

```sh
set -a
. ./.env
set +a
npm start
```

Open the LAN or Tailscale URL on the phone, enter the same token, and connect — or open a link with `?token=...` appended to skip typing it in. The token is kept in `localStorage` and survives closing the tab; use the page's 「忘记 Token」 (Forget Token) button to clear it.

## LaunchAgent installation

The tracked plist and installed LaunchAgent are both secret-free. The installer asks for the token with hidden input, stores it in `~/.config/phone-drop/token` under a mode-`700` directory with file mode `600`, and restarts the LaunchAgent `local.phone-drop`. The server refuses symlinks, wrong ownership, weak permissions, and malformed values:

```sh
BRIDGE_LISTEN_HOSTS=<your Tailscale or LAN address> npm run launchd:install
```

`BRIDGE_LISTEN_HOSTS` is required; the installer stops with a hint if it is missing, so the service never listens on every interface by accident. Pass `0.0.0.0` explicitly if that is what you want. For non-interactive use, pass `BRIDGE_TOKEN` from a local secret store in the process environment. Optionally set `BRIDGE_INBOX` or `BRIDGE_NODE_BIN` to override the inbox or the Node executable; path overrides must be absolute. The tracked plist is a portable template; the installer renders all home, repository, executable, and log paths for the current machine. It never prints the token or places it in a process argument.

## Configuration

- `HOSTS`: comma-separated listen addresses; defaults to `127.0.0.1`.
- `PORT`: defaults to `8765`.
- `INBOX`: destination directory; defaults to `~/Desktop/phone-drop-inbox`.
- `BRIDGE_TOKEN`: direct token input; at least 32 characters when any `HOSTS` entry is non-loopback.
- `BRIDGE_TOKEN_FILE`: preferred launchd token source; must be a user-owned regular file with mode `600`.
- `MAX_BODY`: upload limit in bytes; defaults to 200 MiB.
- `TRUST_PROXY`: defaults to false; enable only when direct access is restricted to a trusted proxy that overwrites the forwarded address or appends its directly observed client address.

## Upgrading from iphone-sensor-bridge-poc

v0.3.0 renamed the defaults. An existing install keeps running untouched until you reinstall. To keep your old inbox and token, reinstall with:

```sh
BRIDGE_LISTEN_HOSTS=<address> \
BRIDGE_INBOX="$HOME/Desktop/iphone-sensor-inbox-v2" \
BRIDGE_TOKEN_FILE="$HOME/.config/iphone-sensor-bridge-poc/token" \
npm run launchd:install
```

then remove the old LaunchAgent: `launchctl bootout gui/$(id -u)/com.alice.iphone-sensor-bridge-poc` and move `~/Library/LaunchAgents/com.alice.iphone-sensor-bridge-poc.plist` to the Trash. If you cloned the old repository, update its remote with `git remote set-url origin git@github.com:AliceLJY/phone-drop.git`.

## Verification

```sh
npm run check
npm test
curl -s http://127.0.0.1:8765/api/health
```

Protected API example:

```sh
curl -H 'Authorization: Bearer <token-from-local-secret-store>' \
  http://127.0.0.1:8765/api/items
```

CI runs the syntax and behavior tests on Node.js 18, 20, and 22.

When multiple `HOSTS` values are configured, startup succeeds only after every address is listening. A failure on any address closes listeners that already opened and exits nonzero so launchd can restart the service instead of leaving a partial service running.
