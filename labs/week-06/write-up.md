# Week 6 evidence record

Your name: Khushi Ganatra
Date: 23/10/2026
Atrium version or date of local copy: 23/10/2026

Keep this file in `my-work/week-06-write-up.md`. Record what you observed,
including an unexpected result. Do not paste a password or full session cookie.

## 1. Evidence audit

| Item | Scanner claim, observed evidence, background or missing evidence? | Where I can find it |
|---|---|---|
| Static alert and rule | Semgrep reported an error because the SQL query is built by joining pieces together using +. It detects the joining of text, but does not check where the input comes from. | Run npm run scan → S1 at src/handlers/directory.js:8. |
| Static code excerpt | Looking at the code, the search input q comes from the URL at line 19. It is then added to the SQL query at lines 10–11, which runs at line 13.	| src/handlers/directory.js, lines 8–13 and 19. |
| Dynamic alert and risk label | ZAP reported a possible reflected XSS issue (Z2) with High risk and Medium confidence on GET /directory?q=. This means the scanner found a possible issue, but it does not prove the website is vulnerable. My own ZAP scans (8 alerts and 20 alerts) did not find SQL Injection or XSS. They only found basic security header and cookie issues. | The Z2 label comes from the class recorded run (the brief’s queue). My own saved ZAP results do not contain it — they show only the header and cookie alerts. |
| Dynamic request and response | I have not yet captured the request and response showing the possible XSS issue. This evidence is still missing. | Next step: Log in as Alice, open DevTools - Network, and visit GET /directory?q=<b>zz</b>. Record the response status and the HTML showing the search results. |
| Signed-in coverage check | Both of my ZAP scans were done without logging in and found no High-risk alerts. This matches the brief, which says Z1 and Z2 may only appear when logged in. The authentication code also sends signed-out users to the login page. | My two ZAP alert lists and src/middleware/auth.js:11. A signed-out request to /directory returns 302 and redirects to /login. |
| Ordinary comparison | I have not yet captured a normal search to compare with the possible XSS result. The search for Nolan will be used as the normal example. | Next step: Log in as Alice, open DevTools → Network, and visit GET /directory?q=Nolan. Record the 200 status and the message saying "Showing colleagues matching…". |

What is the most important missing artefact for each alert?

- Static: a runtime request/response proving a changed `q` changes the query's behaviour — the apostrophe test (GET /directory?q=Nolan' → HTTP 500, matching Z1). The code (src/handlers/directory.js:10–11) shows the unparameterised sink; the 500 would show the running app actually reaches it.
- Dynamic: an observed request/response showing where the value lands unescaped. The code already pinpoints it — the result summary prints `q` raw (views/directory/index.ejs:20, `<%- q %>`) while the search field escapes it (line 13, `<%= q %>`) — so the missing piece is a browser repeat confirming a payload renders in the summary, not the field.

## 2. Static finding

**Title:** SQL injection in the directory search — the signed-in user's `q` value is concatenated straight into the SQL statement.

**Affected route, file and lines:** `GET /directory?q=`, handled in `src/handlers/directory.js`. Source at line 19; sink at lines 10–11 (inside `searchDirectory`, `sql` assembled lines 8–11); executed at line 13. Route wired with `requireAuth` in `src/routes/directory.js:8`.

**Account and conditions tested:** alice.nolan (ordinary member, non-admin), signed in. Local Atrium at http://localhost:9090 after `npm run reset`. The route is behind `requireAuth`, so any authenticated colleague can reach it; no admin rights needed.

**Tool output used:** Semgrep **S1** — "SQL text built by joining pieces", ERROR, at `src/handlers/directory.js:8`. Supporting: class-recorded ZAP **Z1** (SQL Injection, High risk, Low confidence; a changed search returned HTTP 500). My own ZAP runs did not reproduce Z1 (they were unauthenticated).

**Code trace:**
- *Source:* `q` enters from the query string at `directory.js:19` — `const q = typeof req.query.q === "string" ? req.query.q : ""`. A signed-in user controls it fully; the only transform coerces a non-string to `""`, which does not neutralise SQL metacharacters.
- *Use / sink:* when `q` is non-empty, `showDirectory` calls `searchDirectory(db, q)` at line 21. `searchDirectory` builds `sql` at lines 8–11, concatenating `q` directly into the string — line 10 `"WHERE display_name LIKE '%" + q + "%' " +` and line 11 `"OR department LIKE '%" + q + "%'"` (`q` is used twice) — then runs it at line 13 `db.prepare(sql).all()`.
- *Control on the path:* only `requireAuth` (sign-in). No validation, escaping, or parameter binding between source and sink, unlike `resources.js`, which uses a `?` placeholder.

**Ordinary request and result:** `GET /directory?q=Nolan` → [RECORD status — expected 200] → [RECORD excerpt — expected: "Showing colleagues matching Nolan" and Alice Nolan's row].

**Test request and result:** `GET /directory?q=Nolan'` (one apostrophe appended) → [RECORD status — expected 500] → [RECORD short excerpt — expected: a server-error page]. Difference from ordinary: [expected: the normal 200 result list becomes a 500 error, consistent with the apostrophe closing the string literal on line 10 and breaking the SQL].

**My conclusion:** Established: a signed-in user's input is concatenated into SQL text (lines 10–11) with no binding, and [if confirmed] a single apostrophe turns a normal result into a 500, consistent with the quote breaking the statement. Untested: whether the query can be steered to return data beyond the directory (other columns or tables, e.g. a UNION), and what the database error discloses.

**Impact within this scope:** Any signed-in colleague (member-level, no admin) can change the structure of the directory query. The same database holds the `users` table, which includes a `password_hash` column (`src/db/schema.sql:15`, populated in `src/db/reset.js:12`). If the injection can be extended (untested), those hashes and other user fields could become readable. This is a possible, not an observed, impact.

**Repair and how I would check it:** Replace the concatenation with bound parameters — `WHERE display_name LIKE ? OR department LIKE ?`, passing `'%' + q + '%'` twice through `db.prepare(sql).all(...)`, the pattern `resources.js` already uses (`WHERE resources.id = ?`, bound at line 18). Check: (1) repeat `q=Nolan'` → expect a normal 200 with no matches, not a 500; (2) repeat `q=Nolan` → result unchanged; (3) re-run `npm run scan` → S1 no longer matches that line.

**One other static match:** **S2**, `src/handlers/resources.js:1`. The same rule fires because `RESOURCE_QUERY` is assembled with `+`, but every concatenated piece is fixed text, and the only external value — `req.params.id` — is passed as a bound parameter (`WHERE resources.id = ?` at line 5; `db.prepare(RESOURCE_QUERY).get(req.params.id)` at line 18). The rule matches concatenation *syntax*, not *data flow*, so it cannot settle the status — reading the source does. Status: ruled out. (S3, `src/db/reset.js:12`, is the same pattern: fixed SQL joined, seed values bound with `?`.)

## 3. Dynamic finding

**Title:** 

**Affected route, file and lines:** 

**Account and conditions tested:** 

**ZAP alert, risk and confidence:** 

**Ordinary request and result:** 

**Alert request repeated and result:** 

**Code that explains the result:** 

**My conclusion:** 

**Impact within this scope:** 

**Repair and how I would check it:** 

## 4. Cross-check and review

What did runtime evidence add to the static finding?

What did code evidence add to the dynamic finding?

Who or what reviewed my drafts?

One specific criticism:

What I changed, or why I did not change it:

## 5. My report draft

Finding reviewed:

| Check | Present and where, or missing and next action |
|---|---|
| Plain title and exact location | |
| Account, route and conditions | |
| Normal and tested result | |
| Code or HTTP artefact supporting each claim | |
| Impact limited to the evidence | |
| Repair that addresses the cause | |

The next action I will take:
