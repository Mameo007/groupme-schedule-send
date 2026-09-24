# Open Issues

**Project:** GroupMe Scheduled-Send Web App

---

Every question about the project that can't be answered yet, and how it gets answered: a **decision** (a product call to make) or **research** (reading GroupMe's docs or testing against the API). When a question is answered, move it to **Resolved** with the answer and the date, and put the substance where it belongs: the [glossary](project-glossary.md), the [SRS](software-requirements-specification.md), or the [use cases](use-cases.md).

## Priorities

Settle these first. They have the highest cost if the guess turns out wrong:

1. **OI-1** and **OI-2**: they decide whether the product has any "manage my messages" feature and how often users have to log in. That shapes the data model and the whole Setup flow.
2. **OI-3** and **OI-4**: they decide what the Scheduler does when a send can't happen on time, including whether a message may ever be posted twice.
3. **OI-10**: the punctuality target decides which hosting options are viable.

OI-7, OI-15, OI-16 and OI-18 need research, which can happen while coding the first use cases.

## Open

| ID | Question | Why it matters | How to resolve | Raised |
|---|---|---|---|---|
| OI-1 | After Setup, can a User view, edit, or cancel their Pending Scheduled Messages? | If yes, Scheduled Messages must store the creator's GroupMe user ID, the User must log in again to manage them, a Cancelled status is needed, and new `MSG` use cases have to be written. It blocks SRS section 4 and the data dictionary in 7.2. | Decision | 2026-09-23 |
| OI-2 | Must a User log in again for every Scheduled Message, or can one Setup schedule several before the Token Wipe? If they must log in again, should the confirmation page say so? | It decides when `SEC-token-wipe` runs and how Setup ends in `UC-MSG-schedule-message`. Logging in for every message has a usability cost. | Decision | 2026-09-23 |
| OI-3 | If a Send Time passes while the system is down, should the message be sent late (and up to how late), or marked Failed? | Blocks `ROB-missed-send` and `UC-SND-send-scheduled-message` extension 1a. Sending a message late could be worse than not sending it at all. | Decision | 2026-09-23 |
| OI-4 | When GroupMe returns a server error or times out during a send, should the system retry? If it does, is an occasional duplicate post acceptable, given that the first attempt may have gone through? | Blocks `ROB-send-retry`. It conflicts with `ROB-at-most-once` unless one of the two is chosen. | Decision | 2026-09-23 |
| OI-5 | What is the exact Session TTL, within the README's 15 to 30 minute range? | Fills in `SEC-session-ttl`. Too short cuts off slow Users partway through Setup. Too long keeps the Access Token around longer. | Decision | 2026-09-23 |
| OI-6 | How does the system find out the User's time zone: detected by the browser, chosen by the User, or both? | Blocks `DI-send-time-utc` and `UI-local-time-display`. Getting it wrong sends messages at the wrong hour. | Decision | 2026-09-23 |
| OI-7 | GroupMe's OAuth doesn't document a `state` parameter. How does the system protect against login CSRF, and does GroupMe keep extra query parameters on the callback URL? | It is a security gap in `UC-AUTH-log-in`. Without protection, an attacker could log a victim into the attacker's GroupMe account. | Research (GroupMe docs or testing) | 2026-09-23 |
| OI-8 | How long are Sent and Failed Scheduled Messages kept? Is a Bot record deleted once it has no Pending messages? | Blocks SRS section 7.4. Bot IDs are confidential, so keeping them forever keeps a credential around. | Decision | 2026-09-23 |
| OI-9 | How far in the future can a Send Time be? | Adds a validation rule to `UC-MSG-schedule-message`. Also affects how long jobs and Bot IDs stay stored. | Decision | 2026-09-23 |
| OI-10 | Within how many seconds of its Send Time must a message be posted? | Fills in `PER-send-punctuality`. Also decides whether hosting that sleeps or restarts slowly is acceptable (`OE-hosting-platform`). | Decision | 2026-09-23 |
| OI-11 | What uptime target does the system have? | Fills in `AVL-uptime`. Rules out hosting candidates that can't meet it. | Decision | 2026-09-23 |
| OI-12 | Which browsers and versions must be supported, including mobile? | Fills in `OE-supported-browsers`. Decides what the UI has to be tested on. | Decision | 2026-09-23 |
| OI-13 | How many Users and Scheduled Messages are expected, per day and at peak? | Fills in `SCA-expected-load`. Tells us whether one Scheduler instance and the GroupMe rate limits are enough. | Decision | 2026-09-23 |
| OI-14 | What is the maximum number of page transitions from the start page to a stored Scheduled Message? | Fills in `USE-setup-steps`. | Decision | 2026-09-23 |
| OI-15 | What is GroupMe's maximum text length for a Bot post? We believe it is 1,000 characters, but haven't verified it. | Fills in the limit in `FR-MSG-text-length`. If it's wrong, messages pass validation and then fail at send time. | Research (GroupMe docs or testing) | 2026-09-23 |
| OI-16 | What is GroupMe's maximum length for a Bot name? | Adds a validation rule to `UC-BOT-create-bot`. | Research (GroupMe docs or testing) | 2026-09-23 |
| OI-17 | Should a User be able to set an avatar image when creating a Bot? | Adds a field and an upload or URL input to `UC-BOT-create-bot`. | Decision | 2026-09-23 |
| OI-18 | Can a Bot still post after the User who created it has left the Group? | Decides whether `UC-SND-send-scheduled-message` needs another failure path, and whether its assumption holds. | Research (GroupMe docs or testing) | 2026-09-23 |
| OI-19 | Should the Group list be searchable or sorted by name for Users who belong to many Groups? | Changes the display strategy in `UC-BOT-choose-group`. | Decision | 2026-09-23 |

## Resolved

| ID | Question | Answer | Decided by | Date | Filed in |
|---|---|---|---|---|---|
