# tgclone

A self-hosted messenger that uses the MyTelegram server (open edition) with a client built from Telegram Web A.

```
client/  mytelegram-weba (fork of Ajaxy/telegram-tt, GPL-3.0)  → http://localhost:1234
server/  MyTelegram docker-compose (Apache-2.0)                → ws://<host>:30444 (Web), tcp :20443/:20543
re/      Reverse-engineering notes on Telegram Air
```

## Run locally

1. Install a Docker runtime and open it once. This Mac uses OrbStack; its `docker` command lives in `/Applications/OrbStack.app/Contents/MacOS/xbin`.
2. Build the two patched messenger images (first time only; see "Server patches" below):
   ```bash
   cd server/mytelegram-src/source
   docker build -f ../build/docker/Dockerfile-mytelegram-messenger-query-server -t tgclone/mytelegram-messenger-query-server:layer232 .
   docker build -f ../build/docker/Dockerfile-mytelegram-messenger-command-server -t tgclone/mytelegram-messenger-command-server:layer232 .
   ```
3. Create `server/.env` from the upstream template. Replace `192.168.1.100` with this machine's LAN IP, and regenerate the secrets before any public use:
   ```bash
   cd server && cp .env.upstream .env
   ```
   Then start the server. `docker-compose.override.yml` swaps in the patched images:
   ```bash
   cd server && mkdir -p data/mytelegram && chmod -R a+w data/mytelegram && docker compose up -d
   ```
4. Start the client:
   ```bash
   cd client && npm run dev
   ```
5. Open http://localhost:1234. Log in with any valid-format phone number, and use the code `22222`. To test a second account in the same browser, open http://127.0.0.1:1234: it is a different origin, so it keeps a separate session.

## Desktop app (macOS)

The desktop app is a Tauri wrapper in `client/tauri`. It opens a native window on the web client at `BASE_URL`, which defaults to `http://localhost:1234`, so the dev server (`npm run dev`) must be running. The wrapper adds the Dock badge, notifications and a tray icon.

```bash
cd client && npx tauri build --bundles app
```

The output is `client/tauri/target/release/bundle/macos/TruckConnect.app`. The build is unsigned: on another Mac, open it the first time with right-click → Open. To point a build at a hosted client, set `BASE_URL=https://…` when running `tauri build`.

## Troubleshooting

- **Logins hang, or the server logs `Message ID is too old/new`.** The Docker VM clock has drifted, usually after the Mac slept. The protocol rejects messages with skewed timestamps. Restart OrbStack to resync the clock: `/Applications/OrbStack.app/Contents/MacOS/bin/orbctl stop && …/orbctl start`. The containers come back on their own.

## Branding

- **Name and colour.** The client is branded TruckConnect, in orange (#F97316). Code-level name references use `APP_BRAND_NAME` in `client/src/config.ts`. The interface text lives in `client/src/assets/localization/fallback.strings`, which the client uses because the server serves no language pack.
- **Logo sources.** The logos are SVGs in `client/src/assets/truckconnect-logo*.svg`. Every PNG, `.ico` and `.icns` in `client/public` and `client/tauri/icons` is rendered from them.

## Server patches

The published open-source images (v0.45.232.1030) do not work with the layer-232 Web A client as shipped. `server/mytelegram-src` holds upstream `dev` (commit `b9bd5ba`) plus two local commits:

1. **Remove the `LayerN` handlers.** These handlers convert requests from older API layers, but the converters they need ship only in the Pro edition. In the published query-server image they crash startup, and the published command server rejects `auth.sendCode`. Without them, the server supports only the current layer 232.
2. **Add `TMessageEntityTonAddress`.** Upstream registers this type but never added its class, so every `messages.sendMessage` failed with a JSON serialization error.

Other notes:

- The client starts on phone-number login. The server does not implement QR login (`auth.exportLoginToken`) or passkeys; those calls only log errors.
- The `session-server` and `file-server` images still come from upstream. Their source is not in the open edition.
- MinIO reports `unhealthy` because its healthcheck calls `curl`, which that image does not include. MinIO itself is running.

## Tested (2026-11-18)

- Two accounts can sign up with code `22222`.
- Users can find each other by name.
- Private chat works in both directions, with live delivery and read receipts.
- Groups work: create, add a member, and send messages. A user added to a new group sees it only after a reload; after that, messages arrive live.
- Channels work: create, add a subscriber, and post.

## Configuration

- `client/.env`
  - `SERVER_HOST` is the host the web client connects to, at `ws://SERVER_HOST:30444`.
  - `TELEGRAM_API_ID` and `TELEGRAM_API_HASH` must be set, but they are placeholders.
- `server/.env`
  - `App__DcOptions__*__IpAddress` holds the addresses the server gives to native clients. It is currently set to this Mac's LAN IP.
  - `server/.env.upstream` is the original file.

## Before going public

- The secrets in `server/.env` are copied from the upstream repo, so they are publicly known. Regenerate them:
  - the RabbitMQ and MinIO passwords
  - `AccessHashSecretKey`
  - the `EncryptionConfig` keys
- Remove `App__FixedVerifyCode`, and configure an SMS or email sender.
- Serve the client over HTTPS and the server over WSS (port 30443 with certificates).
- Rebrand the app: do not use the Telegram name or logo. Keep the client's source public, because it is GPL-3.0.
