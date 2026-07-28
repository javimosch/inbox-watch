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

## Inbound classification (2026-07-28)

Not all inbound mail deserves an interruption. `classify_inbound()` sorts it into three kinds and
`run.sh` decides what to send:

| kind | what it is | notified? |
|---|---|---|
| `dmarc` | machine-generated aggregate reports | **no** — counted, and appended to a ping you were getting anyway |
| `autoreply` | out-of-office | yes, but **labelled** `[auto-reply]` so it doesn't read as a reply |
| `reply` | a person typed something | yes — this is what the tool is for |

Three of four consecutive pings had been DMARC reports. An alert channel earns its attention by
being quiet when there is nothing to say; one that mostly cries wolf gets muted, and then the
reply that mattered is muted with it.

The **poller still reports everything** — the split is a notification policy, not data loss. A
jump in the suppressed DMARC count is itself a deliverability signal, which is why it rides along
on pings rather than being discarded.

An item with **no** `kind` is treated as a real reply: an older poller or a new channel must fail
loud, never be silently swallowed.

`run.sh.template` is the versioned copy; the deployed `/opt/inbox-watch/run.sh` has the secrets
filled in. (Those secrets are inline in the deployed script — worth moving to an env file
someday, but not changed here.)
