# Week 3 record

Your name: Khushi Ganatra
Date: 5/10/2026

Fill this in as you work, rather than at the end. Where you are unsure, write
that you are unsure and say why. That is worth more than a confident sentence you
cannot support.

---

## 1. The finding I chose

State which of the three the tool reported.

- File: reset.js
- Line: 12 & 13
- What the tool said about it: This SQL statement is built by joining pieces of text together. If any of those pieces came from outside the application, the database will read it as part of the command rather than as a value. 

## 2. What an assistant told me

Say which assistant you asked, and what it said in a sentence or two.

- Assistant used: Github copilot
- Its explanation, in your own words: The scanner flagged the code that joins text together to build a database command, which can be dangerous if a user's input gets mixed in. But the assistant says it is safe as the joined text is fixed and the real values go in separately through the placeholders.
- One thing it asserted that I had not verified at that point: The assistant said the joined pieces are all fixed text and the real values go in through placeholders, so user input can't become SQL. I had not checked this in the code.

## 3. What the code shows

Answer all four. If you cannot answer one, say so.

**Where does the data come from?** - It comes from USERS database in seed-data.js

**What happens to it on the way?** - It hashes each password and passes the records to the insert statement.

**Where does it become dangerous?** - If outside input is added to sql statements, SQL injection can occur before database is run.

**What stands in the way?** - The placeholders and sqlite3’s parameter binding keep each supplied field as a value rather than SQL syntax.

## 4. What the running application shows

Record both. A single result on its own proves nothing.

**Ordinary case**

- What I entered: I entered the username alice.nolan and its password SpringRiver44
- What came back: I was able to log in as Alice nolan and it showed me the Atrium interface.

**The case I was testing for**

- What I entered: I could not enter SQL inputs to the reset insert through the app.
- What came back: I could not test the testcase through the app. The reset command runs in the command line and uses its fixed data.
- How this differs from the ordinary case: I could test a normal method for login, but the app doesn’t expose this reset insert for an injection attempt.

## 5. My answer

Delete the two that do not apply.

**Not real** 

**Why, in two or three sentences.** Write for somebody who has not seen any of this. - The SQL text is made from fixed strings, and the user-record fields are passed separately through placeholders. I found no user input being added to this INSERT statement.

**What would change my mind.** If new information would alter this answer, say what. - I would reconsider if I found a path that puts user-controlled input into the SQL text, or if the code being run differs from the code I inspected.

**How far this answer reaches.** What you established applies to a particular page, a particular set of data and this version of the application. Say what you have shown, and be careful not to claim more. - This conclusion is about the INSERT in reset.js, using the fixed seed data in this version of the app. It does not establish that every database query in the application is safe.

## 6. Back to the assistant

The thing you noted in section 2, that you had not verified at the time.

- Did I check it? - Yes. I checked reset.js, the USERS data in seed-data.js, and how the insert passes values to the prepared statement.
- Was it right? - Yes for this insert: its SQL text is fixed, and the record fields are passed through placeholders. That does not prove every query in the app is safe.

---

## Optional, if you had time

The other two findings matched the same rule. Why are they not the same situation?
Two sentences.

1. 
- File: directory.js
- Line: 8,9,10 & 11
- What the tool said about it: This SQL statement is built by joining pieces of text together. If any of those pieces came from outside the application, the database will read it as part of the command rather than as a value.        
- Why: This one is a real problem. In directory.js, the search text that comes from the URL gets added straight into the SQL string, with nothing separating it from the rest of the command. That means someone could type SQL into the search box and the database would run it as part of the query, so they could change what the query does, like pulling back rows they shouldn't see.

2. 
- File: resources.js
- Line: 1,2,3,4 & 5
- What the tool said about it: This SQL statement is built by joining pieces of text together. If any of those pieces came from outside the application, the database will read it as part of the command rather than as a value.    
- Why: The scanner flagged this query because the SQL is built by joining strings together with +. When I looked at it, every piece being joined is fixed text written in the code. The resource ID from the URL doesn't get added to that text. It's passed in separately and fills the ? placeholder, so the database treats it as a value and not as part of the command. So the pattern matched, but nothing from outside the application ever ends up inside the SQL itself.