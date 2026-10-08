# Osysharp.Sms.Elks

[46elks](https://46elks.com) as a sender for [Osysharp.Sms](https://osyrin.com/templates/kits/sms/): the app's one-time codes, reminders and
notices go through the Nordic carrier — alphanumeric sender names that actually deliver in Sweden, EU-only handling, and
per-message pricing a small app can read.

## Use it

```osy
// app.osy
use Osysharp.Sms@0;
use Osysharp.Sms.Elks@0 { egress "api.46elks.com"; }
```

```osy
app.Secrets = [ new Secret("ElksUsername"), new Secret("ElksPassword") ];

app.Sms = new SmsSetup {
  Sender = new ElksSmsSender {
    Username = Security.IsSecretSet("ElksUsername") ? Secret.ElksUsername : "",
    Password = Security.IsSecretSet("ElksPassword") ? Secret.ElksPassword : "",
  },
  From = "Acme",
};
```

- **Not ready until both credentials are set** — and then messages are HELD in the outbox with a sentence naming the
  missing one (`46elks has no API password — set Password on ElksSmsSender in app.Sms.`), so an app runs before anybody
  has an account with 46elks.
- **`DryRun = true`** validates and prices every message without sending it — for a staging app that must never text a
  real person.
- **The credentials are the API username and password** from the 46elks dashboard, not the dashboard login.

⚠ 46elks takes no idempotency key. The outbox sends each message once when the carrier answers; a delivery whose answer
was lost in transit and is retried can text twice.

## Why a package of its own

The text-message contract is people-aware (its outbox is guarded by `Osysharp.Identity`), and an app has one
`[Principal]`. [Osysharp.Elks](https://osyrin.com/templates/kits/elks/) stays the plain client — an app texting through it with an account
model of its own is not handed a second principal — and this is the few lines that adapt it, the shape
`Osysharp.Mail.Resend` has for mail.

## Its tests

Its tests stub `Http.Post` and assert the form body that reaches 46elks' API; none sends a real message.
