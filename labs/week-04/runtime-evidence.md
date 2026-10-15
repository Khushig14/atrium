# Week 4 runtime evidence

Your name: Khushi Ganatra
Date: 12/10/2026
Atrium address and port: http://localhost:9090
ZAP run: version 2.17.0

Fill this in as you work. If a step failed, say what you tried. Do not paste a full session cookie value.

## 1. Coverage before sign-in

- Account state: signed out
- Routes reached by the spider: /, /robots.txt, /sitemap.xml, /css and /css/app.css (/css/ too), /login
- My direct `GET /directory` request: status, redirect and final page: 
Status: 302 Found
Redirect: http://localhost:9090/login
Final page: Sign in page
- What this run could not test: The spider never reached /directory or the search (/directory?q=), so this signed-out run says nothing about them. /dashboard was only seen as a redirect to sign-in. Pages that need a staff login were not tested.

## 2. Coverage after sign-in

- Seeded account used: alice.nolan
- Evidence that ZAP received the dashboard as this account: pages like dashboard, directory show 200 status in history section
- Evidence that ZAP received `/directory?q=Alice` as this account: when search or clicking on alice that status for `/directory?q=Alice` is shown as 200.
- Routes reached as this user:/dashboard, /directory, /resources, /css, /profile, /documents
- Routes or states newly reached compared with section 1: /dashboard, 'directory, /profile, /documents
- Active scan target and how I kept login and logout out of it: I active scanned only GET:directory(q), the search request with the q parameter, with my Context and User Alice selected. I excluded http://localhost:9090/logout.* in the Context settings, and I did not include GET /login, POST /login or POST /logout in the scan.

## 3. Scanner alert

- Alert name: Cross Site Scripting (Reflected)
- Risk and confidence as ZAP reported them: High risk, Medium confidence
- Method and affected URL: GET http://localhost:9090/directory?q=%3C%2Fstrong%3E%3Cscrlpt%3Ealert%281%29%3B%3C%2FscRipt%3E%3Cstrong%3E
- The input ZAP sent: </strong><scrIpt>alert(1);</scRipt><strong>
- Response status and relevant evidence: 200 OK. ZAP’s Evidence field holds the same string as the attack. In the Response tab the string appears inside <p class="result-summary">Showing colleagues matching ...</p>, in the page body.
- Where I saw the request and response: Alerts tab for the alert details, then the linked request and response in History (Response tab).

## 4. Manual comparison

**Ordinary search**

- Account, method and full URL: alice.nolan, GET, http://localhost:9090/directory?q=Alice
- Status and any redirect: 200 OK, no redirect
- Relevant response text or visible result: The page shows “Showing colleagues matching Alice.” and one result, Alice Nolan (Operations, alice.nolan@atrium.example). The header says “1 colleague shown across Atrium.”

**Alert request repeated by me**

- Account, method and full URL: alice.nolan, GET http://localhost:9090/directory?q=%3C%2Fstrong%3E%3Cscrlpt%3Ealert%281%29%3B%3C%2FscRipt%3E%3Cstrong%3E
- Status and any redirect: 200 OK, no redirect
- Relevant response text or visible result: in <p class="result-summary">, the submitted string appears unencoded, with the scrIpt tags as raw markup. The page also says “No colleagues found”.
- Did the alert's observation repeat? If not, what happened? Yes. Sending the same request in ZAP’s Requester as alice.nolan returned the same 200 response with the string reflected in the same place.

## 5. Conclusion

**What this evidence proves.** Limit the answer to this route, account and response.
For GET /directory?q= as alice.nolan, Atrium returned a 200 page in which the submitted q value (</strong><scrIpt>alert(1);</scRipt><strong>) was written into the result-summary paragraph as raw HTML, not encoded. Sending the same request again in ZAP’s Requester, with Alice’s session, returned the same 200 response with the string in the same place. So the search value is reflected without output encoding on this route.

**What this evidence does not prove.** Name an impact or wider claim that would need another test.
It does not prove that a script runs in a browser. I did not record an alert box or any script execution, only that the markup appears in the response body. It does not prove that other pages, parameters or accounts are affected, that any data was read, stolen or changed, or that an attacker could get a victim to open such a link. Each of those would need another test, for example opening the URL in a browser and recording what happens, or testing other routes and accounts.

## 6. Short finding

Write a short paragraph that gives the observation, location, possible impact, evidence and a high-level repair. Keep untested impact conditional.
On my local Atrium, logged in as alice.nolan, the staff directory search at GET /directory?q= writes the q value into the result-summary paragraph without encoding it. ZAP’s active scan raised Cross Site Scripting (Reflected), High risk and Medium confidence, with the payload </strong><scrIpt>alert(1);</scRipt><strong> (History row 486). When I repeated the same request myself in ZAP’s Requester, it returned 200 OK and the string appeared in the page as raw markup, in the same place. An ordinary search for Alice (History row 259, 200 OK) shows the value in the same paragraph as plain text, so the difference is whether the markup is encoded. If a browser runs this markup, a crafted link opened by a signed-in colleague could run script in their session, I did not test that, so I have not shown script execution or any data exposure. A repair would be to encode the search value for HTML output (or use a template engine that does so by default), and to add a Content-Security-Policy as a second layer.

---

## Optional, if you had time

Which signed-in page did you open that was absent from your spider's route list? What does that tell you about this scan's coverage?
Any page I opened by hand appears in ZAP’s tree because ZAP proxies my browser traffic, but the spider did not find it by crawling. So the tree alone overstates what the scan reached. Only the spider’s URLs list, and the active scan’s target, show what ZAP tested.
