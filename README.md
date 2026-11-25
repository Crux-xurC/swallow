# swallow

Design: [docs/design.md](docs/design.md) (the Japanese original: [docs/ja/](docs/ja/))

swallow takes the connections [Citlalpol](https://github.com/Crux-xurC/Citlalpol) hands it and does what needs to know a protocol:
it matches a request against what a project's designer wrote in `access.json` (`stages`: what is sent, what should come back),
answers with the reply written there, fires the program named there, and checks the written replies against the real service when
asked (`swallow verify`). Where the designer has not yet said what to return or what to fire, swallow is the default: it records what
arrived — the material for deciding — and answers in a way that cannot be mistaken for the real service.

It never guesses what a request means; it reads what a person wrote. It plays the part of Pact's mock server and provider verifier,
but from outside the program under test: nothing is added to the application.

swallow is to Citlalpol what embody is to Xipe Totec: Citlalpol works without it (it then stops at the connection), and swallow
follows Citlalpol's contract for programs it starts.

## Use

swallow was designed to be started by Citlalpol, not by hand: a destination of `access.json` that goes to a stand-in without naming one gets
`swallow serve`, with the connection on stdin / stdout. What it reads is the destination's `stages`:

```json
{ "host": "pay.example.com", "do": "stand_in",
  "stages": { "send":   [ { "id": "status", "match": { "method": "GET", "path": "/status" } },
                          { "id": "charge", "match": { "method": "POST", "path": "/v1/charges" } } ],
              "expect": [ { "on": "status", "reply": "replies/status.json" } ] } }
```

`replies/status.json` (beside `access.json`): `{"status": 200, "headers": {...}, "body": ...}`. Here `GET /status` gets that reply;
`POST /v1/charges` is declared but nothing to return is decided, so it gets `501 Not Implemented` with `X-Swallow: undecided`, and
the request (method, path, the names of its headers, the size of its body — never their values) is recorded as captured: the
material for deciding. Needs python3.

**Status:** a first, plain version (2026-11-24): HTTP/1.1, `send` / `expect` / inline or file replies, the undecided default.
Not yet: `fire`, `verify`, `draft`, TLS, other protocols. Since Citlalpol v0.8 Citlalpol starts no programs (a `to` reaches only a
[zoozve](https://github.com/Crux-xurC/zoozve) gate), so nothing starts `swallow serve` today; how it is reached from now on is open —
see [docs/design.md](docs/design.md).
