# Osysharp.Sms.FortySixElks

[46elks](https://46elks.com) as a sender for [Osysharp.Sms](https://osyrin.com/templates/kits/sms/): the app's one-time codes, reminders and
notices go through the Nordic carrier — alphanumeric sender names that actually deliver in Sweden, EU-only handling, and
per-message pricing a small app can read.

## Use it

```osy
// app.osy
use Osysharp.Sms@0;
use Osysharp.Sms.FortySixElks@0 { egress "api.46elks.com"; }
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
`[Principal]`. [Osysharp.FortySixElks](https://osyrin.com/templates/kits/forty-six-elks/) stays the plain client — an app texting through it with an account
model of its own is not handed a second principal — and this is the few lines that adapt it, the shape
`Osysharp.Mail.Resend` has for mail.

## Its tests

Its tests stub `Http.Post` and assert the form body that reaches 46elks' API; none sends a real message.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Http;
using Osysharp.Identity;
using Osysharp.Sms;

[Test(Desk)]
void a_message_reaches_46elks_as_a_form_and_answers_its_id() {
  runas (Ada) {
    var sms = Text("+46701234567", "Your code is 123456.");
    UnitOfWork.Commit();
    Workflow.Settle(sms);
  }
  runas (Ada) {
    var sent = OutgoingSms.Single(m => m.Text == "Your code is 123456.");
    Assert.Equal(SmsStatus.Sent, sent.Status);
    Assert.Equal("46elks", sent.SentWith);
    Assert.Equal("s-elks-1", sent.ProviderId);
    var wire = WireLog.Single(w => w.Body.Contains("123456"));
    Assert.Equal("https://api.46elks.com/a1/SMS", wire.Url);
    // The `+` is escaped: a bare `+` in a form body is a space, and 46elks would read no country code.
    Assert.Contains("to=%2B46701234567", wire.Body);
    Assert.Contains("from=Acme", wire.Body);
    Assert.Contains("message=Your%20code%20is%20123456.", wire.Body);
    Assert.Equal("Basic " + Convert.ToBase64String(Text.ToBytes("u_test:p_test")), wire.Auth);
  }
}
```

From [`tests/elks.test.osy`](tests/elks.test.osy).

```osy
using Osysharp.Http;
using Osysharp.Identity;
using Osysharp.Sms;

[Test(Desk)]
void with_no_password_the_message_is_held_and_says_which_credential() {
  ElksPassword.Stub(() => "");
  runas (Ada) {
    var sms = Text("+46701234567", "Held.");
    UnitOfWork.Commit();
    Workflow.Settle(sms);
  }
  runas (Ada) {
    var held = OutgoingSms.Single(m => m.Text == "Held.");
    Assert.Equal(SmsStatus.Held, held.Status);
    // The adapter's own NotReadyReason, not the contract's default.
    Assert.Equal("46elks has no API password — set `Password` on ElksSmsSender in app.Sms.", held.Problem);
    Assert.False(WireLog.Any(w => w.Body.Contains("Held.")));
  }
}
```

From [`tests/elks.test.osy`](tests/elks.test.osy).

## Learn more

- [The Osysharp.Sms.FortySixElks reference](https://osysharp.com/reference/ui/sms-kit/)
- [Osysharp.Sms.FortySixElks in the package catalogue](https://osysharp.com/packages/#osysharp-sms-fortysixelks)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
