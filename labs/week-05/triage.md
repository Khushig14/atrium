# Week 5 cross-tool triage

Your name: Khushi Ganatra
Date: 14/10/2026
Atrium address and port: http://localhost:9090
Atrium version or date of local copy: 14/10/2026

Fill this in as you work. If a check could not be completed, record what stopped you and what evidence you used instead. Do not paste a full session cookie value.

## 1. Scan coverage and sources

- Static evidence: my `npm run scan` output: It reported 3 code findings, all rule scan-rules.sql-built-by-joining-text, severity Blocking: src/handlers/directory.js lines 8-11 (S1), src/handlers/resources.js lines 1-5 (S2) and src/db/reset.js lines 12-13 (S3).
- Runtime evidence: my ZAP run, Week 4 record, or direct browser checks: My Week 4 ZAP session (ZAP 2.17.0, Edge as the ZAP-launched browser, signed in as alice.nolan, 14/10/2026), including an active scan of GET:directory(q), plus direct browser checks in Edge DevTools (Network tab, cache disabled).
- Signed-in account used for protected routes: alice.nolan (seeded staff account, password from the Atrium README file).
- Evidence that a protected route returned its page rather than `/login`: In my Week 4 signed-in ZAP session, GET /dashboard returned 200 OK, 3,314 bytes (History row 239), and GET /directory?q=Alice returned 200 OK, 2,856 bytes (row 259), with no redirect to /login. In Edge DevTools (Network, cache disabled) the directory response was 200 OK with Content-Length 2856, matching the ZAP row, and the page showed Alice Nolan's entry.
- Active scan target and how login and logout stayed outside it, if I used ZAP: I active scanned only GET:directory(q) (the search request with the query parameter), with my Context and User Alice. I excluded http://localhost:9090/logout.* in the Context settings, and I did not include GET /login, POST /login or POST /logout in the scan. Later directory responses in History were still 200 pages, so the scan stayed signed in.
- An entry my scan did not reproduce, and how I checked it instead: Z5 was not reproduced as queued. My active scan kept POST /login outside it, so the entry was not tested from that request. I checked it directly instead: in the signed-in Edge browser I signed in as alice.nolan with DevTools open (Network, Preserve log, cache disabled), selected the POST /login response, and read the Set-Cookie attributes (HttpOnly, Secure, SameSite, Path). I recorded only the attribute names, not the cookie value.

## 2. The queue

For each entry, use **supported**, **ruled out**, or **needs review**. Keep overlap separate from status. Name a file and line, or a request and response, in the evidence column. If evidence is missing, say what check would settle it.

| ID | Status | Evidence or missing check | Overlap with another ID? | Investigate now, later, or close? |
|---|---|---|---|---|
| S1 | Supported | In `src/handlers/directory.js`, line 19 reads `req.query.q` and only checks that it is a string. Lines 8-11 join `q` into the SQL text twice, and line 13 runs it with `db.prepare(sql).all()` and no bound parameters, so outside input becomes part of the SQL command. I haven't yet shown that a real request can change what the query does; Z1 covers that. | Same route and input as Z1 (the SQL side). Z2 and Z3 use the same `q` but write it into the HTML instead. | Now |
| S2 | Ruled out | In `src/handlers/resources.js` lines 1-5, `RESOURCE_QUERY` is built only from fixed strings. Line 18 runs `db.prepare(RESOURCE_QUERY).get(req.params.id)`, so the id fills the `?` on line 5 as a value and never becomes SQL text. The rule matched the `+` joining, not any outside input. | Same rule and pattern as S1 and S3, but a different path: S1 joins outside input into the SQL, S2 doesn't. | Close |
| S3 | Ruled out | In `src/db/reset.js` lines 12-13, `INSERT_USER` is built only from fixed strings with eight `?` placeholders. Lines 30-41 run it with `addUser.run(...)`, so every value is bound. The data comes from `./seed-data.js` inside Atrium and the function only takes a database path, so nothing from a request reaches it. | Same rule and joining pattern as S1 and S2, but no outside input is involved. | Close |
| Z1 | Supported | Signed in as alice.nolan, `GET /directory?q=Alice` returned 200 OK (2,856 bytes, one result). `GET /directory?q=Alice%27` returned 500 with `SqliteError: near "'%'": syntax error`, raised in `searchDirectory` at `directory.js:13` (ZAP History row 132). So the search value changes the SQL command's syntax. I haven't shown that data can be read or changed. | Same path as S1. Same route and input as Z2 and Z3, but a different effect (SQL, not HTML). | Now |
| Z2 | Supported | ZAP's active scan raised Cross Site Scripting (Reflected), High risk, Medium confidence, parameter `q`, with attack and evidence `</strong><scrIpt>alert(1);</scRipt><strong>`. When I repeated that request myself in the Requester as alice.nolan, it returned 200 OK and the string appeared as raw markup in `<p class="result-summary">`. I haven't shown that a script actually runs in a browser. | Same route and input as Z1 and Z3, but a different sink. Z3 is the `value` attribute of the same response, which is encoded. | Now |
| Z3 | Ruled out | ZAP History row 500: `GET /directory?q=Bob%22x` as alice.nolan returned 200 OK. The search field shows `value="Bob&#34;x"`, so the quote is encoded and can't close the attribute. My Week 4 payload also showed `&lt;` and `&gt;` in the same attribute. ZAP's alert is only a passive note (Informational, Low) that user input appears in an attribute. I used `q=Bob%22x`, not the queue's `q=Bob%20Keane`. I haven't tried other characters or encodings. | Same route, input and response as Z1 and Z2. Z2 is the raw use in `result-summary`; Z3 is the encoded use in the `value` attribute. | Close |
| Z4 | Supported | ZAP History row 505: `GET /dashboard` as alice.nolan returned 200 OK (3,314 bytes) and the response headers contain no `Content-Security-Policy`. I haven't shown that a CSP would have blocked anything, or that there is an injection on `/dashboard` itself. ZAP's Medium risk, High confidence only describes the missing header. | Loosely related to Z2, since a CSP could limit the damage from reflected script, but it is a different place and weakness. It doesn't show Z2 is exploitable. | Later |
| Z5 | Supported | In Edge DevTools (Preserve log on, cache disabled), the `POST /login` response as alice.nolan sets `connect.sid` with only `Path=/`: no HttpOnly, Secure or SameSite. I haven't shown that a script could read the cookie; that would need a working injection such as Z2 plus a `document.cookie` check. Still to do: compare with `src/config/session.js`. | Only linked to Z2 as a possible chain (script on the page could read a cookie without HttpOnly). Different place and weakness. | Later |

## 3. First three matters to investigate

Rank the matters. Combine entries only when they concern one underlying path. For each, record the relevant account, route or file, possible consequence, uncertainty and next check.

1. **First:**
- Queue ID or IDs: S1 + Z1
- Account and place: alice.nolan, `GET /directory?q=` (code at `src/handlers/directory.js:8-11,19`)
- Why first: this is the only entry where I've shown the search value changes what the SQL command does, not just how it's displayed. A query that can be broken can usually also be changed, so this could let someone read or alter data beyond their own, not just this one page. The code (S1) and the runtime check (Z1, the 500 error) point to the same cause, so combining them is one path, not two findings.
- Next check: try a payload that produces a valid altered query instead of an error, for example a UNION-based or boolean-based value, to see whether it returns rows it shouldn't. This needs care and should be done on my own local copy only.

2. **Second:**
- Queue ID or IDs: Z2
- Account and place: alice.nolan, `GET /directory?q=` (the `result-summary` output)
- Why second: the search value is reflected in the page as raw HTML, confirmed by both ZAP's active scan and my own repeat. If this runs as a script in a real browser, it could act on behalf of whoever opens a crafted link, which is a different kind of risk from Z1 but just as serious. I haven't yet seen a script actually execute.
- Next check: open the alert's exact URL in a normal signed-in browser tab (not just inspect the response) and note whether a script actually runs, for example an alert box, without doing anything beyond that.

3. **Third:**
- Queue ID or IDs: Z5
- Account and place: alice.nolan, `POST /login` (the `Set-Cookie` header)
- Why third: on its own this is a smaller issue, since it needs another weakness to matter. But if Z2 does turn out to run a script, a cookie with no HttpOnly flag means that script could read the session cookie, so this could turn Z2 into full account takeover rather than just a page defacement. Worth checking now while Z2 is still open, rather than treating it as unrelated.
- Next check: once Z2's next check is done, if a script does run, confirm (without exploiting it) whether `document.cookie` is readable from the page, and compare with `src/config/session.js` to see how the cookie is configured.

## 4. One exclusion or combination

- Entry ID: S2
- Close, exclude from active work, or combine with another entry? Close
- Evidence for this decision: `src/handlers/resources.js:1-5` builds `RESOURCE_QUERY` only from fixed string literals, with a `?` placeholder for the id. Line 18 runs it as `db.prepare(RESOURCE_QUERY).get(req.params.id)`, so the id is bound as a parameter, never joined into the SQL text. The scan rule matched the `+` joining pattern, not any outside input reaching the query. I'm not closing it because of its severity label; the rule marked it the same Blocking severity as S1.
- What change would make me reopen it: if a future change joins `req.params.id`, or any other request value, directly into `RESOURCE_QUERY` with `+` instead of passing it to `.get()`, or if the query is ever built dynamically based on request input.


## 5. One static and runtime comparison

- SSemgrep entry and ZAP entry: S1 and Z1
- What the code observation shows: in `directory.js:19`, the search value is read from the request with only a check that it's a string, nothing more. Lines 8-11 then paste that value straight into the SQL text, twice, and line 13 runs it with no safety parameter. So the code shows exactly where and how an unchecked value ends up inside a SQL command.
- What the request and response show: when I actually searched for `Alice'` as alice.nolan, the page broke with a 500 error and a database syntax error. That proves this isn't just a theoretical problem in the code; a real request through the real app reaches it and breaks the query.
- Do they concern the same underlying matter? Why? Yes. Same input, same file and line, same database call. The code explains why it breaks, and the request proves it does break when you actually try it.
- One claim they do not yet prove: that someone could actually read or change data they shouldn't be able to see. All I've shown is that the query breaks, not that it can be bent into something useful to an attacker.

## 6. One finding for a colleague

Write a short paragraph. Include what and where, the account and condition, possible impact, evidence references and a repair direction. Keep untested impact conditional.

**Finding:**
The staff directory search (`GET /directory?q=`) is vulnerable to SQL injection. In `src/handlers/directory.js`, the search value is read from the request and pasted directly into the SQL query, with no checks. As alice.nolan, searching `Alice` works fine, but searching `Alice'` breaks the page with a 500 error and a database syntax error. This shows the search value is reaching the database as part of the command, not just as data. I haven't checked whether this could be used to read or change other data, but the way the query is built makes that possible. The fix is to use parameterised queries, as `resources.js` already does, instead of building the SQL string by hand.


**Evidence references:**
- Code: `src/handlers/directory.js:8-11`, `src/handlers/directory.js:19`
- Request/response: `GET /directory?q=Alice` - 200 OK; `GET /directory?q=Alice%27` - 500, `SqliteError: near "'%'": syntax error` (ZAP History row 132)

---

## Optional, if you had time

What does `src/config/session.js` show about the session that the Z5 response cannot? What does the response confirm that source inspection alone cannot?
It shows that `httpOnly`, `sameSite` and `secure` are each explicitly set to `false`, not just left unset. So this isn't a missing default; someone deliberately configured the cookie this way. The code also shows it applies to every session the app creates, not just the one I happened to check.
The Z5 response confirms the setting actually takes effect at runtime: the `Set-Cookie` header I captured really does carry no HttpOnly, Secure or SameSite attributes, exactly as the code says it should. Reading the source alone wouldn't rule out some other middleware overriding this configuration before the cookie is sent; the response shows that didn't happen.