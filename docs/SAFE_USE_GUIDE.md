# Safe use guide

How to use this without losing your WhatsApp account, and what to do when WhatsApp
changes something underneath it.

## The honest risk picture

This is an unofficial client. It logs in as a **linked device** — the same mechanism as
WhatsApp Web on a laptop — but it is automated, and WhatsApp's terms do not permit
unofficial clients. Accounts do get banned.

What matters is **behaviour**, not fingerprints. In practice the signals associated with
enforcement are:

| Signal | Risk |
|---|---|
| Messaging people who never messaged you | **Highest** |
| Bursts of outbound messages | High |
| Sub-second replies, around the clock | High |
| A brand-new number automating on day one | High |
| Replying to inbound conversations at human speed | Low |
| Reading, searching, summarising | Lowest — no outbound traffic at all |

The server's pacing addresses the middle rows. Nothing addresses the first row except
your own judgement.

## Recommended path

**1. Start read-only.** Run with `--read-only` for the first few days:

```bash
node server/index.js --read-only
```

Your assistant can list and search chats, read history, look up contacts and inspect
groups, but cannot send or change anything. This is where most of the value is anyway —
searching thousands of messages is something you cannot do by hand.

**2. Use a spare number if you have one.** A SIM you can afford to lose removes the
worst-case outcome entirely.

**3. Turn on sending once you trust it.** Drop `--read-only`. Pacing is on by default.

**4. Check the ceilings before bulk work.** Ask your assistant to call
`whatsapp_pacing_status`. It reports messages sent in the last hour and day, new chats
contacted this hour, and the current caps.

## What the pacing actually does

Every outbound message:

1. Waits out a global cooldown (2.5 s) and a per-chat cooldown (6 s)
2. Pauses 1.2–4.5 s, as if reading and thinking
3. Shows a typing indicator for a duration proportional to the message length
   (~5 characters/second, capped at 12 s)
4. Clears the indicator, then sends

Sends are serialised — two messages never go out at once — and every delay is jittered,
because a fixed delay is itself a machine signature.

Ceilings: **40/hour**, **200/day**, **5 new chats/hour**. Exceeding one returns an error
the model can read and respond to, rather than silently queueing.

To adjust them you must rebuild from the source project (`src/whatsapp/humanize.js`).
`--no-humanize` disables pacing entirely and is not recommended.

## Protecting the session

Linking creates `.wwebjs_auth/` beside the server. **This folder is a credential.** It
holds a Chrome profile containing WhatsApp's Signal keys, and anyone who copies it can
read and send as you until the device is unlinked. No password is involved.

- Keep it out of OneDrive, Dropbox, and any backup that leaves the machine
- Never commit it
- To revoke immediately: **WhatsApp → Settings → Linked Devices → log out**

Unlinking invalidates the session everywhere at once. It is the kill switch.

## Destructive tools

Thirteen tools cannot be undone: delete for everyone, block, unblock, leave group, remove
participants, delete channel, transfer channel ownership, clear chat, delete chat, delete
profile picture, delete group picture, delete contact, revoke status.

Each is flagged `[DESTRUCTIVE — cannot be undone]` in its description, but a model acting
on an ambiguous instruction can still fire one. `--read-only` removes them entirely.

`logout` is **not exposed** by default — it destroys the session and forces a re-link.
`--allow-logout` enables it.

## When WhatsApp breaks it

WhatsApp Web ships new builds without notice, and this library breaks silently when it
does — usually with a one-character minified error that says nothing.

Ask your assistant to run these, in order:

| Tool | Answers |
|---|---|
| `whatsapp_diag_versions` | Is the live web build newer than the version the library targets? |
| `whatsapp_patch_verify` | Do the bundled runtime patches still work? |
| `whatsapp_diag_chat_sweep` | How many chats still map correctly, and how do failures cluster? |
| `whatsapp_diag_modules` | Which of WhatsApp's internal modules moved or disappeared? |
| `whatsapp_diag_chat_model` | Replays the library's own mapping step by step, naming the exact failing line |

That last one is the only reliable way past a minified error. Report what it says on the
issue tracker — it is far more useful than "it stopped working".

## Known limits

- **Buttons are silently downgraded.** `send_buttons` reports success, but WhatsApp strips
  the buttons and stores plain text. `send_list` survives. A successful API call is not
  proof the feature worked — check the stored message `type`.
- **Poll votes lose their option text.** `vote_update` gives the option's numeric id, not
  its label; fetch the poll message to map the two.
- **Some tools need a WhatsApp Business account** — labels, customer notes, orders,
  payments.
- **Seven tools take a local file path** (`send_media`, `send_voice`, `send_sticker`,
  `set_profile_pic`, `group_picture`, `channel_picture`, `media_probe`). They read the
  **server's** filesystem, which is your own machine when running locally.
- **Six library functions are broken upstream** and cannot be patched from outside:
  `pinned_messages`, `edit`, `group_description`, `group_revoke_invite`,
  `channel_subscribers`, and `block`/`unblock`.
