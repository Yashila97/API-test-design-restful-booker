# Findings — restful-booker

Defects and observations from executing the suite. Severities are given as they would be
if this were a production payments API, since that is the useful comparison.

**Record your execution date and confirm each finding still reproduces before
publishing.** Public practice APIs get patched.

---

## RB-001 — No server-side validation on a monetary field

**Severity if in production:** High
**Found by:** TC-NG-003
**Endpoint:** `POST /booking`

`totalprice` accepts a negative number and stores it unchanged.

```
Request:  { "firstname": "Negative", "totalprice": -100, ... }
Response: 200 OK
          { "bookingid": 4821, "booking": { "totalprice": -100, ... } }
```

**Expected:** 400 with a validation message naming the field.

**Impact.** On a payments or booking system, a negative amount inverts the direction of
a transaction. Depending on what consumes the record downstream, it produces a credit
rather than a charge, or a negative line in a reconciliation report that has to be
manually investigated. The absence of validation at the API boundary also means any new
client can introduce the problem again, because the only protection is whatever
validation each client happens to implement.

**Note.** Pinned in the collection rather than asserted as a failure — see the
repository README for why.

---

## RB-002 — Inverted date range accepted

**Severity if in production:** Medium
**Found by:** TC-NG-004
**Endpoint:** `POST /booking`

A booking with `checkin` after `checkout` is created successfully.

```
Request:  { "bookingdates": { "checkin": "2026-11-08", "checkout": "2026-11-01" } }
Response: 200 OK
```

**Expected:** 400. A stay cannot end before it begins.

**Impact.** Any duration calculation on this record produces a negative number of
nights. In the domain I work in, the equivalent is accepting a bet on a fixture that has
already finished — the record looks structurally valid and fails only when something
tries to reason about it.

---

## RB-003 — Failed authentication returns 200

**Severity if in production:** Medium
**Found by:** TC-AU-002
**Endpoint:** `POST /auth`

An invalid password returns `200 OK` with `{ "reason": "Bad credentials" }` rather than
`401 Unauthorized`.

**Impact.** Not a security hole in itself — no token is issued — but it is a
correctness problem with real consequences. Any client that branches on the status code
will treat a failed login as successful and then fail confusingly further along. It also
breaks conventional monitoring, since failed authentication attempts do not appear as
4xx in the metrics, which removes the signal you would use to spot credential stuffing.

**Recommendation.** Return 401 with the reason in the body. Keep the message generic so
it does not distinguish "no such user" from "wrong password".

---

## Observations, not defects

| # | Observation | Why it's worth recording |
|---|---|---|
| O-1 | `DELETE` returns 201 rather than 204 | Unconventional but harmless. Worth knowing before writing a client |
| O-2 | Auth token supplied as a `Cookie` header, not `Authorization: Bearer` | Non-standard. Would need documenting for any consumer |
| O-3 | First request after idle can take 10s or more | Free-tier cold start, not a defect. Health check first in every cycle |
| O-4 | `additionalneeds` omitted from the response when not supplied | Absent rather than null. Clients must handle a missing key |
