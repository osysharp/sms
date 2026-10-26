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
use Osysharp.Sms.FortySixElks@0 { egress "api.46elks.com"; }   // the carrier — or your own ISmsSender
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

- **[Osysharp.Sms.FortySixElks](https://osyrin.com/templates/kits/sms-forty-six-elks/)** — 46elks: Nordic alphanumeric senders, EU-only handling, a dry run.
- **Your own** — a class implementing `ISmsSender` over `Http.*` for any provider, with its host declared in your
  package's manifest.

`osy kits Sms` lists every implementation in scope and which one this app wired.

## What it is not

⚠ **An SMS is not a second factor an app can require.** A code by text proves a number *today*; a SIM swap or a port-out
moves the number to somebody else (NIST 800-63B lists SMS as "restricted"). The UserAccounts kit uses it to prove a number,
to sign in with a code, and as an optional second step an account may choose — never as the factor an app demands of its
admins.

## Its tests

Its tests are in `tests/` — every one asserts what a recording sender received; none sends a real message.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.Sms;
using SmsKitTests;

[Test(Desk)]
void a_message_goes_after_the_commit_and_is_recorded_as_sent() {
  runas (Ada) {
    var sms = Text("+46701234567", "Your table is ready.");
    UnitOfWork.Commit();
    Workflow.Settle(sms);
  }
  runas (Ada) {
    var sent = OutgoingSms.Single(m => m.Text == "Your table is ready.");
    Assert.Equal(SmsStatus.Sent, sent.Status);
    Assert.Equal("recording", sent.SentWith);
    Assert.Equal("carried-+46701234567", sent.ProviderId);
    var carried = Carried.Single(c => c.Text == "Your table is ready.");
    Assert.Equal("+46701234567", carried.To);
    // No sender on the message: the app's default.
    Assert.Equal("Acme", carried.From);
  }
}
```

From [`tests/sms.test.osy`](tests/sms.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Sms;
using SmsKitTests;

[Test(Desk)]
void a_number_is_normalised_before_it_is_written() {
  runas (Ada) {
    var sms = Text("+46 70-123 45 67", "Spaced.");
    UnitOfWork.Commit();
    Workflow.Settle(sms);
  }
  runas (Ada) { Assert.Equal("+46701234567", Carried.Single(c => c.Text == "Spaced.").To); }
  Assert.Equal("+46701234567", Normal("0046 70 123 45 67"));
  Assert.Equal("", Normal("0701234567"));
}
```

From [`tests/sms.test.osy`](tests/sms.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Sms;
using SmsKitTests;

[Test(Desk)]
void a_local_number_is_refused_where_it_was_written() {
  runas (Ada) { Assert.Throws(() => Text("0701234567", "No country code.")); }
}
```

From [`tests/sms.test.osy`](tests/sms.test.osy).

## Learn more

- [The Osysharp.Sms reference](https://osysharp.com/reference/ui/sms-kit/)
- [Osysharp.Sms in the package catalogue](https://osysharp.com/packages/#osysharp-sms)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
