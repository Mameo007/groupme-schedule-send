# Project Glossary

**Project:** GroupMe Scheduled-Send Web App
**Version:** 0.1

---

## Conventions

- The term itself is the identifier. Cite a term by writing it, with the spelling used here.
- One entry per concept. Synonyms are listed under the entry, not given their own.
- Entries are alphabetical.
- Definitions describe the concept, not its implementation.
- Where GroupMe already has a word for something, this project uses GroupMe's word.

## Revision History

| Date | Version | Description | Author |
|---|---|---|---|
| 2026-09-23 | 0.1 | Initial terms from the README | Kanta Endo |

---

## Definitions

### Access Token

The credential GroupMe issues when a User logs in through OAuth. Anyone holding it can act as that User across their whole GroupMe account, which is why this system keeps it only inside a Session and never anywhere else.

**Not to be confused with:** Bot ID, which only lets its holder post into one Group.

**Source:** GroupMe OAuth (see Implicit Grant). GroupMe does not issue a refresh token, so a revoked Access Token means the User has to log in again from scratch.

### Bot

A GroupMe participant that belongs to exactly one Group and posts messages under its own name, not under a User's. In this system a Bot only posts Scheduled Messages. It does not read or reply to anything in the Group.

**Not to be confused with:** a conversational chatbot. It is also not the User: members of the Group see the Bot's name on the message, not the User's name.

**Source:** GroupMe Bots API.

### Bot ID

The identifier GroupMe assigns to a Bot. Posting through the Bots API needs the Bot ID and nothing else, so no Access Token is needed when a message is sent. That also means anyone who has a Bot ID can post into its Group, so this system treats it as confidential.

### Bots API

The part of the GroupMe API used to create Bots, list them, and post messages through them. Creating and listing Bots need an Access Token. Posting does not.

**Not to be confused with:** Messages API.

### Group

A GroupMe group chat the User is a member of. A Scheduled Message is always sent to one Group, through a Bot that belongs to that Group.

**Not to be confused with:** a GroupMe direct message between two people. Bots cannot post into direct messages, so those are out of scope.

### Implicit Grant

The only OAuth flow GroupMe supports. The User approves the app on GroupMe's site, and GroupMe sends the Access Token straight back to the app's callback URL. There is no authorization code and no refresh token.

### Message Status

Where a Scheduled Message is in its lifecycle:

- **Pending**: stored and waiting for its Send Time.
- **Sent**: GroupMe accepted the post.
- **Failed**: the Scheduler tried to send it and could not. The failure reason is recorded.

A **Cancelled** status would also be needed if cancelling a Scheduled Message becomes part of the scope. That has not been decided yet.

### Messages API

The part of the GroupMe API that posts messages as the User. This system deliberately does not use it for sending, because the Access Token would then have to be kept until the Send Time.

**Not to be confused with:** Bots API.

### Scheduled Message

A piece of text that a User has asked to be posted to a Group, through a specific Bot, at a specific Send Time. It is the core record of the system.

**Not to be confused with:** the scheduler job that fires it. A job is how the system carries out a Scheduled Message, and is not a concept a User ever sees.

### Scheduler

The background part of the system that watches Scheduled Messages and posts each one through its Bot when its Send Time arrives. It works without any User credentials.

### Send Time

The moment a Scheduled Message should be posted. The User enters it in their own local time. The system stores it as an absolute instant (UTC), so it means the same moment no matter where the server runs.

### Session

A short-lived, server-side record that holds a User's Access Token during Setup. The browser holds only an opaque identifier for it. A Session ends when its Session TTL runs out or when a Token Wipe happens, whichever comes first.

**Not to be confused with:** a user account or a persistent login. This system has no accounts. Once the Session ends, it no longer knows who the User is.

### Session TTL

The fixed lifetime of a Session, counted from when it is created. After it runs out, the Session and its Access Token are gone automatically, even if the User never logged out. The README gives a 15 to 30 minute range. The exact value has not been decided yet.

### Setup

The sequence during which the system needs the User's Access Token: log in, choose a Group, select or create a Bot, and schedule a message. Setup finishes successfully when a Scheduled Message has been stored.

### Token Wipe

Deleting the Access Token from the Session as soon as Setup finishes successfully, instead of waiting for the Session TTL to run out.

### User

A person with a GroupMe account who logs in to this system to schedule a message. The User has no separate account in this system. Their identity comes from GroupMe, and only for the length of a Session.
