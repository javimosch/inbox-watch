# inbox-watch

Non-interactive poller for the intrane outreach inboxes. Prints NEW items since the last
run as JSON (or `--human`), exit **10** if any new items, else 0 — so a cron/timer can
alert only when something actually arrives. State: `~/.inbox-watch/state.json`.

    inbox-watch            # JSON: {"new":[...],"count":N,"disabled":[...]}
    inbox-watch --human    # readable lines
    inbox-watch setup      # per-channel enablement instructions

## Channels

| Channel | Status | Needs |
|---|---|---|
| **github** | ✅ working now | nothing — uses the authenticated `gh` CLI. Surfaces inbound only (mention/comment/review_requested/team_mention + new non-bot comments on the x402 PR #2612 thread); drops CI/self-authored/subscribed noise. |
| **zoho** (javi@intrane.fr) | ⚙️ needs a credential | `export ZOHO_IMAP_PASS=<Zoho app-specific password>` (Zoho Mail → Settings → Security → App Passwords; IMAP ON). Then polls UNSEEN INBOX over IMAPS (imap.zoho.eu:993), skips no-reply/digest mail. This is where cold-email replies land. |
| **linkedin** | 🚧 not built | LinkedIn has no notifications API. The authed session exists (`~/.config/intrane-gtm/li_at`), but a CDP scraper of `linkedin.com/notifications` needs writing before reply/reaction-watching works. |

## What I need to actually watch your inboxes

- **GitHub:** nothing — already live.
- **Zoho (the important one — outreach replies):** a **Zoho app-specific password**. That's the single missing credential. With it, `inbox-watch` sees replies to your 31 cold emails non-interactively.
- **LinkedIn:** a bit of code (a notifications-page CDP fetcher), plus the session already present.

## Wire it to alert (optional)

    */15 * * * *  inbox-watch --human | grep . && <notify: telegram/relais/email>

Or a systemd timer. Pipe new items to the perrus Telegram bot or POST them to a relais inbox.
