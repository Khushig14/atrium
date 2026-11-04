# Week 6 evidence record

Your name: Khushi Ganatra
Date: 23/10/2026
Atrium version or date of local copy: 23/10/2026

Keep this file in `my-work/week-06-write-up.md`. Record what you observed, including an unexpected result. Do not paste a password or full session cookie.

## 1. Evidence audit

| Item | Scanner claim, observed evidence, background or missing evidence? | Where I can find it |
|---|---|---|
| Static alert and rule | Semgrep reported an error because the SQL query is built by joining pieces together using +. It detects the joining of text, but does not check where the input comes from. | Run npm run scan - S1 at src/handlers/directory.js:8. |
| Static code excerpt | Looking at the code, the search input q comes from the URL at line 19. It is then added to the SQL query at lines 10–11, which runs at line 13.	| src/handlers/directory.js, lines 8–13 and 19. |
| Dynamic alert and risk label | ZAP reported a possible reflected XSS issue (Z2) with High risk and Medium confidence on GET /directory?q=. This means the scanner found a possible issue, but it does not prove the website is vulnerable. My own ZAP scans (8 alerts and 20 alerts) did not find SQL Injection or XSS. They only found basic security header and cookie issues. | The Z2 label comes from the class recorded run (the brief’s queue). My own saved ZAP results do not contain it — they show only the header and cookie alerts. |
| Dynamic request and response | I have not yet captured the request and response showing the possible XSS issue. This evidence is still missing. | Next step: Log in as Alice, open DevTools - Network, and visit GET /directory?q=<b>zz</b>. Record the response status and the HTML showing the search results. |
| Signed-in coverage check | Both of my ZAP scans were done without logging in and found no High-risk alerts. This matches the brief, which says Z1 and Z2 may only appear when logged in. The authentication code also sends signed-out users to the login page. | My two ZAP alert lists and src/middleware/auth.js:11. A signed-out request to /directory returns 302 and redirects to /login. |
| Ordinary comparison | I have not yet captured a normal search to compare with the possible XSS result. The search for Nolan will be used as the normal example. | Next step: Log in as Alice, open DevTools - Network, and visit GET /directory?q=Nolan. Record the 200 status and the message saying "Showing colleagues matching…". |

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

- **Ordinary request and result:** GET /directory?q=Nolan - status 200 OK (2,856 bytes) - the result summary readsShowing colleagues matching <strong>Nolan</strong>. and Alice Nolan's row appears in the results table.

- **Test request and result:** GET /directory?q=Nolan’(sent asq=Nolan%27) - status 500 Internal Server Error (1,145 bytes) → the response is plain text that begins with the SQLite error message [paste the first line you see, e.g. SqliteError: near “%”: syntax error] followed by a stack trace. Difference from ordinary: the normal 200 result list is replaced by a 500 error, consistent with the apostrophe closing the string literal on line 10 and breaking the SQL.

- **My conclusion:** Established: a signed-in user's input is concatenated into SQL text (lines 10–11) with no binding, and [if confirmed] a single apostrophe turns a normal result into a 500, consistent with the quote breaking the statement. Untested: whether the query can be steered to return data beyond the directory (other columns or tables, e.g. a UNION), and what the database error discloses.

- **Impact within this scope:** Any signed-in colleague (member-level, no admin) can change the structure of the directory query. The same database holds the `users` table, which includes a `password_hash` column (`src/db/schema.sql:15`, populated in `src/db/reset.js:12`). If the injection can be extended (untested), those hashes and other user fields could become readable. This is a possible, not an observed, impact.

- **Repair and how I would check it:** Replace the concatenation with bound parameters — `WHERE display_name LIKE ? OR department LIKE ?`, passing `'%' + q + '%'` twice through `db.prepare(sql).all(...)`, the pattern `resources.js` already uses (`WHERE resources.id = ?`, bound at line 18). Check: (1) repeat `q=Nolan'` → expect a normal 200 with no matches, not a 500; (2) repeat `q=Nolan` → result unchanged; (3) re-run `npm run scan` → S1 no longer matches that line.

- **One other static match:** **S2**, `src/handlers/resources.js:1`. The same rule fires because `RESOURCE_QUERY` is assembled with `+`, but every concatenated piece is fixed text, and the only external value — `req.params.id` — is passed as a bound parameter (`WHERE resources.id = ?` at line 5; `db.prepare(RESOURCE_QUERY).get(req.params.id)` at line 18). The rule matches concatenation *syntax*, not *data flow*, so it cannot settle the status — reading the source does. Status: ruled out. (S3, `src/db/reset.js:12`, is the same pattern: fixed SQL joined, seed values bound with `?`.)

## 3. Dynamic finding

**Title:** Reflected XSS in the directory search results. The search input q is added to the page without being safely escaped.

**Affected URL and output location:** GET /directory?q=. The search value appears in two places. The result summary at line 20 uses <%- q %>, which does not escape the input and causes the issue. The search box at line 13 uses <%= q %>, which escapes the input and is safe. This finding is about the result summary, not the search box.

**Account and conditions tested:** I tested the issue while logged in as Alice Nolan (member) on the local Atrium application. I reset the application before testing and used Chrome with DevTools to check the requests and responses. ZAP 2.17.0 was used for the scanner reference.

**ZAP alert, risk and confidence:** ZAP reported a Reflected XSS issue with High risk and Medium confidence on GET /directory?q=. My own ZAP scans did not find this issue because I ran them without logging in. I therefore checked the issue manually in the browser instead.

**Ordinary request and result:** GET /directory?q=Nolan(logged in as Alice) - status 200 OK - the result summary showedShowing colleagues matching <strong>Nolan</strong>. The browser displayed "Showing colleagues matching Nolan" with Nolan in bold and Alice Nolan's row in the table.

**Alert request repeated and result:** GET /directory?q=<b>zz</b>(sent URL-encoded asq=%3Cb%3Ezz%3C%2Fb%3E, logged in as Alice) → status 200 OK (2,256 bytes) - the result summary in the response contained the raw tags Showing colleagues matching <strong><b>zz</b></strong>.— not escaped — and no colleague matched, so the "No colleagues found" state showed below it. The browser displayed **zz in bold** in the summary, while the search box showed the literal text<b>zz</b> because line 13 escapes it.

**Code that explains the result:** views/directory/index.ejs:20 uses <strong><%- q %></strong>. The <%- tag adds the input directly into the HTML without escaping it. This is what causes the vulnerability. Line 13 uses value="<%= q %>", which escapes the input and makes the search box safe. The handler passes q to the page without changing it (src/handlers/directory.js:19, 23).

**My conclusion:** The result summary can display user input as HTML because it does not escape the value. The search box handles the same input safely by escaping it. I used a simple <b>zz</b> test to check this rather than running an actual script. Full JavaScript execution has not been tested, so I would not claim that part is confirmed yet.

**Impact within this scope:** If a signed-in colleague opens a specially created directory link, an attacker could potentially run their own JavaScript in that colleague's session. The user must be signed in because the directory page requires authentication. The application also does not have a Content-Security-Policy header, which could provide extra protection against injected scripts. However, full script execution has not been tested.

**Repair and how I would check it:** Change line 20 from <%- q %> to <%= q %>. This will make the result summary escape the search input, just like the search box already does. I would also recommend adding a Content-Security-Policy header for extra protection. After making the change, I would repeat both searches. The <b>zz</b> input should appear as normal text instead of making zz bold, while the normal Nolan search should continue working as before.

## 4. Cross-check and review

What did runtime evidence add to the static finding? 
The scanner and the code only showed that the search value is joined into the SQL text. They do not prove the running app actually breaks. When I searched for Nolan' (with one apostrophe), the page returned a 500 error instead of a normal result. That showed the live app really does reach that line, and the apostrophe ends the quote and breaks the query. The normal search for Nolan returned 200, so I had something to compare it against. The runtime check also showed who can reach it: when signed out, /directory just sends you to the login page, so this is something only a signed-in user can do.

What did code evidence add to the dynamic finding?
ZAP only said the search value showed up in the response. The code showed why and where. Line 20 prints the value raw with <%- , so the HTML is not escaped there. Line 13 prints it with <%= , which escapes it, so the search box is safe. So the code told me Z2 and Z3 are two different places in the same page, not one problem counted twice. It turned "the value appeared" into a clear reason.

Who or what reviewed my drafts?
I used an AI assistant (Github Copilot) to help set out the record and to check my reasoning. I read the source files myself, ran the browser checks myself, and edited the wording so it is in my own words.

One specific criticism:
My first impact note said the password hashes could be extracted. I had not actually tested that, so it was a claim without evidence.

What I changed, or why I did not change it:
I rewrote the impact so it says the hashes might be readable only if the injection can be extended, and that this part is untested. I kept the proven part (the input is joined into the SQL, and the apostrophe breaks the query) separate from the part I am not sure about.

## 5. My report draft

Finding reviewed: The static finding — SQL injection in the directory search (S1, with Z1 as runtime support).

| Check | Present and where, or missing and next action |
|---|---|
| Plain title and exact location | Present. Title in Section 2, and the exact spot: GET /directory?q= at src/handlers/directory.js lines 10–11, which runs at line 13. |
| Account, route and conditions | Present. alice.nolan signed in, local Atrium after npm run reset, and the route is behind requireAuth so a signed-in user can reach it. |
| Normal and tested result | Missing for now. Next action: run q=Nolan and q=Nolan' in the browser and record the status and a short excerpt. Once added, the normal result is 200 and the apostrophe result is the error I saw. |
| Code or HTTP artefact supporting each claim | Code is present (directory.js lines 10–13 and 19, and resources.js lines 5 and 18 for the contrast). The HTTP artefact (the apostrophe request and its response) still needs to be captured and pasted in. |
| Impact limited to the evidence | Present. I wrote that reading the password hashes is only possible if the injection can be extended, and I marked that part untested. Only the concatenation and the broken query are stated as proven. |
| Repair that addresses the cause | Present. Use bound parameters (the ? placeholder) the way resources.js already does, with three checks to confirm it worked. |

The next action I will take: Sign in as Alice, open DevTools - Network, and run GET /directory?q=Nolan and GET /directory?q=Nolan'. Record the status and a short piece of each response, then paste them into Section 2 so the "normal and tested result" row rests on my own evidence instead of a gap.