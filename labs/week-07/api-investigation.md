# Week 7 API investigation

Your name: Khushi Ganatra
Date: 29/10/2026
Atrium address and port: http://localhost:9090
Atrium version or date of local copy: 29/10/2026

Fill this in as you work. Give statuses and short response excerpts. Record
whether a request carried a session cookie, but never paste its full value
or a password.

## 1. Original profile request

- Signed-in account: alice.nolan (Alice Nolan, id 17, role member)
- Page that made the request: My profile page at http://localhost:9090/profile. The request is sent by the page's own script (public/js/profile.js), which fetches the profile to refresh the details after the page loads.
- Method and full API path: GET http://localhost:9090/api/profile/17
- Was a session cookie present? (yes/no, value hidden): Yes (value hidden). The script calls fetch with credentials: "same-origin", so the browser sends Alice's session cookie.
- First response status and content type: 200 OK, Content-Type: application/json (captured with Disable cache on; without it the browser revalidates with the ETag and returns 304 Not Modified).
- Returned ID, username and field names: id 17, username alice.nolan, displayName "Alice Nolan". Field names present: id, username, displayName, email, department, role, bio.
- What did the request ask for, in your own words? It asks the server to return the profile record for the user whose id is 17, which is Alice's own account, as JSON. The id number in the path (17) is the part that identifies which record comes back, and the page uses the response to refresh the name, email and department.

## 2. Replayed and changed profile requests

| Request | Account whose session I used | First status | Returned ID and name, or error | Evidence reference |
|---|---|---|---|---|
| Repeat of the original path | alice.nolan (Alice, id 17), signed in | 200 OK | id 17, Alice Nolan (alice.nolan) | DevTools Network, GET /api/profile/17 |
| Path with one ID changed | alice.nolan (Alice, id 17), signed in | 200 OK | id 18, Bob Keane (bob.keane), email bob.keane@atrium.example, department Finance, role member, plus bio | DevTools Response, GET /api/profile/18 |

- What I changed: Only the id at the end of the path, from 17 to 18. I also sent it from the address bar instead of the profile page, but it is the same GET request to the same route.
- What stayed the same: The signed-in account and its session cookie (still Alice), the method (GET), and the route /api/profile/:id.
- Whose record the changed ID names, and how I checked: id 18 is Bob Keane. I checked against the seeded account list in README.md and the staff directory at /directory, and the response's own fields (username bob.keane, displayName Bob Keane) confirm it.
- What the comparison shows: While signed in as Alice, I received Bob's full profile just by changing the id. The server checks that the caller is signed in (authentication) but not that the record belongs to the caller (authorisation between users). This is an insecure direct object reference — the :id has no ownership check. Reading every account by walking through ids is likely but untested; I only requested 18.

## 3. Without sign-in

- How I made sure the window had no Atrium session: I opened a new InPrivate window, which does not share cookies with the Alice window. It showed the signed-out state and had no Atrium session before I visited anything.
- `GET /api/profile/17`: first status and response body: 401 Unauthorized - {"error":"You need to sign in to use this part of Atrium."}
- `GET /profile`: first status, redirect if any, and final page: 302 redirect to /login; the browser finished on the Atrium sign-in page (localhost:9090/login).
- What this establishes about authentication: Both the profile API and the profile page require a signed-in session. Without one, the API returns 401 and the page redirects to /login. The sign-in boundary (authentication) is enforced on these routes.
- What it does not establish about authorisation: It says nothing about what a signed-in user is allowed to read. Part 2 showed that once signed in, Alice (a member) could read Bob's record by changing the id. So enforcing "are you signed in?" does not mean the server enforces "are you allowed this particular record?" The 401 only proves the sign-in gate, not per-record permission.

## 4. Administration page and API

| Account | Request path | First status | Relevant response text or JSON fields | Evidence reference |
|---|---|---|---|---|
| Alice | `/admin/users` | 403 | plain text "Administrators only." (page refuses a member) | screenshot |
| Alice | `/api/admin/users` | 200 | JSON array of all three accounts incl. Morgan (`role:admin`); fields id, displayName, username, department, role | screenshot |
| Morgan | `/admin/users` | 200 | Staff accounts page renders; same three accounts | screenshot |
| Morgan | `/api/admin/users` | 200 | same JSON array (the call the admin page makes on load) | Fetch/XHR on page load |

- Which request did Morgan's page make after loading? GET /api/admin/users — the page script (public/js/admin-users.js) fetches it to fill the staff table with id, displayName, username, department and role.
- What role decision should the server make for each route? Both routes expose the same administrator-only staff list, so both should require the admin role (requireAdmin). The API currently requires only sign-in (requireAuth).
- Where did the observed decisions agree or differ? They agree for Morgan (admin, 200 from both page and API) and the page is consistent (403 for Alice, 200 for Morgan). They differ for Alice at the two entry points to the same data: the page returned 403 (requireAdmin), but the API returned 200 (requireAuth only). A member is refused by the page yet served the full staff list by the API behind it.

## 5. Security conclusions

**Profile request.** Signed in as alice.nolan (a member), I sent GET /api/profile/18 — changing only the id from my own 17 to 18. The server returned 200 with Bob Keane's full profile (id 18, username bob.keane, his Finance email, department, role and bio). The rule the server should apply is an ownership or authorisation check: the profile API should confirm the requested id belongs to the signed-in user (or that the user is an admin) before returning the record, instead of only checking that someone is signed in. I observed one other account (id 18); reading every account by stepping through ids is likely but I did not test it, so I keep that conditional.

**Administration request.** Signed in as alice.nolan (a member), I sent GET /api/admin/users. The server returned 200 with the full staff account list, including Morgan Doyle (role admin) — even though GET /admin/users returned 403 "Administrators only." for the same account. The role rule the server should apply is the admin check: /api/admin/users should require the administrator role (the same requireAdmin the page uses), so both entry points to the same data make the same decision.

**Directory comparison.** At /directory, a signed-in colleague can already see each person's display name, department, email and role. The profile API adds username and bio (and the numeric id); the admin API adds username and returns every account in one response. Neither API returns the password hash. Because name, email, department and role are already visible to any signed-in colleague, the extra data exposed by these routes is limited to username and bio. This means I can claim a control error — the server returns data without the right authorisation check — without claiming that every returned value was secret.

**Evidence still missing or uncertain.** I did not test whether the profile IDOR lets a member read every account. A check that would settle it: as Alice, request a few more ids (for example 99, and a non-existent id such as 4242) and record whether each returns a record, an empty result, or a 404. That would show whether the gap is one record or the whole table.

**Assistant or self-review.** I used an AI assistant to review my profile conclusion. It claimed that the profile API exposes little beyond the directory, because email, name, department and role are already shown at /directory, and that the only fields it adds are username and bio. I checked this against my own evidence: the /api/profile/18 response fields versus the /directory columns, and the field list in src/handlers/api.js. The evidence supports the claim, so I bounded my impact statement to username and bio rather than calling the whole record secret.

---

## Optional, if you had time

- `GET /api/profile/4242` as Alice: status and body: status and body: 404 Not Found - {"error":"There is no colleague with that id."}

- Which guard does each API route call in `src/routes/api.js`?
Both API routes call requireAuth and nothing more — apiRoutes.get("/api/profile/:id", requireAuth, profileJson) (line 8) and apiRoutes.get("/api/admin/users", requireAuth, staffAccountsJson) (line 9). Neither calls requireAdmin, and neither checks that the requested record belongs to the signed-in user.

- Which guard does the administration page call in `src/routes/admin.js`?
The admin page calls requireAdmin — adminRoutes.get("/admin/users", requireAdmin, showAdminUsers) (line 8).

- How does that source comparison explain, but not replace, your recorded responses?
The guards explain the cause of what I saw. The admin page uses requireAdmin, so Alice got 403; the admin API uses only requireAuth on the same data, so Alice got 200 — the different guards are why one entry point refused her and the other did not. The profile API also uses only requireAuth with no ownership check, which is why changing 17 to 18 returned Bob's record. But the source does not replace the recorded responses: reading the code shows how the routes are configured, not what the running server actually returned. Middleware order, other code paths, or a different build could behave differently. The captured requests and responses prove the deployed behaviour; the source explains the mechanism behind it. The conclusion needs both — the evidence and its cause.
