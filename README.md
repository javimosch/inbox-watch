# inbox-watch

**Tell me only when a human actually replied.**

A non-interactive poller across GitHub, IMAP and Resend inbound. It prints what
is new since the last run, and exits **10** if there was anything — so a cron job
or systemd timer alerts you *only* when something real arrived.

```console
$ inbox-watch --human
[mailbox/reply] yohji.sakamoto@gmail.com: Re: benchmark collaboration
$ echo $?
10
```

## The point is triage, not polling

Inbound is mostly machine noise. On a domain with DMARC reporting on, aggregate
reports outnumber real mail several to one; add out-of-office autoreplies and CI
chatter and a naive notifier pings constantly. **A notifier that mostly cries wolf
gets muted, and then the reply that mattered is muted with it.**

So every item is classified, and only one kind is a person:

| kind | what it is | alert? |
|---|---|---|
| `dmarc` | aggregate reports | no — counted, never pinged alone |
| `autoreply` | out-of-office | yes, but **labelled**, so it doesn't read as a reply |
| `reply` | a human typed something | **yes** — this is what the tool is for |

## Install

Single file, Python 3, no dependencies.

```sh
git clone https://github.com/javimosch/inbox-watch && cd inbox-watch
install -m755 inbox-watch ~/.local/bin/inbox-watch
mkdir -p ~/.inbox-watch && cp config.example.json ~/.inbox-watch/config.json
$EDITOR ~/.inbox-watch/config.json
inbox-watch setup          # what each channel needs
```

Config holds what to watch; **env vars hold every secret**.

## Channels

| Channel | Needs | Notes |
|---|---|---|
| **mailbox** | `INBOX_TOKEN` + `mailbox.url` | A token-gated service exposing `GET /api/inbound` (e.g. [machin-resend-inbox](https://github.com/javimosch/machin-resend-inbox)). **Preferred over `resend`** — a read-only token cannot send mail as you. |
| **resend** | `RESEND_API_KEY` | Resend inbound directly. Note this key *can also send*. |
| **imap** | `ZOHO_IMAP_PASS` + `imap.user` | Any IMAP host; needs an app-specific password. |
| **github** | authenticated `gh` | Mentions/review-requests, plus specific issue/PR **threads** you name — the ones GitHub buries under subscribed-repo noise. |
| **linkedin** | — | Not built (no notifications API). |

Unconfigured channels report *why* they're disabled rather than failing.

## Two traps worth knowing

Both were found the hard way, in production.

**The cursor is consumed on read.** Any run marks items seen — so a manual
diagnostic run *eats the alert your cron would have sent*. Use `--peek` for
anything diagnostic; it reports without committing state.

**Two watchers on one host steal from each other.** Same cause: they share a
cursor, so whichever polls first wins and the other's alert never fires. Give
each its own with `--consumer NAME`, and run `--seed` once so a new consumer
doesn't alert on the entire backlog.

```sh
inbox-watch --peek --human            # diagnose safely
inbox-watch --consumer laptop --seed  # add a second watcher
inbox-watch --consumer laptop
```

## Agent-first output contract

Follows [cli-output-spec](https://cli-specs.intrane.fr/): **stdout is data**
(versioned JSON), **stderr is context and typed errors**, and the exit code is
the signal.

```console
$ inbox-watch                     # stdout
{ "ok": true, "version": "1.0.0", "new": [ ... ], "count": 1,
  "disabled": [ { "channel": "resend", "reason": "set RESEND_API_KEY …", "recoverable": true } ] }

$ inbox-watch help-json           # command catalog
$ inbox-watch guide               # embedded operator manual
```

| exit | meaning |
|---:|---|
| `0` | success — nothing new |
| `10` | success — **new items** (the cron signal) |
| `80` | input/validation (bad config) |
| `90` | precondition/resource |
| `100` | external/integration |
| `110` | internal |

`--exit-zero` forces 0 on success for runners that treat any non-zero exit as a
failure; read `count` instead.

Errors are typed, on stderr, with `recoverable` and `suggestions`:

```json
{"ok":false,"error":{"code":80,"type":"config_invalid","message":"…",
 "recoverable":true,"suggestions":["validate with: python3 -m json.tool …"]}}
```

No internal retries — the caller decides.

## Wire it to alert

```
*/15 * * * *  inbox-watch --human | grep . && <notify: telegram / webhook / mail>
```

`run.sh.template` is a worked example: poll, drop DMARC, label autoreplies, push
real replies to Telegram — and **log the delivery result**, because a send that
fails silently is the same failure mode this tool exists to prevent.

## supercli plugin

```sh
supercli plugins install ./plugin --on-conflict replace --json   # or: supercli/pull/363
supercli inbox-watch inbox peek      # safe diagnosis
supercli inbox-watch inbox check     # consumes the cursor
supercli inbox-watch self guide
```

`inbox check` and `inbox peek` are separate commands on purpose: the destructive
one should never be the one you reach for while debugging.

## Licence

MIT.
