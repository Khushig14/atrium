# Week 2 security requirements

Your name: Khushi Ganatra
Date: 1/10/2026

Fill this in as you work, rather than at the end. Where you are unsure, write that you are unsure and say why. A sentence you can support is worth more than a confident one you cannot.

---

## 1. What this application is

Three or four sentences, in your own words, describing what Atrium does and who uses it.

**Where the assistant's explanation did not match the application.** Anything you checked and found different, however small. Write "nothing found" if that is the honest answer.

Atrium is local web application which is by staff of a company/directory, read documents uploaded by colleagues and keep their profile updated. Different credentials are used by different people to sign in into the application. Nothing different is found in the explaination of the assisstant.

## 2. What is worth protecting

Four assets. For each one, say what it is and what it would cost if it were seen, changed or unavailable. Write the cost so that somebody outside the team could understand it.

| Asset             | What it costs if this goes wrong                                          |
|-------------------|---------------------------------------------------------------------------|
| Staff details     | If the data is exposed, then the privacy of the staff can be compromised. And if manipulated, wrong person would be contacted and would be able to read the information shared with them. |
| Login accounts    | If credentials are exposed, someone could impersonate a staff member or someone could gain unauthorized access or be blocked from their account. |
| Role assignments  | Attackers can gain more access if they get to know a particular user has more priviledged access.                                                  |
| Shared resources  | If resources are exposed, internal applications/data and work can be revealed to outsiders which can be used for malicious benefits. |

## 3. The requirements

Four sentences, in your own words. Each one should say what is not allowed and to whom.

1. No one who is not signed in may view another person’s directory information, such as their name, email, department, or profile details.
2. No staff member may change another person’s profile, email, department, or bio, and only the account owner should be able to update their own information.
3. No one except an administrator may view the user list or change a person’s role, including granting or removing administrator access.
4. No user may delete, rename, or alter the shared resource information for a document unless they are authorized to manage that resource.

**Which of these did you write yourself, and which started as a draft from your assistant?** Say plainly. Both are fine.
The first two requirements started as my own thoughts and used assitand for further clarification. The rest is written with the help of the assistant. 

## 4. One I rejected or rewrote

- The original sentence: No one except an administrator may view the user list or change a person’s role, including granting or removing administrator access.
- My version: Only signed-in administrators may view the user list and staff users must be denied access to that page.
- Which test it failed, and why: Atrium has an administrator-only user-list page but it has no feature for changing user's roles.

## 5. How somebody would check one of these

Pick one requirement. Write the steps for a person who has never seen Atrium and cannot ask you anything.

- The requirement: Staff may update their own profile, but must not be able to change another person’s profile.
- Sign in as: bob.keane a staff member
- Steps: Open the profile and change the bio, save and confirm the change. Then use the browser’s developer tools to change the form’s hidden userId from Bob’s ID to Alice’s. Submit a different test bio. Sign out, sign in as alice.nolan, and check the profile.
- What result would mean the requirement is met: Bob can save his own change, but Alice’s profile remains unchanged after Bob’s second submission.
- What result would mean it is not met: Alice’s profile shows the bio Bob submitted.

---

## Optional, if you had time

Your four requirements in order, most important first, with one sentence each on why it is in that position.
1. Requirement 2: I rank profile protection first because the current app lets a signed-in staff member submit changes to another person’s profile.
2. Requirement 3: I rank administrator-only access next because the user list contains staff information that ordinary staff should not access.
3. Requirement 1: I rank the sign-in requirement third because it protects directory information from visitors who are not signed in.
4. Requirement 4: I rank resource protection fourth because Atrium currently displays resource details but does not provide features to edit or delete them.