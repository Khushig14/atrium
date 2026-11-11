# Week 8 remediation record

Your name: Khushi Ganatra
Date: 8/10/2026
Atrium version or copy used: http://localhost:9090

Fill this in as you work. Keep exact inputs and observed results. If you have not checked something, say so.

## 1. Original finding and before evidence

- Page and route: The colleague directory search, GET /directory
- Account used: alice.nolan
- Earlier finding or source I started from: In Week 6, I found a possible SQL Injection issue in the directory search (src/handlers/directory.js). ZAP also reported a related alert (Z1).
- Ordinary search text, URL, status and results-table rows: I searched for Nolan. The page returned status 200 and showed Alice Nolan, Operations, with her email and Member role.
- No-match search text, URL, status and results-table rows: I searched for zzzz. The page returned status 200 and displayed "No colleagues found" with no results.
- Comparison search text, URL, status and results-table rows: I searched for Nolan'. The page returned status 500 with a SQLite error. This showed that the apostrophe was being treated as part of the SQL query.
- What the comparison shows, and what it does not show: The apostrophe broke the SQL query and caused an error. This suggests that the search input was being added directly to the SQL instruction instead of being treated as normal search text. I did not try to access other database tables or information.


## 2. Root cause

- Source of the search text, with file and line: The search value comes from req.query.q in src/handlers/directory.js:19.
- Path through the handler: The handler reads the search value at line 19, passes it to searchDirectory(db, q) at line 21, and builds the SQL query using string concatenation at lines 10–11. The query then runs at line 13.
- Database operation, with file and line: db.prepare(sql).all() at src/handlers/directory.js:13 prepares and runs the SQL query. The error from the earlier test also points to this line.
- Control missing on that path: The query was not using parameter binding. The search value was added directly into the SQL text, allowing special characters to affect the query.
- Difference between the visible result and the root cause: The 500 error was only the visible result. The real problem was that user input was being added directly to the SQL query.

## 3. Security property and useful behaviour

- One sentence stating the property the repair must enforce: Search input must always be treated as normal data, not as part of the SQL instruction. Whatever the user types, it must not change the structure of the query.
- Ordinary behaviour that must remain, including name and department search: Searching by name or department must continue working. Partial searches should still work, searches with no matches should show "No colleagues found", and an empty search should still list all colleagues.

## 4. Candidate fixes

| Candidate | Where the control acts | Effect on the property and ordinary search | Decision |
|---|---|---|---|
| Parameterised query (bound `?` placeholders) | At the database boundary (db.prepare + bound values) | Fully enforces the property — input is data; name/department LIKE search unchanged | Chosen |
| Escape/double the apostrophes in `q` | In the handler before building the SQL string | Fragile and easy to miss cases; still assembles SQL from text; does not generalise to other metacharacters | Rejected |
| Strip or allow-list characters in `q` | In the handler on the input | Breaks legitimate names (e.g. O'Brien) and still does not address the root cause | Rejected |


- Assistant used, if any: Github Copilot.
- One suggestion I changed or rejected, and why: I rejected escaping/stripping the input — it treats the symptom, can be bypassed, and would break valid names like O'Brien. Parameter binding fixes the cause instead.
- Reason for my choice: parameter binding is the standard, robust fix. The database treats `q` strictly as a value, and the existing LIKE behaviour is preserved by binding the "%term%" pattern.

## 5. Change made

- File and function changed: src/handlers/directory.js, function `searchDirectory`.
- Changed lines or a short before and after code extract:
Before:
    export function searchDirectory(db, q) {
      const sql =
        "SELECT id, display_name, email, department, role FROM users " +
        "WHERE display_name LIKE '%" + q + "%' " +
        "OR department LIKE '%" + q + "%'";

      return db.prepare(sql).all();
    }

After:
    export function searchDirectory(db, q) {
      const sql =
        "SELECT id, display_name, email, department, role FROM users " +
        "WHERE display_name LIKE ? OR department LIKE ?";
      const pattern = "%" + q + "%";
      return db.prepare(sql).all(pattern, pattern);
    }
- How the change keeps search text separate from the database instruction: The SQL query now has a fixed structure. The search value is passed separately using parameter binding, so the database treats it as data rather than SQL instructions. The % symbols are still used to allow partial matches.
- What I left outside this repair: The document description issue was handled separately in Section 8. Other routes that already use parameter binding were left unchanged. The listColleagues function was also left as it does not take user input.

## 6. First checks after the change

- Same ordinary search: I searched for Nolan. The page returned status 200 and still showed Alice Nolan, Operations, with her email and Member role.
- Same no-match search: I searched for zzzzzz. The page returned status 200 and showed "No colleagues found" with no results.
- Same comparison search: I searched for Nolan'. This time the page returned status 200 instead of 500. It showed no matching results, meaning the apostrophe was treated as normal text.
- Empty search result: I opened /directory without entering anything. The page returned status 200 and listed all three colleagues: Alice Nolan (Operations), Bob Keane (Finance), and Morgan Doyle (IT Services).
- `npm test` result and any failing test: I ran npm test. All 22 tests passed, with 0 failures.
- `npm run scan` result, if available: I ran npm run scan. The directory search was no longer flagged. Two other findings remained in src/db/reset.js:12 and src/handlers/resources.js:1, but these were false positives because the SQL values were already passed using parameter binding.

## 7. Handoff to Week 9

- What this evidence supports: The directory search now uses parameter binding to keep user input separate from the SQL query. The apostrophe that previously caused a 500 error now returns a normal 200 response. Normal searches still work, and all 22 tests pass.
- Cases or related paths still to check: Other pages and routes that use user input in SQL queries or HTML output should also be checked. The remaining scanner alerts in reset.js and resources.js should be reviewed to confirm they are false positives.
- A focused automated test I would add: I would add an automated test to check that searching for Nolan' returns status 200 with no results, and that searching for Nolan still returns Alice Nolan. This would help prevent the same issue from coming back.
- Any known regression or remaining risk: No functional problems were found after the change. One small point is that % and _ are still treated as search wildcards. This is acceptable for the current search, but they could be escaped if exact matching is needed. The scanner may also report some string concatenation that is not actually unsafe.

## 8. Document description output

- Resource changed for the temporary check: I temporarily added <b> tags around the description of resource 1 ("Expense claim form") in src/db/seed-data.js. I then ran npm run reset to load the change.
- List and detail page before the repair: On both the Documents list page (/resources) and the document detail page (/resources/1), the description appeared in bold. This showed that the pages were displaying the description as HTML instead of normal text.
- Rule for printing a description as page content: A document description should be displayed as normal text. It should use <%= %> to escape the content, not <%- %> which displays raw HTML.
- Two view files and lines changed: I updated two files:
1. views/resources/show.ejs:11 – changed <%- resource.description %> to <%= resource.description %>.
2. views/resources/index.ejs:14 – changed <%- item.description %> to <%= item.description %>.
- List and detail page after the repair: Both pages displayed the <b> tags as normal text instead of making the description bold. This confirmed that the HTML was now being escaped.
- Ordinary description checked after restoring the seed data: removed the temporary <b> tags from the seed data and ran npm run reset again. The original description displayed normally on both the Documents list and detail pages.
- What this check does not establish: This test only checks the document description on these two pages. It does not prove that every other page or user input in the application is safe from HTML injection.