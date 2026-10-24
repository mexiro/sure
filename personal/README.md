# Personal setup: N26, Deutsche Bank, TF Bank

How I run this fork of Sure on my MacBook. Everything runs locally in Docker,
and bank data stays in the Postgres volume on the Mac.

## 1. Run it

Install [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop/), then:

```bash
git clone https://github.com/mexiro/sure.git ~/sure
cd ~/sure
cp compose.example.yml compose.yml      # git ignores compose.yml
cp personal/.env.example .env           # git ignores .env
openssl rand -hex 64                    # paste into SECRET_KEY_BASE in .env
```

Fill in `.env` (secret key, a database password, Anthropic API key), then start:

```bash
docker compose -f compose.yml -f personal/compose.override.yml up -d
```

Open https://localhost and create your account. The first user is the admin.
The browser warns about the certificate the first time (it comes from a local
Caddy CA, not a public one): choose *Show details → visit this website* in
Safari or *Advanced → Proceed* in Chrome. http://localhost:3000 also works,
but bank connections need the https address.

Stop with the same command, replacing `up -d` with `down`. Data is kept.

## 2. Connect N26 and Deutsche Bank (Enable Banking)

1. Sign up at [enablebanking.com](https://enablebanking.com/) and create an
   application. It gives you an **Application ID** and a **private key / certificate**.
   Choose **Production** and *Generate in the browser*; keep the downloaded
   `.pem` file private.
2. Register the redirect URL `https://localhost/enable_banking_items/callback`
   (Enable Banking rejects `http://`). Open Sure at https://localhost, go to
   **Settings → Bank sync → Enable Banking**; the callback URL it shows must match.
3. Paste the Application ID and the full contents of the `.pem` file
   (`open -e ~/Downloads/*.pem`), choose **Germany**, save.
4. **Add Connection** → pick N26, log in on N26's page. Repeat for Deutsche Bank.

PSD2 consent expires (usually after 90–180 days). When Sure shows
"Session expired", click **Reconnect**.

If a bank isn't listed, use CSV import instead (step 3).

## 3. CSV import (fallback for any bank)

- **N26:** app or web → Account → Downloads → CSV.
- **Deutsche Bank:** online banking → Umsatzanzeige → Export → CSV.

In Sure: **Imports → New import → Transactions (CSV)**, upload the file, then
map the columns (date, amount, name). German exports use `;` as separator and
`,` as decimal mark — set those in the import options.

## 4. TF Bank credit card

1. Create a **Credit Card** account in Sure for the TF Bank card.
2. Download the monthly statement PDF from TF Bank online banking.
3. **Imports → New import → PDF**, choose *credit card statement*. Claude reads
   the PDF and extracts the transactions. Review them before publishing.

This needs `ANTHROPIC_API_KEY` set. If TF Bank also offers a CSV export, the CSV
import in step 3 works too.

## 5. AI assistant

With `LLM_PROVIDER=anthropic` and `ANTHROPIC_API_KEY` set, the chat sidebar can
answer questions about your accounts and spending. The same key powers
auto-categorization and the PDF import. Set a monthly spend limit in the
Anthropic console. Details: [docs/hosting/ai.md](../docs/hosting/ai.md).

## Touch ID sign-in

In Sure (at https://localhost): **Settings → Security**. First **Enable 2FA**
with an authenticator app (Google Authenticator, 1Password, …) and save the
backup codes; Sure requires it before adding a passkey. Then **Add passkey or
security key** and confirm with Touch ID. Afterwards you can sign in with your
fingerprint. Log
out when you're done (or use a strong password) so anyone using the Mac while
it's unlocked doesn't land straight in your finances.

## Backups

Backups go to `~/sure-backups` (set by `BACKUP_DIR` in `.env`), outside the
repo so they can never be pushed to GitHub. Each one holds the full database,
uploaded files and `.env` with your keys — keep it private.

```bash
cd ~/sure
alias sure-compose='docker compose -f compose.yml -f personal/compose.override.yml'
sure-compose run --rm backup create          # back up now
sure-compose --profile backup up -d backup   # daily at 02:00 UTC, keeps 7 days
sure-compose run --rm backup list            # list backups
```

Copy `~/sure-backups` to an external drive now and then; a backup on the same
laptop doesn't survive losing the laptop. Restoring: see
[docs/hosting/docker.md](../docs/hosting/docker.md#backups-upgrades-and-rollbacks).

## Keeping the fork updated

On GitHub, click **Sync fork** on `mexiro/sure`, then on the Mac:

```bash
git pull
docker compose -f compose.yml -f personal/compose.override.yml pull
docker compose -f compose.yml -f personal/compose.override.yml up -d
```

Everything personal lives in `personal/`, so syncing never conflicts.
