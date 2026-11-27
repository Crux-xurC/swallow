# swallow — design

This is the English design document (v0.2 of the Japanese memo). The Japanese original is [ja/design-memo.md](ja/design-memo.md); where
the two differ, the code is right, then the original. How to use it: [../README.md](../README.md).

> **Where swallow stands now (2026-11-25).** swallow was designed as the program Citlalpol starts for a destination that goes to a
> stand-in, with the connection on stdin / stdout. Since Citlalpol v0.8, **Citlalpol starts no programs**: a `pass` with `to` reaches
> only a [zoozve](https://github.com/Crux-xurC/zoozve) gate in front of a local server the person runs
> ([Citlalpol's design, "What moved out"](https://github.com/Crux-xurC/Citlalpol/blob/main/docs/design.md#what-moved-out)). So
> `swallow serve` (stdio) has no one to start it today. How swallow is reached from now on — for example listening on a local port
> behind a zoozve gate — is open. What follows is the design as written, which still holds for what swallow does with a request.
>
> **Given up (2026-11-27).** It is not reached any more: answering a connection Citlalpol refused would make the program believe it got
> out. The name went to `citl swallow` (the refused attempts as a small table) and `citl digest` (a decision written into
> `access.json`) in [Citlalpol's design](https://github.com/Crux-xurC/Citlalpol/blob/main/docs/design.md#from-the-records-to-accessjson-swallow-digest-residual).

## What it is

swallow takes a connection and does **the work that needs to know a protocol**.

- **What it does.** Matches a request against what a project's designer wrote in `access.json` (`stages`: what is sent, what should come
  back), answers with the reply written there, and fires the program named there (`swallow serve`). Checks the written replies against
  the real service when asked (`swallow verify`).
- **The default.** Where the designer has not yet decided what to return or what to fire, swallow is the default receiver: it records
  what arrived (the material for deciding) and answers in a way that cannot be mistaken for the real service.
- **What it does not do.** Guess what a request means — it reads only what a person wrote. Block or route connections (Citlalpol).
- **Its place.** What embody is to Xipe Totec. It plays the part of Pact's mock server and provider verification, but from outside the
  program (at the network layer), so nothing is added to the application.

## The three stages

| | Citlalpol | swallow |
| --- | --- | --- |
| stage 1 — the connection (name, port, ClientHello) | decides (refuse / pass / elsewhere) | — |
| stage 2 — what is sent | the route only; does not read the content | reads the request, matches it against `send` |
| stage 3 — what should come back | — | answers `expect`'s `reply`, fires `fire` |
| against the real service | — | `swallow verify` compares |
| records | every attempt | the result of matching (`matched`, `fired`, `captured`) |

## `stages` in `access.json`

```json
{ "host": "pay.example.com", "port": 443,
  "stages": {
    "send": [
      { "id": "status", "match": { "method": "GET",  "path": "/status" } },
      { "id": "charge", "match": { "method": "POST", "path": "/v1/charges" } } ],
    "expect": [
      { "on": "status", "reply": "replies/status.json" },
      { "on": "charge", "reply": "replies/charge.json", "fire": "after_charge", "verify": "never",
        "why": "a charge is never replayed; the reply is the shape the client reads" } ] } }
```

- `send`: what is sent, each with a `match` and an optional `do` (`stand_in` — the default —, `refuse`, `pass`).
- `expect`: what should come back for a `send` row: `reply` (a table in the repository, `{status, headers, body}`, inline or a JSON file
  beside `access.json`), `fire` (the **name** of a program to fire when it arrives — what it starts is this machine's setting, never
  the project's), `verify` (may it be checked against the real service: `safe` / `never` / `elsewhere`).
- What `match` can compare is per protocol; swallow decides it (HTTP first).

## `swallow serve`

The first, plain version (python3, one file, `bin/swallow`):

- Reads one request (head up to 64 KiB, body up to 1 MiB by `Content-Length`). Not an HTTP/1.1 request line: closes without answering
  and reports `captured` with `kind: not-http`.
- Matches `send` from the top (`method` equal or `*`, `path` a glob). The matched row's `do`: none or `stand_in` → the `expect` row with
  the same `on` and a `reply`; `refuse` → `403` with `X-Swallow: refused`; `pass` → `500` with `X-Swallow: misplaced` (passing is
  Citlalpol's decision, not swallow's).
- A `body` that is a JSON value is sent as `application/json`.
- **Undecided** (declared but no reply, or not declared): `501 Not Implemented` with `X-Swallow: undecided`.
- The result is one JSON line per request on the fd named by `CITLALPOL_RESULT_FD`: `request` (method, path with a query's **names**
  only, header **names**, body **size** — never values), `matched`, `replied`, `default`, `captured`.

```json
{"request":{"method":"POST","path":"/v1/charges","headers":["content-type","authorization"],"body_bytes":312},
 "matched":"charge","replied":"replies/charge.json","fired":"after_charge","default":false}
```

What it was given by the starter (the stand-in contract of Citlalpol up to v0.7): the connection on stdio (or an fd), and
`CITLALPOL_PROTOCOL`, `CITLALPOL_HOST`, `CITLALPOL_PORT`, `CITLALPOL_SNI`, `CITLALPOL_RUN`, `CITLALPOL_LABEL`, `CITLALPOL_ACCESS` (a JSON
file with the destination's entry and the directory of `access.json`).

### When TLS cannot be read

Without a local CA the client trusts, or with a pinned certificate, swallow cannot read inside TLS. It then records only stage 1 (SNI,
ALPN) and closes: what cannot be read is recorded as such.

## `swallow verify` *(planned)*

- For each `expect` row, by its `verify`: `safe` (no side effects) is sent to the real service; `never` is not; `elsewhere` goes to the
  verification destination written there.
- What to compare can be written in `expect` (times and ids change every time); otherwise the status code and the body's shape (keys
  and types).
- One JSON line per row: `{"row": "charge", "result": "match" | "differ" | "skipped", "why": "…"}`.
- The talk with the real service goes through Citlalpol, so it is recorded too.

## Pact

- The stage-3 table (`send`, `expect`, `reply`) can be written out in Pact's format (`swallow export pact`), keeping a way to Pact
  brokers and providers' CI.
- The other way (Pact contracts → `stages`) when the shapes fit.

## The first acceptance: the ping, steps c and d

`citl-ping` connects to a name, sends `GET /ping` and waits for `pong`.

| | Condition | Expected of swallow |
| --- | --- | --- |
| c | `pong.citlalpol.test` declared, no `reply` in `expect` | `501` with `X-Swallow: undecided`, `captured` recorded (S1) |
| d | `expect` has a `reply` (`pong`) | matches `GET /ping`, answers `pong`, `matched` recorded (S2) |

(Steps a — outside the wall — and b — not declared — are Citlalpol's alone. In Citlalpol's current acceptance, c and d are played by a
zoozve gate in front of a pong server instead.)

## Milestones

| | What | Done when |
| --- | --- | --- |
| S1 ✔ | the default only (HTTP/1.1): record and answer 501 | step c passes; the value of `Authorization` is not kept |
| S2 ✔ (but `draft`) | matching and replies: `send`'s `match`, `expect`'s `reply`, `swallow draft` | step d passes; a draft made from `captured`, corrected and put in, lets the request go further next time |
| S3 | `fire` | the named program fires; `fired` is recorded |
| S4 | `verify` | only `safe` rows go to the real service; a result per row |
| S5 | TLS (with a local CA), protocols other than HTTP, `export pact` | |

## Open

- how swallow is reached now that Citlalpol starts no programs (see the top)
- the language (the same as Citlalpol's, Elixir?)
- whether a `reply` may carry values from the request, and how far
- whether `fire` fires before or after the reply, and whether it may change the reply
- the answer for undecided requests: is HTTP `501` right (or `503`, or per kind of application)
- what `match` can hold (path patterns, query, headers, part of the body)
