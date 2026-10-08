# Osysharp.Warehouses.Snowflake — Snowflake as a data source and an export destination

Entities whose rows live in a Snowflake warehouse, read from an Osy# app with ordinary LINQ and secured by the app's
own rules — and an app's own entities sent into Snowflake tables as they change.

## Use case

Teams whose numbers already live in Snowflake — sales, product usage, finance — and who want an app on top of them
without copying the data out: an account page showing a customer's usage, an approval screen checking a spend against
the budget, a dashboard for regional managers that shows each one only their region. The app declares the tables it
reads as entities and queries them like any other entity. Each query runs in Snowflake as one statement that already
carries the reading person's access rule, so rows they may not see never leave the warehouse.

The other direction is for data the app owns and the warehouse should have: the orders, tickets or approvals people
create in the app, arriving in Snowflake minutes after they happen, ready to join against everything else there. The
app declares which entity's changes leave and which columns go with them; the platform delivers every change in commit
order, and each batch is one `MERGE` that a redelivery cannot undo.

It has no user interface of its own, so there is nothing to screenshot: the entities appear wherever your pages use
them.

## Install and use

```osy
// app.osy
app Revenue {
  model "**/*.osy";
  use Osysharp.Warehouses.Snowflake {
    egress "*.snowflakecomputing.com";
    secret "SnowflakePrivateKey";
  }
}
```
```osy
using Osysharp.Warehouses.Snowflake;

app.Secrets = [ new Secret("SnowflakePrivateKey") ];

app.DataSources = [
  new SnowflakeSource("Warehouse") {
    Account = "MYORG-MYACCOUNT", User = "OSY_READER", PublicKeyFingerprint = "SHA256:…",
    Warehouse = "ANALYTICS_WH", Database = "SALES", Schema = "PUBLIC", Role = "REPORTING"
  }
];

[DataSource(DataSource.Warehouse), ExternalName("PUBLIC.ORDERS")]
entity Order {
  [Key, ExternalName("ORDER_ID")] string Number;
  [ExternalName("REGION")] string Region;
  [ExternalName("TOTAL")] decimal Total;
  [ExternalName("PLACED")] DateOnly Placed;
  security { allow read where Region == user.Region; }
}
```

The app authenticates as a Snowflake user with a key pair. Create the key, register its public half on the user, and
read back the fingerprint `PublicKeyFingerprint` names:

```console
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out snowflake_key.p8 -nocrypt
openssl rsa -in snowflake_key.p8 -pubout -out snowflake_key.pub
```
```sql
ALTER USER OSY_READER SET RSA_PUBLIC_KEY = 'MIIBIjANBgkqh…';   -- the public key, without its BEGIN/END lines
DESC USER OSY_READER;                                          -- RSA_PUBLIC_KEY_FP is the fingerprint
```
```console
osy secret set SnowflakePrivateKey     # the contents of snowflake_key.p8
```

`Account` is the account identifier as your account's URL spells it — `MYORG-MYACCOUNT` for
`https://myorg-myaccount.snowflakecomputing.com`. An account locator with its region, `XY12345.eu-west-1`, also works.
Give the user a role that can read the tables your entities name and nothing else.

From then on `Order.Where(o => o.Total > 100).OrderBy(o => o.Placed).Take(20)` is one statement in Snowflake, and so
are `Count`, `Any`, `Sum`, `Average`, `Min`, `Max`, `GroupBy` (with a `.Where` on the groups), `Select(…).Distinct()`, arithmetic, text joined with `+`, `??` and `?:` on a row's
columns (`Order.Sum(o => o.Total * o.Quantity)`), a date's parts (`o.Placed.Year`, `DayOfWeek`, `DayOfYear`, `DayNumber`, `.Date`) and a date moved (`AddDays` — a fraction too — to `AddYears`) and `Math.Round` (to a number of digits too), `Floor`, `Ceiling`,
`Abs`, `Truncate`, `Sign`, `Min`, `Max`, `Pow`, `Sqrt` and `Clamp`, `Trim`, `TrimStart` and `TrimEnd` — given every character .NET calls whitespace, or the characters you name — and `Replace`, `IsEmpty`, `IsBlank` and `Like`, and `ToUpper`, `ToLower` and `Length` — case by the table the platform
sends with the query (.NET's, where this engine's own `UPPER` maps `ß` to `SS`), length in UTF-16 units. An `Average` is asked for as its sum and count
and divided the way C#'s `decimal` divides, so it carries every digit a `decimal` holds rather than Snowflake's rounding.
Conversions are one statement too — `Convert.ToInt`, `ToInt64`, `ToDecimal`, `ToDouble`, `ToBool` and `ToString`, and a
cast such as `(int)o.Total` — meaning what they mean in C#: toward zero, text read by the platform's number grammar or 0,
a null converted to 0, `false` or `""`, and a value with no answer failing the query. A whole number converted to a decimal becomes a `DECFLOAT`, so an average of whole numbers read inside a
condition — `Order.Where(o => Shipment.Where(…).Average(s => s.Count) > 3m)` — divides as decimals, never as whole numbers.
A decimal read from text is exact to
38 digits; whether Snowflake keeps its trailing zeros has not been measured against a live account. A double written as
text, text read as a double, and a double converted to a decimal are not translated.
`Where(…).Update(o => { … })` and `Where(…).Delete()` — with an `OrderBy(…).Take(n)` page before them too — run as one `UPDATE` or `DELETE`, under the reading person's read
rule and their update or delete rule, and answer the rows Snowflake's SQL API reports in `stats` (`SnowflakeSource`
implements `IWritableEntitySource`). `Other.Where(…).Insert(o => new Order { … })` runs as one `INSERT … SELECT` from another
table of the account — and `list.Insert(x => new Order { … })` as one from a `VALUES` table of the list's values — under the create rule of the entity inserted into, and answers the rows `stats` reports inserted.
The user the source connects as needs `UPDATE`, `DELETE` or `INSERT` on the tables written to.
`Union`, `Concat`, `Intersect` and `Except` of two reads of one entity run as one statement, each read under its own rule.
`Join`, `LeftJoin` and `SelectMany` between entities of the one account run as one statement too, the joined entity under its own rule.
A date written as text — `"at " + o.Placed`, `o.Placed.ToString("D")` — is one statement too: the platform sends the
culture's pattern as date parts, names and literals, so the text is C#'s, in the viewer's culture, and never Snowflake's
own.

Two entities of the same source can reference each other. The reference's `ExternalName` names the column holding the
target's key, and the target declares exactly one `[Key]`:

```osy
[DataSource(DataSource.Warehouse), ExternalName("PUBLIC.CUSTOMERS")]
entity Customer {
  [Key, ExternalName("CODE")] string Code;
  [ExternalName("NAME")] string Name;
  [ExternalName("REGION")] string Region;
  security { allow read where Region == user.Region; }
}

// on Order:
  [ExternalName("CUSTOMER_CODE")] Customer? Customer;
```

`Order.Where(o => o.Customer.Name == "Acme").Select(o => new { o.Number, Customer = o.Customer.Name })` is then one
statement with a `LEFT JOIN`, and so is ordering, grouping or aggregating through the reference. The join reads only the
customers the reading person may read, so an order whose customer they may not read keeps its row and reads the
customer as `null`; reading `order.Customer` in memory goes back to Snowflake by the customer's key under the
same rule. A reference to an entity in another source, or in the platform's own storage, is refused when the app
compiles.

### Export changes into Snowflake

```osy
using Osysharp.Warehouses.Snowflake;

app.Exports = [
  new SnowflakeExport("OrdersToWarehouse") {
    Account = "MYORG-MYACCOUNT", User = "OSY_LOADER", PublicKeyFingerprint = "SHA256:…",
    Warehouse = "LOAD_WH", Database = "SALES", Schema = "APP_EXPORTS", Role = "LOADER",
    Table = "ORDERS",
    Rows = Order.Select(o => new { o.Number, o.Customer, o.Total, o.PlacedAt }),
  }
];
```

The export authenticates exactly as a source does — the same key pair, the same secret — and the settings mean the
same. Give its user a role that can create a table in the schema and write to it:

```sql
GRANT USAGE ON WAREHOUSE LOAD_WH TO ROLE LOADER;
GRANT USAGE ON DATABASE SALES TO ROLE LOADER;
GRANT USAGE, CREATE TABLE ON SCHEMA SALES.APP_EXPORTS TO ROLE LOADER;
```

The table is created when the export first delivers, and a column added to `Rows` later is added to it. It holds one row
per exported row: `Id` (the row's id, its key), the exported columns, `_SEQUENCE` — the change that last wrote the row —
and `_DELETED`. Types follow the column: text as `TEXT`, whole numbers as `NUMBER(38,0)`, a `decimal` as
`NUMBER(38,18)`, instants as `TIMESTAMP_NTZ` or `TIMESTAMP_TZ`, an id as `TEXT`.

Every batch is one `MERGE`: a row is written only when its change is newer than the one the table already holds, so a
batch delivered twice — which delivery allows — changes nothing the second time. A delete sets `_DELETED` and keeps the
row, so an older change delivered after it cannot bring the row back. Read the table through a view that leaves those
rows out:

```sql
CREATE VIEW SALES.APP_EXPORTS.ORDERS_CURRENT AS
  SELECT * EXCLUDE ("_SEQUENCE", "_DELETED") FROM SALES.APP_EXPORTS.ORDERS WHERE NOT "_DELETED";
```

## What it ships

| file | what it is |
|---|---|
| `model/SnowflakeConnection.osy` | `abstract class SnowflakeConnection` — the account settings, key-pair authentication and the statement client a source and an export share |
| `model/SnowflakeSource.osy` | `class SnowflakeSource : SnowflakeConnection, IEntitySource` — runs a query plan through Snowflake's SQL API and hands back its rows; describes a database's tables |
| `model/SnowflakeExport.osy` | `class SnowflakeExport : SnowflakeConnection, IExportSink` — creates and widens the export's table, and merges each batch into it |
| `model/SnowflakeSql.osy` | `class SnowflakeSql` — translates a query plan into one Snowflake statement with bound values |
| `model/SnowflakeValues.osy` | `class SnowflakeValues` — decodes the SQL API's value encodings (dates as day counts, timestamps as epoch seconds) |
| `model/SnowflakeResponses.osy` | the SQL API's response shapes |

What the statements guarantee:

- every value is a bound parameter, never text spliced into the SQL;
- strings compare, sort and group ordinally, whatever collation the account or table defaults to;
- null sorts as C# sorts it — first ascending, last descending — and `StartsWith`, `EndsWith` and `Contains` match their text literally;
- distinct rows compare strings exactly and count `null` as one value, as C#'s `Distinct()` does;
- a computed value means what it does in C#: whole numbers divide truncating toward zero, a remainder keeps the sign of
  the number divided, and arithmetic on `null` is `null`;
- the reading person's access rule is part of the same `WHERE` as the query's own filter, so paging counts only rows
  they may read;
- a join through a reference is a `LEFT JOIN` over only the target rows the reading person may read, so a row whose
  target they may not read stays, with the target's columns `null`.

A statement Snowflake reports as still running is checked again after a pause that doubles from a quarter of a second
to eight seconds, until it finishes or Snowflake cancels it at `TimeoutSeconds`. A large result is read partition by
partition as it is consumed. A request Snowflake refuses throws
`ExternalServiceException` with Snowflake's own message.

`DescribeSchema` lists the tables in `Database` (in `Schema` alone when one is set) with each column's type and each
table's primary key. Columns of a type a query plan cannot carry — `VARIANT`, `OBJECT`, `ARRAY`, `BINARY`, `GEOMETRY`,
`VECTOR` — are left out. A `NUMBER` with no scale is described as `long`, because Snowflake's `INTEGER` is
`NUMBER(38,0)`; declare the property as `decimal` instead if its values can exceed a `long`.

## Build your own on top

**Another account, database or role** needs no change to the package: register one `SnowflakeSource` per name, and
point each entity at the one it lives in; declare one `SnowflakeExport` per table the app fills.

**Another warehouse** is an `IEntitySource` of your own, and this package is the worked example of one: a translator
from the query plan to the warehouse's SQL, a client for its API over `Http.*`, and the hosts and secrets it needs
declared in its manifest. Fork this repository into your GitHub account or organisation, rename the package to match
(`package Acme.Warehouse` lives at `github.com/acme/warehouse`), change the translator's dialect and the client's
endpoints, and `osy publish`. The query plan conformance cases — every plan, and the rows it must return — are what to
hold your translator to.
