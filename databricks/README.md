# Osysharp.Warehouses.Databricks — Databricks as a data source and an export destination

Entities whose rows live in a Databricks SQL warehouse, read from an Osy# app with ordinary LINQ and secured by the
app's own rules — and an app's own entities sent into Databricks tables as they change.

## Use case

Teams whose data already lives in a Databricks lakehouse — sales, usage, operations — and who want an app on top of it
without copying the data out: a customer page that shows that customer's consumption, an approval screen that checks a
request against last quarter's actuals, a dashboard where each regional manager sees only their region. The app
declares the tables it reads as entities and queries them like any other entity. Each query runs on your SQL warehouse
as one statement that already carries the reading person's access rule, so rows they may not see never leave the
warehouse.

The other direction is for data the app owns and the lakehouse should have: the orders, tickets or approvals people
create in the app, arriving in a Delta table minutes after they happen, ready to join against everything else there. The
app declares which entity's changes leave and which columns go with them; the platform delivers every change in commit
order, and each batch is one `MERGE` that a redelivery cannot undo.

It has no user interface of its own, so there is nothing to screenshot: the entities appear wherever your pages use
them.

## Install and use

```osy
// app.osy
app Revenue {
  model "**/*.osy";
  use Osysharp.Warehouses.Databricks {
    egress "*.cloud.databricks.com";
    egress "*.azuredatabricks.net";
    egress "*.gcp.databricks.com";
    secret "DatabricksToken";
  }
}
```
```osy
using Osysharp.Warehouses.Databricks;

app.Secrets = [ new Secret("DatabricksToken") ];

app.DataSources = [
  new DatabricksSource("Lakehouse") {
    Host = "adb-1234567890123456.7.azuredatabricks.net", WarehouseId = "abcdef1234567890",
    Catalog = "main", Schema = "sales"
  }
];

[DataSource(DataSource.Lakehouse), ExternalName("sales.orders")]
entity Order {
  [Key, ExternalName("order_id")] string Number;
  [ExternalName("region")] string Region;
  [ExternalName("total")] decimal Total;
  [ExternalName("placed")] DateOnly Placed;
  security { allow read where Region == user.Region; }
}
```
```console
osy secret set DatabricksToken     # a personal access token, or an OAuth token for a service principal
```

`Host` is your workspace's host name, as its URL spells it. `WarehouseId` is the SQL warehouse's id, shown in the
warehouse's connection details. Give the identity behind the token `CAN USE` on the warehouse and `SELECT` on the tables
your entities name — and `MODIFY` on the ones the app updates or deletes from — and nothing else.

From then on `Order.Where(o => o.Total > 100).OrderBy(o => o.Placed).Take(20)` is one statement on the warehouse, and so
are `Count`, `Any`, `Sum`, `Average`, `Min`, `Max`, `GroupBy` (with a `.Where` on the groups), `Select(…).Distinct()`, arithmetic, text joined with `+`, `??` and `?:` on a row's
columns (`Order.Sum(o => o.Total * o.Quantity)`), a date's parts (`o.Placed.Year`), `AddDays` to `AddYears`, `.Date`,
`DayOfWeek`, `DayOfYear` and `DayNumber`, `Math.Round` (to a number of digits too), `Floor`, `Ceiling`, `Abs`, `Truncate`, `Sign`,
`Min`, `Max`, `Pow`, `Sqrt` and `Clamp`, `Trim`, `TrimStart` and `TrimEnd` — given every character .NET calls
whitespace — `Replace`, `IsNullOrEmpty`, `IsNullOrWhiteSpace`, `Like`, `ToUpper` and `ToLower` by .NET's own case map
(this engine's own mapping is the full one), `Length`, counted in UTF-16 units as .NET counts it, `Convert.ToString`,
`Convert.ToBoolean`, `Convert.ToDouble` and the numeric casts. `Convert.ToInt32`, `Convert.ToInt64` and
`Convert.ToDecimal` are refused when the app compiles: they read text into C#'s `decimal`, whose scale is whatever the
text says, and a Databricks `DECIMAL` has one fixed scale.

Each of these is one statement too: `Where(…).Update(o => { … })` and `Where(…).Delete()` (an `OrderBy(…).Take(n)` page of them too), under the reading person's
access rule and the update or delete rule; `Other.Where(…).Insert(o => new Order { … })` from another table of the
warehouse, or `list.Insert(x => new Order { … })` from a list the app holds, under the create rule of the entity inserted into; another entity read inside a condition (`Customer.Any(c => c.Code ==
o.CustomerCode)`), under that entity's own rule; `Union`, `Concat`, `Intersect` and `Except` of two reads of one
entity; a `Window.*` value in a projection; and `Join`, `LeftJoin` and `SelectMany` between entities of the one
warehouse, the joined entity under its own rule.
An `Average` is asked for as its sum and count and divided the way C#'s `decimal` divides, so it carries every digit a
`decimal` holds rather than the warehouse's rounding.

Two entities of the same source can reference each other. The reference's `ExternalName` names the column holding the
target's key, and the target declares exactly one `[Key]`:

```osy
[DataSource(DataSource.Lakehouse), ExternalName("sales.customers")]
entity Customer {
  [Key, ExternalName("code")] string Code;
  [ExternalName("name")] string Name;
  [ExternalName("region")] string Region;
  security { allow read where Region == user.Region; }
}

// on Order:
  [ExternalName("customer_code")] Customer? Customer;
```

`Order.Where(o => o.Customer.Name == "Acme").Select(o => new { o.Number, Customer = o.Customer.Name })` is then one
statement with a `LEFT JOIN`, and so is ordering, grouping or aggregating through the reference. The join reads only the
customers the reading person may read, so an order whose customer they may not read keeps its row and reads the
customer as `null`; reading `order.Customer` in memory goes back to the warehouse by the customer's key under the
same rule. A reference to an entity in another source, or in the platform's own storage, is refused when the app
compiles.

### Export changes into Databricks

```osy
using Osysharp.Warehouses.Databricks;

app.Exports = [
  new DatabricksExport("OrdersToLakehouse") {
    Host = "adb-1234567890123456.7.azuredatabricks.net", WarehouseId = "abcdef1234567890",
    Catalog = "main", Schema = "app_exports", Table = "orders",
    Rows = Order.Select(o => new { o.Number, o.Customer, o.Total, o.PlacedAt }),
  }
];
```

The export authenticates exactly as a source does — the same `DatabricksToken` secret — and the settings mean the same.
Give the token's principal the right to create a table in the schema and to write to it:

```sql
GRANT USE CATALOG ON CATALOG main TO `osy-loader`;
GRANT USE SCHEMA, CREATE TABLE, MODIFY, SELECT ON SCHEMA main.app_exports TO `osy-loader`;
```

The table is created when the export first delivers, and a column added to `Rows` later is added to it. It holds one row
per exported row: `Id` (the row's id, its key), the exported columns, `_sequence` — the change that last wrote the row —
and `_deleted`. Types follow the column: text as `STRING`, whole numbers as `BIGINT`, a `decimal` as `DECIMAL(38,18)`,
a `DateTime` as `TIMESTAMP_NTZ`, an id as `STRING`. Databricks has no time-of-day type and its `TIMESTAMP` holds no
offset, so a `TimeOnly` is kept as its text and a `DateTimeOffset` as its instant.

Every batch is one `MERGE`: a row is written only when its change is newer than the one the table already holds, so a
batch delivered twice — which delivery allows — changes nothing the second time. A delete sets `_deleted` and keeps the
row, so an older change delivered after it cannot bring the row back. Read the table through a view that leaves those
rows out:

```sql
CREATE VIEW main.app_exports.orders_current AS
  SELECT * EXCEPT (_sequence, _deleted) FROM main.app_exports.orders WHERE NOT _deleted;
```

## What it ships

| file | what it is |
|---|---|
| `model/DatabricksConnection.osy` | `abstract class DatabricksConnection` — the workspace settings, the bearer token and the statement client a source and an export share |
| `model/DatabricksSource.osy` | `class DatabricksSource : DatabricksConnection, IWritableEntitySource` — runs a query plan through the SQL Statement Execution API and hands back its rows, or runs an update or delete and answers the `num_affected_rows` the warehouse reports; describes a catalog's tables |
| `model/DatabricksExport.osy` | `class DatabricksExport : DatabricksConnection, IExportSink` — creates and widens the export's table, and merges each batch into it |
| `model/DatabricksSql.osy` | `class DatabricksSql` — translates a query plan into one Databricks SQL statement with named, typed parameters |
| `model/DatabricksResponses.osy` | the SQL Statement Execution API's response shapes |

What the statements guarantee:

- every value is a bound parameter, never text spliced into the SQL;
- strings compare, sort and group ordinally (`COLLATE UTF8_BINARY`), whatever collation a schema or table defaults to;
- null sorts as C# sorts it — first ascending, last descending — and `StartsWith`, `EndsWith` and `Contains` match their text literally;
- distinct rows compare strings exactly and count `null` as one value, as C#'s `Distinct()` does;
- a computed value means what it does in C#: whole numbers divide truncating toward zero, a remainder keeps the sign of
  the number divided, and arithmetic on `null` is `null`;
- the reading person's access rule is part of the same `WHERE` as the query's own filter, so paging counts only rows
  they may read;
- a join through a reference is a `LEFT JOIN` over only the target rows the reading person may read, so a row whose
  target they may not read stays, with the target's columns `null`;
- a join on a value, and a read of another entity inside a condition, read that entity under its own access rule;
- what C# throws on — `Math.Clamp` with its minimum above its maximum, `Replace` of an empty string, a number with no
  place in an `int` — fails the statement with `raise_error`, whatever the session's ANSI setting;
- a `TIMESTAMP` column arrives with its UTC offset, so it names the same instant whatever time zone the warehouse uses.

Results are requested inline as JSON and read chunk by chunk as they are consumed. A statement the warehouse reports as
still running is checked again after a pause that doubles from a quarter of a second to eight seconds; one still running
after `TimeoutSeconds` (600 by default) is cancelled, so it stops using the warehouse, and the read fails. A statement that fails throws
`ExternalServiceException` with the warehouse's own message.

`DescribeSchema` lists the tables in `Catalog` (in `Schema` alone when one is set) with each column's type and each
table's primary key. Columns of a type a query plan cannot carry — `BINARY`, `ARRAY`, `MAP`, `STRUCT`, `VARIANT`,
`INTERVAL` — are left out. A `TIMESTAMP` column is described as `DateTimeOffset` and a `TIMESTAMP_NTZ` one as
`DateTime`.

The ordinal collation needs a runtime that supports `COLLATE`; a SQL warehouse on a current channel does.

## Build your own on top

**Another workspace, warehouse or catalog** needs no change to the package: register one `DatabricksSource` per name,
and point each entity at the one it lives in; declare one `DatabricksExport` per table the app fills.

**Another warehouse product** is an `IEntitySource` of your own, and this package is a worked example of one: a
translator from the query plan to the engine's SQL, a client for its API over `Http.*`, and the hosts and secrets it
needs declared in its manifest. Fork this repository into your GitHub account or organisation, rename the package to
match (`package Acme.Warehouse` lives at `github.com/acme/warehouse`), change the translator's dialect and the client's
endpoints, and `osy publish`. The query plan conformance cases — every plan, and the rows it must return — are what to
hold your translator to.
