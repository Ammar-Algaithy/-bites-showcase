# 6. 161 accessibility checks

[← Back to the overview](../README.md)

## The problem

Ordering lunch should not depend on how well someone sees or whether they use a mouse. Bites puts ordering screens
in front of the public on phones, on shop websites in three visual templates, and on kiosks. I had built them with
care and without a method, which is how most accessibility debt is made.

## Start with an honest record

I audited every customer-facing screen against a 161-point checklist (WCAG 2.2 A and AA, the ordering journeys,
devices and screen readers, and the kiosk) and wrote down what the code actually did:

| Result | Checks |
|---|---:|
| Fail | 77 |
| Pass | 28 |
| Needs a person with a browser, a screen reader or the device | 27 |
| Not applicable | 16 |
| A business or legal matter, not software | 13 |

Seventy-seven failures is an uncomfortable number to publish. It is also the useful one. On the websites the
audit found nobody locked out: every control was already a real button, link or field, so an order could be
completed with a keyboard. It found that the screens were hard to *perceive*: faint text, focus that vanished,
changes nobody announced.

It also found two plain bugs that had nothing to do with accessibility. A kiosk could place the same order twice
after a dropped connection, and after a reload a kiosk showed the previous customer's order, name and phone number
to the next customer. Both are fixed. Reading every screen slowly has a way of finding things.

## Fix once, fix everywhere

Most failures traced back to a few shared causes, so the work went into shared pieces rather than screen by screen.

**Color became four tokens instead of one.** The brand orange read at 3.4:1 as text on the cream page, where 4.5:1 is
needed, and it was used 119 times. Darkening one color would have changed the brand. The accent is now four tokens
with four jobs: a fill, the words that sit on that fill, accent words on a light surface, and accent words on a dark
one. Each pairing is chosen to pass.

**One dialog.** Ten overlays had been built by hand, and none held the Tab key inside or gave focus back to what
opened it. They now share a single component built on the browser's own `<dialog>`: focus moves in, Tab stays in,
Escape closes the one on top, and focus returns where it came from.

**One announcer.** "Added to your order", search results and "your order is ready" appeared silently. A single
always-present live region now speaks them. The detail that matters: a live region inserted already filled is not
announced by screen readers, so the region exists from the start and messages are written into it.

**And the smaller things, done everywhere**: a skip link to the main content, a visible focus ring on every field,
field borders at 3:1 against the page, and scrollbars that show an area scrolls.

## Making it stay fixed

A fix that depends on the next person remembering is temporary. Three guards run with the tests:

**Contrast is a unit test.** It reads the color tokens from the stylesheet and holds every pairing to WCAG AA in
every look the app has: three shop templates and the wholesale console.

```ts
describe.each(Object.entries(looks))("%s colors", (_look, t) => {
  it.each(surfaces)("words read on %s", (surface) => {
    expect(ratio(t.ink, t[surface])).toBeGreaterThanOrEqual(4.5);
    expect(ratio(t.muted, t[surface])).toBeGreaterThanOrEqual(4.5);
    expect(ratio(t["orange-ink"], t[surface])).toBeGreaterThanOrEqual(4.5);
  });
});
```

Change a token to something too faint and the build fails before the color reaches a screen.

**An automatic check reads the customer's screens** in the component tests, and the accessibility lint rules run
with the type check.

**Conventions are written down** where the next change will see them: an overlay goes inside the shared dialog, never
a hand-made one; no opacity on words; nothing removes a focus outline without giving another.

## The bug only a real browser showed

The shared dialog passed every test and then, in the development build, closed every dialog the instant it opened.

Two harmless behaviours met. In development React runs each effect twice when a component appears: set up, clean up,
set up again. And a browser fires a dialog's `close` event slightly *after* the call that closed it. The clean-up
closed the dialog; the second set-up opened it again; then the late `close` event from the clean-up arrived, and the
dialog treated it as the user closing it.

No test saw it, because the simulated browser used by the tests does not implement `<dialog>` timing. Two lessons
went into the project's notes: a piece like this is tested inside React's strict mode, and new shared interface code
is opened once in a real browser before it is called done.

## What is still open

Not everything is closed, and saying so is part of the record:

- A non-visual mode for the kiosk (speech and navigation without sight) is not built.
- 27 checks can only be judged by a person with a screen reader or at the device itself.

## What I would tell another engineer

Audit before fixing, and count the failures honestly. Then look for the handful of shared causes. Here, four shared
pieces and three guards closed most of the list, and they keep closing it for every screen written since.
