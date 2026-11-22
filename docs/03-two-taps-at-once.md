# 3. Two taps in the same instant

[← Back to the overview](../README.md)

## The problem

A shop has several screens: two tills, kiosks, the website, the kitchen. In a rush they do tap at the same moment.
The same order is paid at both tills. A family uses one loyalty card on three kiosk orders. The last sandwich on the
shelf is ordered on the website and at the counter.

Code that reads, changes and saves loses one of the two. Nothing crashes and no test fails; a customer is charged
twice or a stamp goes missing, and the owner finds out at the day close.

## The pieces

**A named lock that needs no row.** The second request waits for the first, then reads the truth. The lock is a
PostgreSQL advisory lock taken on a name and held until the transaction ends:

```csharp
public static Task KeyAsync(AppDbContext db, string key, CancellationToken ct)
{
    if (db.Database.CurrentTransaction is null)
        throw new InvalidOperationException("A lock is held for a transaction: start one first (DbLocks.BeginAsync).");
    return db.Database.ExecuteSqlAsync($"SELECT pg_advisory_xact_lock(hashtextextended({key}, 0))", ct);
}
```

A name such as `loyalty:1:2035551234` works before any row exists. That matters: two orders that both create the
same new loyalty member cannot lock a row that is not there yet, but they can queue on its name.

**One change at a time to the thing a route names.** An endpoint filter wraps a route in a transaction and takes the
lock for the id in its address:

```csharp
group.MapPost("/orders/{id}/deliver", Deliver)
     .AddEndpointFilter(DbLocks.Gate("wholesale-order", "id"));
```

The route's own checks then do the rest. The second request waits, reads the order as it now is, sees "delivered
already" (or "refunded in full already", on a till) and says so. No route needed new logic; it only needed to see the truth.

**Counts change in one statement.** Stock is taken with a single conditional update, so the last one on the shelf
goes to exactly one order. The other is told: *"Sorry, … just sold out. Please remove it from your order."*

**A checkout has a key.** Each checkout makes one key and sends it with the order, and a unique index holds it. A
retry returns the order that was already made.

## A bug this closed

An accessibility audit, of all things, turned up a real one. At a kiosk, when the connection dropped after the order
was saved, the screen said "We couldn't place your order". The customer tapped "Place order" again and the kitchen
made the same order twice, with two rounds of loyalty stamps. The website already sent a key per checkout; the kiosk
did not. Now it does, and the same key is answered with the order it made the first time, before anything else is
looked at. Two requests in the same instant are caught by the same index.

## What changed for people

- An order paid at two tills at once is paid once. The other till reads "Order 1104 was just paid at another till."
- The last one on the shelf goes to one customer.
- A refund, a void or a cancel tapped twice happens once.
- A house account cannot pass its limit from two tills.
- One loyalty card on several orders at once gets every stamp.
- "Clock in" tapped twice makes one shift, and a scheduled order prints one kitchen ticket.

## Testing something that only happens by chance

Unit tests fire the two requests together on purpose and assert on both answers. That proves each case, but not the
ones nobody thought of, so there is also a **lunch rush on demand**: a script that plays a hundred website customers,
the kiosks, the tills and the kitchen at once against an API it starts on its own empty database, then checks the
totals that only come out wrong when something was lost.

Two details I am glad I built in:

- It refuses any address that is not this computer, so it can never be pointed at a real shop.
- It leaves the developer's own database and running app untouched, and drops its database when it is done.

## What I would tell another engineer

Do not scatter locking through business logic. Put the wait at the edge, name what is being changed, and let the
ordinary validation run against fresh data. Most of these fixes were one line on a route.
