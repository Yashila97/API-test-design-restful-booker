# API Test Cases — restful-booker

Base URL: `https://restful-booker.herokuapp.com`
Docs: https://restful-booker.herokuapp.com/apidoc/index.html
Technique key: BVA = boundary value · EP = equivalence partition · NEG = negative ·
ST = state transition · SEC = security · NF = non-functional

Every case is also in the Postman collection with the same ID.

---

## Authentication

| ID | Technique | Request | Expected |
|---|---|---|---|
| TC-AU-001 | Happy path | `POST /auth` with valid username and password | 200. Body contains a non-empty `token` string |
| TC-AU-002 | NEG | `POST /auth` with a wrong password | **No token in the body.** Observed: 200 with a `reason` field rather than 401 — recorded as finding RB-003 |
| TC-AU-003 | NEG | `POST /auth` with empty strings | No token issued |
| TC-AU-004 | NEG | `POST /auth` with no body | Rejected. No token, no server error |
| TC-AU-005 | SEC | `POST /auth` and inspect the request | Sent over HTTPS. Credentials in the body, not the query string |

## Booking lifecycle

| ID | Technique | Request | Expected |
|---|---|---|---|
| TC-BK-001 | Happy path | `POST /booking` with all fields valid | 200. Numeric `bookingid` above 0. Echoed booking matches the payload |
| TC-BK-002 | Happy path | `GET /booking/{id}` for the created booking | 200. **Persisted** values match what was sent, including both dates. Verifies storage rather than the echo |
| TC-BK-003 | SEC | `PUT /booking/{id}` with no token | 403. Authorisation is a separate check from validation |
| TC-BK-004 | Happy path | `PUT /booking/{id}` with a valid token cookie | 200. Updated fields returned |
| TC-BK-005 | EP | `PATCH /booking/{id}` with one field only | 200. Patched field changed, untouched fields preserved at their previous values |
| TC-BK-006 | Happy path | `DELETE /booking/{id}` with a valid token | 200 or 201 |
| TC-BK-007 | ST | `GET /booking/{id}` after deletion | 404. A deleted record must not be readable |
| TC-BK-008 | ST | `DELETE /booking/{id}` a second time | Not 200. The caller must be able to tell "removed" from "never existed" |
| TC-BK-009 | SEC | `DELETE /booking/{id}` with an expired or invalid token | 403 |

## Boundary and negative

| ID | Technique | Request | Expected |
|---|---|---|---|
| TC-NG-001 | NEG | `POST /booking` with `{}` | Not 200. No record created |
| TC-NG-002 | BVA | `totalprice: 0` | Stored as exactly 0, not dropped or coerced to null |
| TC-NG-003 | BVA | `totalprice: -100` | **Accepted and stored.** Finding RB-001 — no server-side validation on a monetary field |
| TC-NG-004 | BVA | `checkin` after `checkout` | **Accepted.** Finding RB-002 — inverted date range not validated |
| TC-NG-005 | NEG | `GET /booking/99999999` | 404, not an empty object or 200 |
| TC-NG-006 | NEG | `POST /booking` with malformed JSON | Not 200. No stack trace in the response body |
| TC-NG-007 | BVA | `firstname` of ~250 characters | Handled without a 5xx. Record whether it truncates |
| TC-NG-008 | SEC | `firstname` containing `<script>alert(1)</script>` | Stored as text. `Content-Type` remains JSON |
| TC-NG-009 | EP | `depositpaid` sent as the string `"true"` instead of boolean | Behaviour recorded. Type coercion on a boolean is worth knowing about |
| TC-NG-010 | EP | `totalprice` sent as the string `"250"` | Behaviour recorded — coerced, rejected, or stored as a string |
| TC-NG-011 | NEG | `bookingdates` omitted entirely | Rejected, or defaulted with the default documented |
| TC-NG-012 | NEG | `POST /booking` with `Content-Type: text/plain` | Rejected. Not silently parsed |

## Listing, filtering, non-functional

| ID | Technique | Request | Expected |
|---|---|---|---|
| TC-LS-001 | Happy path | `GET /booking` | 200. JSON array |
| TC-LS-002 | Happy path | `GET /booking?firstname=X&lastname=Y` | 200. Array. Results limited to matches |
| TC-LS-003 | EP | `GET /booking?checkin=...&checkout=...` | 200. Date filter applied |
| TC-LS-004 | NEG | `GET /booking?checkin=not-a-date` | Handled without a 5xx |
| TC-NF-001 | NF | `GET /ping` | 200 or 201. Run first in every cycle |
| TC-NF-002 | NF | `GET /booking` timing | Under 5s warm. Note that a cold free-tier host can take much longer, which is not a defect |

---

## Notes on technique

**Why TC-BK-002 exists as a separate case.** The create response echoes the payload you
sent. It is generated from your own request and proves nothing about the write. Reading
the record back through an independent GET is the only assertion that tests persistence,
and it catches the case where a field is accepted, echoed, and then silently dropped.

**Why TC-BK-005 asserts an untouched field.** A partial update that quietly nulls
everything you did not send is a real and common defect. The assertion that matters is
on the field you *didn't* touch.

**Why the type-coercion cases (TC-NG-009, TC-NG-010) are worth the time.** An API that
accepts `"250"` as a string and stores it as a string will pass every happy-path test
and then fail arithmetic downstream. On a payments endpoint this is the kind of defect
that surfaces in a reconciliation report weeks later.
