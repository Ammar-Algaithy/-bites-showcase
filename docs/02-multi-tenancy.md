# 2. Four fences between businesses

[← Back to the overview](../README.md)

## The problem

Bites Cloud keeps many businesses in one database. A shop must never see another shop's orders, staff or prices, and
"never" has to survive the day someone adds a table in a hurry. One forgotten `WHERE business_id = …` is a data leak.

So the rule is not left to each query. Four independent fences each stop a mistake the others might miss, and a test
fails when a new table is missing one.

## Fence 1: every request belongs to exactly one business

A middleware runs right after authentication and settles the business before any endpoint is reached:

| Who is asking | Their business is |
|---|---|
| Staff, owner, kiosk, screen | The one on their account. It cannot be changed by the request. |
| The platform admin | The one picked in the console, one at a time. |
| A customer | The website's hostname (a shop's own domain or its address on Bites Cloud). |

A shop's public pages name their shop explicitly, and that wins over everything else: if someone signed in to
business A opens business B's website in the same browser, they are treated as an anonymous visitor there. A sign-in
can never be carried into another shop's pages.

## Fence 2: queries cannot see other rows

Every entity that belongs to a business implements one interface, and the database context adds a filter to each of
them when the model is built: `row.BusinessId == CurrentBusinessId`. The filter is added by walking the model, not
by hand per table, so a new entity gets it without anyone remembering.

When no business is set, the comparison is against nothing and **no rows match**. The failure is an empty page, not
everyone's data.

## Fence 3: saves are stamped and checked

On save, new rows are stamped with the request's business. A change or a delete of a row that belongs to a different
business throws before it reaches the database:

```
That belongs to another business.
```

This catches the case a read filter cannot: code that got hold of an entity some other way and tries to write it.

## Fence 4: the database itself refuses

The platform console has a SQL tool for support work. Free-form SQL goes around the application completely, so that
tool connects as a separate read-only role, and PostgreSQL row-level security does the fencing. Each migration that
adds a business table carries this block:

```sql
GRANT SELECT ON wholesale_offers TO bites_tenant_reader;
ALTER TABLE wholesale_offers ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_read ON wholesale_offers FOR SELECT TO bites_tenant_reader
  USING (business_id = current_setting('bites.business_id', true)::int);
```

The business is set per transaction. The role also has no access to password or token hashes at the column level,
and the tool only selects columns that role may read.

## The test that keeps it true

A rule that depends on people remembering is not a rule. This test asks the database for every table that has a
`business_id` column and no read policy, and expects the list to be empty:

```sql
select c.table_name from information_schema.columns c
where c.table_schema = 'public' and c.column_name = 'business_id'
  and not exists (select 1 from pg_policies p
                  where p.tablename = c.table_name and p.policyname = 'tenant_read')
```

Add a table without its policy and the build goes red, naming the table. The rest of the same test class reads like
a specification:

- `Each_business_has_its_own_menu_settings_and_order_numbers`
- `The_database_refuses_to_save_another_businesses_rows`
- `The_super_admin_works_in_one_business_at_a_time_and_its_sql_sees_only_that_one`
- `Turning_a_business_off_signs_everyone_out_and_stops_sign_ins`

## Crossing on purpose

A few things must look across businesses: signing in (usernames are unique platform-wide), the print worker, the
platform's own pages and the sync with store computers. Those call the one method that lifts the filter, by name, so
every crossing can be found with a single search and reviewed.

## Per-business behaviour without per-business code

- **Feature switches** live in one catalog: key, label, the routes and roles each guards. A business stores only the
  switches it changed, and everything defaults to on. A middleware answers 403 for a switched-off feature and the
  same catalog hides its navigation, so the API and the screens cannot disagree.
- **Settings** are jsonb columns on one row per business, with defaults in code. Adding a setting needs no migration.
- **Order numbers, menus, prices and loyalty cards** are all per business, each with its own sequence and prefix.

## What I would tell another engineer

Defence in depth sounds expensive, but each fence here is a few dozen lines written once. The expensive thing is the
fence that depends on every future query being written correctly, and that is the one this design does not have.
