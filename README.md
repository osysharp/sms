# Osysharp.Sms

Text messages an app sends — a one-time code, a booking reminder, "your order has shipped" — through one call, whichever
company carries them. The message is written in the same commit as whatever it is about, delivered after that commit,
retried when the carrier is down, and **held** (kept, with the reason) when the app has no carrier yet — so an app works
from its first minute, and nothing it meant to send is lost.

```osy
SendSms(new SmsMessage { To = booking.Customer.Mobile, Text = $"See you at {booking.StartsAt:HH:mm}." }, $"reminder-{booking.Id}");
```

## Use it

```osy
// app.osy
use Osysharp.Identity@0;
use Osysharp.Sms@0;
use Osysharp.Sms.Elks@0 { egress "api.46elks.com"; }   // the carrier — or your own ISmsSender
use Osysharp.Workflow;
```

```osy
// Who may send is the app's to say — nobody else can put a message in the outbox:
partial entity OutgoingSms { security { allow create, read when IsClerk; } }

// What carries it — here 46elks. No sender: messages are held.
app.Sms = new SmsSetup {
  Sender = new ElksSmsSender {
    Username = Security.IsSecretSet("ElksUsername") ? Secret.ElksUsername : "",
    Password = Security.IsSecretSet("ElksPassword") ? Secret.ElksPassword : "",
  },
  From = "Acme",
};
```

## What it does

- **`SendSms(message, idempotencyKey?)`** writes a row to the outbox (`OutgoingSms`) and answers it. The same key twice
  is one message, so a handler that runs twice texts once.
- **A number is written E.164 or not at all.** `NormalisedPhone("+46 70-123 45 67")` is `+46701234567`; a local form
  (`0701234567`) is refused where the message is written, naming the missing country code — a carrier's refusal of it
  would arrive later and name a parameter.
- **The outbox is the delivery record**: `Queued`, then `Sent` (when, through which sender, the carrier's id) or `Held`
  (and why — the sender's own `NotReadyReason()`).
- **Nobody reads the outbox who was not granted — not even the recipient.** A one-time code sits in those rows; reading
  it there would skip the phone it was sent to.

## Senders

A sender is anything implementing `ISmsSender` — `Key`, `IsReady()`, `SendSms(message, idempotencyKey)`, and optionally
`NotReadyReason()` — so the app changes carrier by changing one setting.

- **[Osysharp.Sms.Elks](https://osyrin.com/templates/kits/sms-elks/)** — 46elks: Nordic alphanumeric senders, EU-only handling, a dry run.
- **Your own** — a class implementing `ISmsSender` over `Http.*` for any provider, with its host declared in your
  package's manifest.

`osy kits Sms` lists every implementation in scope and which one this app wired.

## What it is not

⚠ **An SMS is not a second factor an app can require.** A code by text proves a number *today*; a SIM swap or a port-out
moves the number to somebody else (NIST 800-63B lists SMS as "restricted"). The Accounts kit uses it to prove a number,
to sign in with a code, and as an optional second step an account may choose — never as the factor an app demands of its
admins.

## Its tests

Its tests are in `tests/` — every one asserts what a recording sender received; none sends a real message.
