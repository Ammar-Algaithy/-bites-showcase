# 5. Testing money

[← Back to the overview](../README.md)

## The problem

A point of sale is trusted with one thing above all: the drawer has to match the report at the end of the day. A
wrong cent in tax rounding, a refund that goes back to the wrong tender, a split payment that counts the cash part
twice: each is small, and each is the kind of thing an owner never forgives.

Tests for money fail in a particular way. It is easy to write one that reads a total off the screen and checks that
the screen shows that total. It passes, and proves nothing.

## The server decides, so there is one thing to test

A screen sends ids, choices and quantities. It never sends a price. The server prices the order from the menu it
holds, adds the extras and the tax, applies loyalty, and answers with the result. A request that claims a sandwich
costs a dollar is not refused; the claim is simply never read.

That one rule removes a class of bugs and a class of attacks, and it means the arithmetic lives in one place.

## A shop's whole day, in a real browser

Before every release, eight tests run in Chromium against a server, a website and a database started only for them.
They are one day in one shop, in order, and each builds on the last:

1. The owner signs in on the sign-in page.
2. A cash sale at the till: the total with tax, and the change.
3. The kitchen works that order through to handed over.
4. A website order waits unpaid, is paid by card at the till under its own number, and the customer's page follows.
5. A split payment, part cash and part card.
6. A refund of one item, in cash.
7. A void.
8. The day close, with the drawer counted five dollars short and then exactly right.

The last test expects the sum of everything the others did. A mistake anywhere in the day surfaces at the close,
which is where a real shop would find it too.

## Rules the tests follow

- **Every expected amount is worked out in the test**, in whole cents, from the menu's prices and the shop's tax
  rate. Nothing is read off the screen and compared with itself.
- **Each sale is also read back from the server**, so the screen and the record are both checked.
- **Things are found the way a person finds them**: a button by its name, a dialog by its heading. A renamed button
  fails its test, and the fix is one line. No test depends on a CSS class.
- **The run owns its world.** It creates a brand-new database, starts its own server and website on their own ports,
  and drops everything afterwards. It refuses to start while the development stack is running, because the two would
  share a build folder.
- **A failure leaves evidence**: a screenshot and a trace that replays the browser step by step.

## Underneath: the API on a real database

The 498 API tests do not mock the database. Each test class gets its own PostgreSQL in a container, with every
migration applied, so a test exercises the same constraints, indexes and row-level security as production.

A few fixtures make hard things short:

- **Clients by role**: one line gives an HTTP client signed in as the owner, a cashier, or nobody.
- **A clock that moves**: tests advance time by hours or days to reach a day close, an expired hold or a lapsed
  session, without sleeping.
- **A cloud and a store together**: two servers on two databases, joined in memory, for everything that crosses the
  line between a shop and the cloud.

Test names are sentences, so the list of tests reads as the list of promises:

```
The_database_refuses_to_save_another_businesses_rows
Turning_a_business_off_signs_everyone_out_and_stops_sign_ins
Every_business_table_is_protected_in_the_database
```

## Front end: logic in pure functions

The 740 front-end tests lean on one habit: anything with a decision in it (pricing display, cart rules, which dishes
fit on a TV page, what the mascot says in which season) is a pure function in a library file, tested without
rendering. Components are tested with Testing Library for what a person can do with them, and an automatic
accessibility check reads the customer's screens.

## Run only what the change touches

The whole suite takes minutes, so the working rule is: run the tests for what changed, and the full suites before a
release or after touching code everything depends on. Fast feedback during the work, the full weight before
anything ships.

## What I would tell another engineer

Decide what a money test is allowed to trust. Mine trust the menu's prices and the tax rate, and nothing the
application computed. Everything else follows from that.
