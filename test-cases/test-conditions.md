# Test Condition Analysis — restful-booker

The reasoning behind the case selection.

---

## 1. Equivalence partitions — `totalprice`

| Partition | Representative | Valid on a payments API? | Case |
|---|---|---|---|
| Negative | -100 | No | TC-NG-003 |
| Zero | 0 | Edge — legitimate for a comp booking | TC-NG-002 |
| Positive integer | 250 | Yes | TC-BK-001 |
| Positive decimal | 250.50 | Yes | manual |
| Numeric string | "250" | No — type violation | TC-NG-010 |
| Non-numeric | "abc" | No | manual |
| Absent | — | No | TC-NG-011 |

Seven partitions, one representative each. Trying twenty different positive amounts adds
nothing, because they all sit in the same partition and exercise the same code path.

## 2. Boundaries — dates

```
   checkout < checkin  │  checkout = checkin  │  checkout > checkin
   ───── invalid ──────┼──── edge case ───────┼────── valid ───────
       TC-NG-004                manual              TC-BK-001
```

`checkout = checkin` is the interesting one — a zero-night booking. Either it is valid
(a day-use booking) or it is not, and the API should be consistent about which. Cases
that sit exactly on a boundary are where the `<` versus `<=` mistake lives.

## 3. Authentication state model

```
   [No token] ──POST /auth (valid)──▶ [Token held] ──token expires──▶ [Stale token]
        │                                   │                              │
   PUT → 403                          PUT → 200                       PUT → 403
   TC-BK-003                          TC-BK-004                       TC-BK-009
```

Three states, three write attempts. The one people skip is the third — a stale token
often behaves differently from no token at all, and on a session-security review that
distinction is what gets asked about.

## 4. What a status code alone cannot tell you

The reason several cases assert on the body rather than the code:

| Case | Status returned | What the body reveals |
|---|---|---|
| TC-AU-002 | 200 | No token — the login actually failed |
| TC-NG-003 | 200 | The negative amount was stored |
| TC-BK-005 | 200 | Whether the untouched fields survived |
| TC-BK-002 | 200 | Whether the write persisted what was sent |

Four cases where a suite asserting `200` would report green while the API did something
wrong. This is the argument for writing assertions against the response body as a habit
rather than as an exception.

## 5. Out of scope, and why

- **Load and rate limiting.** Shared public host on a free tier; testing this would
  degrade the service for everyone else using it.
- **Contract testing against a schema.** Worth doing on a real API and the natural next
  step for this repository. Noted rather than pretended.
- **The SOAP interface.** Not exposed by this API.
