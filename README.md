# API Test Design — restful-booker

Manual and semi-automated API test design for
[restful-booker](https://restful-booker.herokuapp.com), a public practice API with
published documentation at [restful-booker.herokuapp.com/apidoc](https://restful-booker.herokuapp.com/apidoc/index.html).

Postman and Swagger are what I use daily on Player Account Management, wallet and
payment-gateway endpoints. This repository is that work, rebuilt against a public API so
it can be shared.

No employer systems, credentials or data appear anywhere in this repository.

## Contents

| File | What it is |
|---|---|
| [`test-cases/api-test-cases.md`](test-cases/api-test-cases.md) | 23 cases with request, expected response and the technique that produced each |
| [`test-cases/test-conditions.md`](test-cases/test-conditions.md) | The analysis — equivalence partitions, boundaries, the auth state model |
| [`test-cases/findings.md`](test-cases/findings.md) | Defects and observations found while executing |
| [`postman/restful-booker.postman_collection.json`](postman/restful-booker.postman_collection.json) | Importable collection, 23 requests in four folders, with assertions |
| [`postman/restful-booker.postman_environment.json`](postman/restful-booker.postman_environment.json) | Environment file — base URL, credentials, chained variables |

## Run it

Import both JSON files into Postman, select the environment, then run the collection
folders in order — folder 1 stores the auth token in a collection variable and folder 2
chains the booking id through create, read, update, patch and delete.

Run **TC-NF-001 (health check)** first. This is a free-tier host that sleeps, and a
suite of red results because the service is cold is not a set of defects. One request
tells you which situation you're in.

## How the collection is organised

By **test type**, not by endpoint. Four folders: authentication, lifecycle, boundary and
negative, then listing and non-functional.

Organising by endpoint feels natural and is less useful. When a run comes back with
failures, folder-by-type tells you immediately whether it's an auth problem, a data
problem or a validation problem. Folder-by-endpoint tells you which URL was involved,
which you already knew from the request name.

## What the assertions actually check

Most Postman collections assert a 200 and stop. These check the things that matter and
would otherwise pass silently:

**Absence, not just presence.** TC-AU-002 sends a wrong password. This API answers
`200` with a `reason` field rather than `401`, so a client checking only the status code
would treat a failed login as a success. The assertion is that no token was issued.

**Persistence, not the echo.** TC-BK-001 creates a booking and TC-BK-002 reads it back
in a separate request. A create response that echoes your own payload tells you nothing
about what was stored — I have seen a field accepted, echoed, and then silently dropped
on write.

**Authorisation separately from validation.** TC-BK-003 attempts an update with no
token and expects `403`. Field validation and access control are different checks and a
suite that only exercises the authenticated path never tests the second one.

**Idempotency.** TC-BK-008 deletes an already-deleted booking. A second delete reporting
success means the caller cannot distinguish "removed" from "was never there", which
matters as soon as anything retries.

## Findings, and why two tests assert the wrong behaviour

TC-NG-003 asserts that the API stores a **negative** total price. TC-NG-004 asserts that
it accepts a checkout date **before** the checkin date.

Neither is correct behaviour. Both are pinned deliberately.

A test that fails because the system under test is wrong is not a regression test — it's
a defect report that runs on every execution and teaches the team to ignore red results.
So: pin the current behaviour, name the finding in a comment and in
[`findings.md`](test-cases/findings.md), and raise it separately. When someone fixes it,
the test fails, and that failure means "this changed" — which is what a regression test
is for.

On a payments API, no server-side validation on a monetary amount is a high-severity
defect. Recording it properly is the job; making the suite red forever is not.

## Author

Naga Yashila Araveti — QA Engineer. REST API and database testing on regulated payment
and gaming platforms. [LinkedIn](https://www.linkedin.com/in/naga-araveti)
