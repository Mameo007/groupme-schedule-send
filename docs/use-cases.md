# Use Cases

**Project:** GroupMe Scheduled-Send Web App
**Version:** 0.1

---

## Identifiers

Use cases are identified as `UC-<AREA>-<slug>`. The area codes are listed in section 3. An identifier is never renumbered, renamed, or pointed at a different use case. Inside a use case, `PRE-n`, `POST-n`, and step numbers are local to that use case.

## Revision History

| Date | Version | Description | Author |
|---|---|---|---|
| 2026-09-23 | 0.1 | Initial use cases based on the end-to-end flow in the README | Kanta Endo |

---

## 1. Introduction

### 1.1 Purpose

This document describes what a User can accomplish with the GroupMe Scheduled-Send Web App, and what the Scheduler does on its own. It is detailed enough for a developer to know what to build and for a tester to know what to check. Terms with capitals are defined in the [project glossary](project-glossary.md). Requirement identifiers point to the [SRS](software-requirements-specification.md).

### 1.2 Scope

This document covers the areas `AUTH`, `BOT`, `MSG`, and `SND`, which together are the full Setup-and-send flow described in the README. It does not yet specify viewing, editing, or cancelling Pending Scheduled Messages after Setup, because whether that is in scope is not decided (SRS section 4).

---

## 2. Use Case Template

Every use case below has these fields, in this order:

- **UC ID and Name**: the identifier, then a name that starts with a verb.
- **Created By** and **Date Created**
- **Primary Actor** and **Secondary Actors**
- **Trigger**: the event that starts the use case.
- **Description**: why the use case exists and what it achieves.
- **Preconditions** (`PRE-n`): must be true before the use case starts, and each must be something the system can check.
- **Postconditions** (`POST-n`): the state of the system when the use case succeeds.
- **Main Success Scenario**: numbered steps, alternating between actor and system, in the present tense.
- **Extensions**: alternative flows and exceptions, numbered from the step they branch off (`3a`, `3a1`).
- **Priority**: High, Medium, or Low.
- **Frequency of Use**
- **Requirements**: SRS identifiers only, never their text.
- **Associated Information**: data fields, validation, and what happens on a system failure.
- **Related Use Cases**
- **Assumptions**
- **Open Issues**

---

## 3. Use Case List

| Area code | Feature area | Use cases |
|---|---|---|
| AUTH | Authentication and Sessions | `UC-AUTH-log-in` |
| BOT | Group and Bot setup | `UC-BOT-choose-group`, `UC-BOT-select-existing-bot`, `UC-BOT-create-bot` |
| MSG | Scheduled Messages | `UC-MSG-schedule-message` |
| SND | Sending | `UC-SND-send-scheduled-message` |

Possible use cases whose scope is not decided yet, so they are not specified: viewing Pending Scheduled Messages and cancelling one (these would be in `MSG`).

---

## 4. Use Cases

## AUTH: Authentication and Sessions

### UC-AUTH-log-in: The User logs in with GroupMe

**UC ID and Name:** `UC-AUTH-log-in`: Log in with GroupMe
**Created By:** Kanta Endo
**Date Created:** 2026-09-23
**Primary Actor:** User
**Secondary Actors:** GroupMe (OAuth)
**Trigger:** The User chooses to log in with GroupMe.
**Description:** The User gives the system temporary access to their GroupMe account so that Setup can happen. The system keeps the Access Token only inside a Session that expires on its own.

**Preconditions:**

- PRE-1. The browser does not have a valid Session.

**Postconditions:**

- POST-1. A Session exists that holds the User's verified Access Token and has the Session TTL.
- POST-2. The browser holds a cookie with only the Session ID.
- POST-3. The Access Token is not stored anywhere else.

**Main Success Scenario:**

1. The User chooses to log in with GroupMe.
2. The system sends the browser to GroupMe's authorization page.
3. The User signs in to GroupMe, if they are not signed in already, and authorizes the app.
4. GroupMe sends the browser to the system's callback URL with an Access Token.
5. The system checks the Access Token with GroupMe.
6. The system creates a Session holding the Access Token, with the Session TTL, and sets the session cookie.
7. The system starts `UC-BOT-choose-group`.

**Extensions:**

- **PRE-1a. The browser already has a valid Session:**
    - PRE-1a1. The system skips login and starts `UC-BOT-choose-group`.
- **3a. The User declines authorization or leaves GroupMe's page:**
    - 3a1. No Session is created. If the browser comes back to the system, it shows the login page again.
- **4a. The callback request has no Access Token:**
    - 4a1. The system creates no Session, shows a login-failed message, and offers to go back to step 1.
- **5a. GroupMe rejects the Access Token:**
    - 5a1. The system throws the token away without storing it, shows a login-failed message, and offers to go back to step 1.
- **5b. GroupMe cannot be reached or times out:**
    - 5b1. The system throws the token away, tells the User that GroupMe is unavailable, and offers to go back to step 1.
- **6a. The Session store is unavailable:**
    - 6a1. The system throws the token away, creates no Session, and tells the User to try again later.

**Priority:** High
**Frequency of Use:** Once for every Scheduled Message a User creates (see Open Issues).
**Requirements:** `FR-AUTH-oauth-redirect`, `FR-AUTH-verify-token`, `SEC-session-ttl`, `SEC-token-not-persisted`, `SEC-token-not-logged`, `SEC-redis-no-persistence`, `SEC-session-cookie`, `SEC-https-only`, `SI-groupme-oauth`

**Associated Information:**

| Property name | Data type | Validation rule | Security or access concerns | Glossary reference |
|---|---|---|---|---|
| access token | string | Required. Checked with GroupMe before a Session is created. | Kept only in the Session. Never logged, and never in any response the system sends. | Access Token |
| session ID | opaque string | Generated by the system, random | `HttpOnly`, `Secure`, `SameSite=Lax` cookie | Session |

On any failure in steps 4 to 6, no Session exists afterwards and the token is not kept anywhere.

**Related Use Cases:** `UC-BOT-choose-group`: Choose a Group.
**Assumptions:** `AS-user-has-groupme-account`. GroupMe sends the Access Token to the callback URL as a query parameter.
**Open Issues:**

- GroupMe's authorization URL does not document a `state` parameter. How does the system protect against login CSRF, where an attacker gets a victim's browser to complete the callback with the attacker's token?
- Is the User expected to log in again for every message, or should one Setup allow scheduling several messages before the Token Wipe?

---

## BOT: Group and Bot setup

### UC-BOT-choose-group: The User chooses a Group

**UC ID and Name:** `UC-BOT-choose-group`: Choose a Group
**Created By:** Kanta Endo
**Date Created:** 2026-09-23
**Primary Actor:** User
**Secondary Actors:** GroupMe (Groups API)
**Trigger:** `UC-AUTH-log-in` finishes, or the User goes back to the Group list during Setup.
**Description:** The User picks the Group the Scheduled Message will go to.

**Preconditions:**

- PRE-1. The browser has a valid Session that holds an Access Token.

**Postconditions:**

- POST-1. The Session records the chosen Group's ID and name.

**Main Success Scenario:**

1. The system gets from GroupMe every Group the User belongs to, using the Session's Access Token.
2. The system shows the Groups by name.
3. The User selects a Group.
4. The system records the chosen Group in the Session.
5. The system shows the Bot choice: select an existing Bot (`UC-BOT-select-existing-bot`) or create a new one (`UC-BOT-create-bot`).

**Extensions:**

- **1a. The User belongs to no Groups:**
    - 1a1. The system says there are no Groups to schedule to. The use case ends.
- **1b. GroupMe rejects the Access Token (it was revoked):**
    - 1b1. The system ends the Session and sends the browser to `UC-AUTH-log-in`.
- **1c. GroupMe cannot be reached or times out:**
    - 1c1. The system tells the User that GroupMe is unavailable and offers to retry step 1.
- **3a. The Session has expired by the time the User selects a Group:**
    - 3a1. The system sends the browser to `UC-AUTH-log-in`.
- **3b. The submitted Group is not one of the User's Groups (for example, a tampered request):**
    - 3b1. The system rejects the selection and goes back to step 2.

**Priority:** High
**Frequency of Use:** Once per Setup.
**Requirements:** `FR-AUTH-require-session`, `FR-AUTH-revoked-token`, `FR-GRP-all-groups`, `SI-groupme-groups`, `SI-groupme-unavailable`

**Associated Information:**

Display strategy: Group name. Sort order: as GroupMe returns them.

| Property name | Data type | Validation rule | Security or access concerns | Glossary reference |
|---|---|---|---|---|
| group ID | string | Required. Must be one of the Groups returned in step 1. | Only the User's own Groups | Group |

Nothing is persisted, so there is nothing to roll back on failure.

**Related Use Cases:** `UC-AUTH-log-in`, `UC-BOT-select-existing-bot`, `UC-BOT-create-bot`.
**Assumptions:** none
**Open Issues:** Should Groups be searchable or sorted by name for Users who belong to many Groups?

### UC-BOT-select-existing-bot: The User selects an existing Bot

**UC ID and Name:** `UC-BOT-select-existing-bot`: Select an existing Bot
**Created By:** Kanta Endo
**Date Created:** 2026-09-23
**Primary Actor:** User
**Secondary Actors:** GroupMe (Bots API)
**Trigger:** The User chooses to use an existing Bot after choosing a Group.
**Description:** The User reuses a Bot they already own in the chosen Group, so the Group doesn't end up with a new Bot for every message.

**Preconditions:**

- PRE-1. The browser has a valid Session that holds an Access Token.
- PRE-2. The Session records a chosen Group.

**Postconditions:**

- POST-1. A Bot record for the selected Bot exists in the database.
- POST-2. The Session records the selected Bot.

**Main Success Scenario:**

1. The system gets from GroupMe the Bots the User owns, using the Session's Access Token.
2. The system shows only the Bots that belong to the chosen Group, by name.
3. The User selects a Bot.
4. The system stores a Bot record (Bot ID, Group ID, Group name, Bot name) if one does not already exist, and records the selection in the Session.
5. The system starts `UC-MSG-schedule-message`.

**Extensions:**

- **1a. GroupMe rejects the Access Token:** as `UC-BOT-choose-group` 1b.
- **1b. GroupMe cannot be reached or times out:**
    - 1b1. The system tells the User that GroupMe is unavailable and offers to retry step 1.
- **2a. The User owns no Bots in the chosen Group:**
    - 2a1. The system says so and offers `UC-BOT-create-bot`.
- **3a. The User decides to create a new Bot instead:**
    - 3a1. The system starts `UC-BOT-create-bot`.
- **3b. The Session has expired:**
    - 3b1. The system sends the browser to `UC-AUTH-log-in`.
- **3c. The submitted Bot is not one of the Bots shown in step 2:**
    - 3c1. The system rejects the selection and goes back to step 2.
- **4a. The database is unavailable:**
    - 4a1. The system stores nothing, tells the User to try again, and stays at step 2.

**Priority:** High
**Frequency of Use:** Once per Setup, when the User already has a Bot in the Group.
**Requirements:** `FR-AUTH-require-session`, `FR-AUTH-revoked-token`, `FR-BOT-filter-by-group`, `SEC-bot-id-confidential`, `SI-groupme-bots`, `SI-groupme-unavailable`

**Associated Information:**

Display strategy: Bot name. Sort order: Bot name, ascending.

| Property name | Data type | Validation rule | Security or access concerns | Glossary reference |
|---|---|---|---|---|
| bot | reference | Required. Must be one of the Bots shown in step 2. | The Bot ID is confidential. The page should refer to Bots by a reference that is not the Bot ID. | Bot, Bot ID |

**Related Use Cases:** `UC-BOT-choose-group`, `UC-BOT-create-bot`, `UC-MSG-schedule-message`.
**Assumptions:** GroupMe lists only Bots the User created, so Bots other Group members created are not offered.
**Open Issues:** none

### UC-BOT-create-bot: The User creates a Bot

**UC ID and Name:** `UC-BOT-create-bot`: Create a Bot
**Created By:** Kanta Endo
**Date Created:** 2026-09-23
**Primary Actor:** User
**Secondary Actors:** GroupMe (Bots API)
**Trigger:** The User chooses to create a new Bot after choosing a Group.
**Description:** The User creates a Bot in the chosen Group that will post their Scheduled Messages.

**Preconditions:**

- PRE-1. The browser has a valid Session that holds an Access Token.
- PRE-2. The Session records a chosen Group.

**Postconditions:**

- POST-1. A new Bot exists in the chosen Group on GroupMe.
- POST-2. A Bot record for it exists in the database.
- POST-3. The Session records the new Bot as selected.

**Main Success Scenario:**

1. The system asks for a Bot name.
2. The User enters a name and confirms.
3. The system validates the name.
4. The system creates the Bot in the chosen Group through GroupMe, using the Session's Access Token and no callback URL.
5. The system stores a Bot record and records the new Bot as selected in the Session.
6. The system starts `UC-MSG-schedule-message`.

**Extensions:**

- **2a. The User cancels:**
    - 2a1. The system goes back to the Bot choice in `UC-BOT-choose-group` step 5.
- **2b. The Session has expired:**
    - 2b1. The system sends the browser to `UC-AUTH-log-in`.
- **3a. The name is empty or not valid:**
    - 3a1. The system shows what is wrong and goes back to step 1, keeping what the User typed.
- **4a. GroupMe rejects the request:**
    - 4a1. The system stores nothing, shows GroupMe's reason when there is one, and goes back to step 1.
- **4b. GroupMe rejects the Access Token:** as `UC-BOT-choose-group` 1b.
- **4c. GroupMe times out:**
    - 4c1. The Bot may or may not have been created. The system stores nothing, tells the User this, and offers `UC-BOT-select-existing-bot` so they can check before trying again.
- **5a. The database is unavailable after GroupMe created the Bot:**
    - 5a1. The Bot exists on GroupMe but not in the system. The system tells the User to try again and offers `UC-BOT-select-existing-bot`, which will list the new Bot.

**Priority:** High
**Frequency of Use:** At most once per Group per User, if Users reuse Bots.
**Requirements:** `FR-AUTH-require-session`, `FR-AUTH-revoked-token`, `FR-BOT-no-callback`, `SEC-bot-id-confidential`, `SI-groupme-bots`, `SI-groupme-unavailable`

**Associated Information:**

| Property name | Data type | Validation rule | Security or access concerns | Glossary reference |
|---|---|---|---|---|
| bot name | string | Required, not blank. Maximum length is GroupMe's limit (to be confirmed). | Visible to every member of the Group | Bot |

Failures leave either nothing, or a Bot on GroupMe with no record in the system (4c, 5a). The second case is recovered through `UC-BOT-select-existing-bot`. The system never rolls back a Bot on GroupMe.

**Related Use Cases:** `UC-BOT-choose-group`, `UC-BOT-select-existing-bot`, `UC-MSG-schedule-message`.
**Assumptions:** Any member of a Group can add a Bot to it.
**Open Issues:**

- Should the User be able to set an avatar image for the Bot?
- What is GroupMe's maximum length for a Bot name?

---

## MSG: Scheduled Messages

### UC-MSG-schedule-message: The User schedules a message

**UC ID and Name:** `UC-MSG-schedule-message`: Schedule a message
**Created By:** Kanta Endo
**Date Created:** 2026-09-23
**Primary Actor:** User
**Secondary Actors:** none
**Trigger:** A Bot has been selected or created during Setup.
**Description:** The User writes a message and chooses a future Send Time. The system stores it as a Scheduled Message and then wipes the Access Token, which ends Setup.

**Preconditions:**

- PRE-1. The browser has a valid Session that holds an Access Token.
- PRE-2. The Session records a selected Bot, and that Bot has a record in the database.

**Postconditions:**

- POST-1. A Scheduled Message with status Pending exists for the selected Bot, and its scheduler job is registered.
- POST-2. The Access Token no longer exists in the Session store.

**Main Success Scenario:**

1. The system shows a message composer with the chosen Group's name, the Bot's name, a text field, and a Send Time field in the User's time zone.
2. The User enters the text and the Send Time and submits.
3. The system validates the text and the Send Time.
4. The system stores the Scheduled Message with status Pending, converts the Send Time to UTC, and registers its scheduler job, all as one unit.
5. The system deletes the Access Token from the Session.
6. The system shows a confirmation with the Group, the Bot, the text, and the Send Time in the User's time zone.

**Extensions:**

- **2a. The User leaves without submitting:**
    - 2a1. Nothing is stored. The Session and its Access Token expire when the Session TTL runs out.
- **3a. The text is empty, only whitespace, or too long:**
    - 3a1. The system shows what is wrong and goes back to step 1, keeping what the User entered.
- **3b. The Send Time is missing, not valid, or not in the future:**
    - 3b1. The system shows what is wrong and goes back to step 1, keeping what the User entered.
- **3c. The Session has expired before submission:**
    - 3c1. The system sends the browser to `UC-AUTH-log-in`. What the User typed is lost.
- **4a. The database is unavailable:**
    - 4a1. Nothing is stored and the Access Token is **not** wiped, because Setup did not finish. The system tells the User to try again, and goes back to step 1 with what they entered.
- **5a. The Session store is unavailable when the system tries the Token Wipe:**
    - 5a1. The Scheduled Message stays stored. The token still expires when the Session TTL runs out. The system shows the confirmation and records the error without the token in it.

**Priority:** High
**Frequency of Use:** Once per Setup. This is the goal of the whole Setup flow.
**Requirements:** `FR-AUTH-require-session`, `FR-MSG-future-send-time`, `FR-MSG-text-length`, `FR-MSG-atomic-schedule`, `DI-send-time-utc`, `UI-local-time-display`, `SEC-token-wipe`, `SEC-session-ttl`

**Associated Information:**

| Property name | Data type | Validation rule | Security or access concerns | Glossary reference |
|---|---|---|---|---|
| text | string | Required. At least 1 character after trimming, and no longer than GroupMe's maximum for Bot posts (`FR-MSG-text-length`). | Posted publicly into the Group | Scheduled Message |
| send time | date and time, in the User's time zone | Required. Must be later than the moment the system receives it. | none | Send Time |

Step 4 either happens completely or not at all. Step 5 happens only after step 4 has succeeded.

**Related Use Cases:** `UC-BOT-select-existing-bot`, `UC-BOT-create-bot`, `UC-SND-send-scheduled-message`.
**Assumptions:** `AS-single-locale`.
**Open Issues:**

- How far in the future may a Send Time be?
- How does the system find out the User's time zone?
- Should the confirmation tell the User they need to log in again to schedule another message?

---

## SND: Sending

### UC-SND-send-scheduled-message: The Scheduler sends a Scheduled Message

**UC ID and Name:** `UC-SND-send-scheduled-message`: Send a Scheduled Message
**Created By:** Kanta Endo
**Date Created:** 2026-09-23
**Primary Actor:** Scheduler
**Secondary Actors:** GroupMe (Bots API)
**Trigger:** A Scheduled Message's Send Time arrives.
**Description:** The Scheduler posts the message to its Group through its Bot, with no User credentials involved, and records what happened.

**Preconditions:**

- PRE-1. The Scheduled Message exists and has status Pending.
- PRE-2. The Scheduled Message's Bot record exists.

**Postconditions:**

- POST-1. The message has been posted to the Group once, under the Bot's name.
- POST-2. The Scheduled Message has status Sent and a sent-at time.

**Main Success Scenario:**

1. The Scheduler fires the Scheduled Message's job.
2. The system loads the Scheduled Message and its Bot ID.
3. The system posts the text to GroupMe through the Bots API, using only the Bot ID.
4. GroupMe accepts the post.
5. The system sets the status to Sent and records when it was sent.

**Extensions:**

- **1a. The system was not running at the Send Time:**
    - 1a1. When the system restarts, it follows `ROB-missed-send`.
- **2a. The Scheduled Message is no longer Pending (for example, it was already sent):**
    - 2a1. The system does not post anything. The use case ends.
- **2b. The Scheduled Message or its Bot record no longer exists:**
    - 2b1. The system does not post anything, logs the event without the Bot ID, and the use case ends.
- **4a. GroupMe says the Bot does not exist (it was deleted on GroupMe):**
    - 4a1. The system sets the status to Failed with the reason "bot no longer exists". No retry.
- **4b. GroupMe returns another client error:**
    - 4b1. The system sets the status to Failed with GroupMe's reason. No retry.
- **4c. GroupMe returns a server error, cannot be reached, or times out:**
    - 4c1. The system follows `ROB-send-retry`. If it gives up, it sets the status to Failed with the reason.

**Priority:** High
**Frequency of Use:** Once per Scheduled Message.
**Requirements:** `FR-SND-status-guard`, `FR-SND-record-outcome`, `SEC-send-without-token`, `SEC-bot-id-confidential`, `ROB-restart-survival`, `ROB-at-most-once`, `ROB-missed-send`, `ROB-send-retry`, `PER-send-punctuality`, `AVL-scheduler-always-on`, `CO-single-scheduler-instance`, `CO-bots-api-only`

**Associated Information:**

The User is not told about the outcome in this release (`CI-no-user-notifications`). The only record is the Message Status.

**Related Use Cases:** `UC-MSG-schedule-message`.
**Assumptions:** The Bot is still in its Group at the Send Time, and GroupMe does not stop it posting because the User who created it has since left the Group.
**Open Issues:**

- What happens to a send that was missed because the system was down (send late, or mark Failed)?
- What is the retry policy, and how does it avoid posting twice after a timeout where GroupMe did in fact receive the post?
- What happens when the User who created the Bot has left the Group?
