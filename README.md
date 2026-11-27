# Paradox FX Mini App

MVP Telegram Mini App interface for Paradox FX. The first stage includes only
the Mini App flow and UI: format selection in Telegram, market selection,
language selection, and a placeholder signal workspace. Broker connections,
trading servers, parsers, and real signal generation are out of scope for this
stage.

## Local Docker Run

```bash
docker compose up --build
```

Services:

```text
frontend  http://localhost:5173
backend   http://localhost:8000
postgres  localhost:5432
```

Health check:

```text
http://localhost:8000/health/
```

## Environment

Backend:

```text
DATABASE_URL=postgresql+psycopg://trader:trader@postgres:5432/trader_mini
CORS_ORIGINS=http://localhost:5173,http://127.0.0.1:5173
TELEGRAM_BOT_TOKEN=<token from BotFather>
TELEGRAM_WEBAPP_URL=http://localhost:5173
TELEGRAM_FUNNEL_TEST_ACCESS_CODE=<private bypass code word>
DEVSBITE_CLIENT_TOKEN=<client token>
DEVSBITE_API_BASE_URL=https://api.devsbite.com
```

Frontend build argument:

```text
VITE_API_BASE_URL=http://localhost:8000
```

For Telegram production, both frontend and backend must use public HTTPS URLs.

## Render Deploy

Push the branch containing `render.yaml` once to `Telegram-Mini-App-Paradox`
(remote `paradox` in this working copy). In the Render Dashboard, choose
**New → Blueprint**, select that repository and branch, and review the
three resources before deploying.
[Render Blueprint setup](https://render.com/docs/infrastructure-as-code)

| Resource | Type |
| --- | --- |
| `paradox-backend` | Python Web Service |
| `paradox-frontend` | Static Site (Node.js 24.22.1 for the build) |
| `paradox-db` | PostgreSQL |

### Base deployment

1. Enter `TELEGRAM_BOT_TOKEN` for the Paradox bot and a separate
   `TELEGRAM_WEBHOOK_SECRET` when Render prompts for them, or set both in
   `paradox-backend` → **Environment**. Generate the webhook secret with
   `openssl rand -hex 32` and keep the same value for webhook registration.
   Render supplies `DATABASE_URL` from `paradox-db` and sets
   `ENVIRONMENT=production` through the Blueprint. Keep real credentials in
   Render or an ignored local `.env` file.
2. Copy the actual HTTPS URLs shown by Render and set the following variables
   under **Environment** for each service. Use the base URLs without a
   trailing slash.

   | Service | Variable | Value |
   | --- | --- | --- |
   | `paradox-backend` | `CORS_ORIGINS` | Public HTTPS URL of `paradox-frontend` |
   | `paradox-backend` | `TELEGRAM_WEBAPP_URL` | Public HTTPS URL of `paradox-frontend` |
   | `paradox-frontend` | `VITE_API_BASE_URL` | Public HTTPS URL of `paradox-backend` |

3. Apply the backend variables with **Save and deploy**. For the frontend use
   **Save, rebuild, and deploy**. Vite embeds `VITE_API_BASE_URL` at build time,
   so every change to that variable requires a new frontend build. Confirm the
   frontend loads and the backend's `/health/` returns `{"status":"ok"}`.
   [Render environment variables](https://render.com/docs/configure-environment-variables)

### Register and verify the Telegram webhook

In the terminal, use the Paradox bot token and the same webhook secret you set
on `paradox-backend`. The example uses Bash to read both without echoing them
or putting their values in shell history. Replace the example URL with the
backend's public HTTPS URL. [Telegram setWebhook](https://core.telegram.org/bots/api#setwebhook)

```bash
bash <<'BASH'
set -eu
backend_url='https://your-paradox-backend.onrender.com'
read -rsp 'Paradox bot token: ' bot_token </dev/tty
printf '\n'
read -rsp 'Webhook secret from Render: ' webhook_secret </dev/tty
printf '\n'
curl --fail-with-body --silent --show-error \
  --data-urlencode "url=${backend_url}/api/v1/telegram/webhook" \
  --data-urlencode "secret_token=${webhook_secret}" \
  "https://api.telegram.org/bot${bot_token}/setWebhook"
printf '\n'
BASH
```

Check that Telegram returns `"ok":true`.

### Public access

In Render, open `paradox-backend` → **Environment**, add
`PARADOX_DEMO_MODE=true`, and choose **Save and deploy**. The access setting takes
effect after the backend deploy and requires no frontend rebuild. Use
`PARADOX_DEMO_MODE=false` or remove the variable and redeploy to restore the
standard funnel. The mode is off by default. Existing funnel state is preserved
and scheduled funnel reminders pause while public access is on.

Public access uses the base deployment variables above, including both Telegram
secrets, the three HTTPS URL settings, and the database connection supplied by
the Blueprint. Leave `TELEGRAM_FUNNEL_TEST_ACCESS_CODE` empty. Pocket Option,
Devsbite, and VIP settings are optional and can stay empty.

After deploying, test with a different Telegram account that has never used
the Paradox bot.

1. Find the bot by its exact `@username`, open the private chat, and tap
   **Start**.
2. Tap **Open Paradox** (or **Открыть Paradox**). Confirm the Mini App opens
   directly. Tap **Continue**, generate a Forex signal, and check the TradingView chart.
3. Change the app language, close the Mini App, and reopen it from the bot
   to verify the saved language. Send `/start` again to check that it still
   sends the app button.

With public access off, `/start` sends the normal funnel welcome and opening the
Mini App from that flow requires the existing funnel access.

### Optional integrations

Set these on `paradox-backend` only when you need the corresponding feature.
Any optional fields shown during Blueprint setup can be configured later under
**Environment**.

| Variables | Feature |
| --- | --- |
| `POCKET_OPTION_PARTNER_ID`, `POCKET_OPTION_API_TOKEN` | Pocket Option trader and deposit verification. The Blueprint supplies the API base URL. |
| `POCKET_OPTION_REF_RU_URL`, `POCKET_OPTION_REF_WW_URL` | Pocket Option registration links in the funnel. |
| `TELEGRAM_FUNNEL_TEST_ACCESS_CODE` | Private code for testing Mini App access without a real deposit. Send it in the bot chat after `/start`. |
| `TELEGRAM_SOURCE_CHANNEL_ID` | Source channel for copied funnel messages. |
| `TEAM_VIP_URL` | VIP team invite link. |
| `TELEGRAM_ADMIN_USER_IDS` | Admin-only bot commands. |
| `DEVSBITE_CLIENT_TOKEN` | Devsbite market data and analysis. The Blueprint supplies the API base URL. |
