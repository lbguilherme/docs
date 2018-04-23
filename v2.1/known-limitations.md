---
title: Known Limitations in CockroachDB v2.1
summary: Known limitations in CockroachDB v2.1.
toc: false
---

This page describes newly identified limitations in the CockroachDB v2.1 release as well as unresolved limitations identified in earlier releases.

<div id="toc"></div>

## New Limitations

None identified yet.

## Unresolved Limitations

### Enterprise backup/restore during rolling upgrades

In the upgrade process, after upgrading all binaries to v2.1, it's recommended to monitor the cluster's stability and performance for at least one day and only then finalize the upgrade by increasing the `version` cluster setting. However, in the window during which binaries are running v2.1 but the cluster version is still not increased, it is not possible to run enterprise [`BACKUP`](backup.html) and [`RESTORE`](restore.html) jobs.

You can track this known limitation [here](https://github.com/cockroachdb/cockroach/issues/24490).

### Write and update limits for a single statement

A single statement can perform at most at most 64MiB of combined updates. When a statement exceeds these limits, its transaction gets aborted. `INSERT INTO ... SELECT FROM` queries commonly encounter these limits.

If you need to increase these limits, you can update the [cluster-wide setting](cluster-settings.html) `kv.raft.command.max_size`, but note that increasing this setting can affect the memory utilization of nodes in the cluster. For `INSERT INTO .. SELECT FROM` queries in particular, another workaround is to manually page through the data you want to insert using separate transactions.

In the v1.1 release, the limit referred to a whole transaction (i.e., the sum of changes done by all statements) and capped both the number and the size of update. In this release, there's only a size limit, and it applies independently to each statement. Note that even though not directly restricted
any more, large transactions can have performance implications on the cluster.

### Memory flags with non-integer values and a unit suffix

The `--cache` and `--max-sql-memory` flags of the [`cockroach start`](start-a-node.html) command do not support non-integer values with a unit suffix, for example, `--cache=1.5GiB`.

As a workaround, use integer values or a percentage, for example, `--cache=1536MiB`.

### Import with a high amount of disk contention

[`IMPORT`](import.html) can sometimes fail with a "context canceled" error, or can restart itself many times without ever finishing. If this is happening, it is likely due to a high amount of disk contention. This can be mitigated by setting the `kv.bulk_io_write.max_rate` [cluster setting](cluster-settings.html) to a value below your max disk write speed. For example, to set it to 10MB/s, execute:

~~~ sql
> SET CLUSTER SETTING kv.bulk_io_write.max_rate = '10MB';
~~~

### Check constraints with `INSERT ... ON CONFLICT`

[`CHECK`](check.html) constraints are not properly enforced on updated values resulting from [`INSERT ... ON CONFLICT`](insert.html) statements. Consider the following example:

~~~ sql
> CREATE TABLE ab (a INT PRIMARY KEY, b INT, CHECK (b < 1));
~~~

A simple `INSERT` statement that fails the Check constraint fails as it should:

~~~ sql
> INSERT INTO ab (a,b) VALUES (1, 12312);
~~~

~~~
pq: failed to satisfy CHECK constraint (b < 1)
~~~

However, the same statement with `INSERT ... ON CONFLICT` incorrectly succeeds and results in a row that fails the constraint:

~~~ sql
> INSERT INTO ab (a, b) VALUES (1,0); -- create some initial valid value
~~~

~~~ sql
> INSERT INTO ab (a, b) VALUES (1,0) ON CONFLICT (a) DO UPDATE SET b = 123132;
~~~

~~~ sql
> SELECT * FROM ab;
~~~

~~~
+---+--------+
| a |   b    |
+---+--------+
| 1 | 123132 |
+---+--------+
(1 row)
~~~

### Referring to a CTE by name more than once

{% include known_limitations/cte-by-name.md %}

### Using CTEs with data-modifying statements

{% include known_limitations/cte-with-dml.md %}

### Using CTEs with views

{% include known_limitations/cte-with-view.md %}

### Using CTEs with `VALUES` clauses

{% include known_limitations/cte-in-values-clause.md %}

### Using CTEs with Set Operations

{% include known_limitations/cte-in-set-expression.md %}

### Assigning latitude/longitude for the Node Map

{% include known_limitations/node-map.md %}

### Placeholders in `PARTITION BY`

{% include known_limitations/partitioning-with-placeholders.md %}

### Adding a column with certain `DEFAULT` values

It is currently not possible to [add a column](add-column.html) to a table when the column uses a [sequence](create-sequence.html), [computed column](computed-columns.html), or certain evaluated expressions as the [`DEFAULT`](default-value.html) value, for example:

~~~ sql
> ALTER TABLE add_default ADD g INT DEFAULT nextval('initial_seq')
~~~

~~~ sql
> ALTER TABLE add_default ADD g OID DEFAULT 'foo'::regclass::oid
~~~

~~~ sql
> ALTER TABLE add_default ADD g INT DEFAULT 'foo'::regtype::INT
~~~

### Available capacity metric in the Admin UI

{% include available-capacity-metric.md %}

### Schema changes within transactions

Within a single [transaction](transactions.html):

- DDL statements cannot follow DML statements. As a workaround, arrange DML statements before DDL statements, or split the statements into separate transactions.
- A [`CREATE TABLE`](create-table.html) statement containing [`FOREIGN KEY`](foreign-key.html) or [`INTERLEAVE`](interleave-in-parent.html) clauses cannot be followed by statements that reference the new table.
- A table cannot be dropped and then recreated with the same name. This is not possible within a single transaction because `DROP TABLE` does not immediately drop the name of the table. As a workaround, split the [`DROP TABLE`](drop-table.html) and [`CREATE TABLE`](create-table.html) statements into separate transactions.

### Schema changes between executions of prepared statements

When the schema of a table targeted by a prepared statement changes before the prepared statement is executed, CockroachDB allows the prepared statement to return results based on the changed table schema, for example:

{% include copy-clipboard.html %}
~~~ sql
> CREATE TABLE users (id INT PRIMARY KEY);
~~~

{% include copy-clipboard.html %}
~~~ sql
> PREPARE prep1 AS SELECT * FROM users;
~~~

{% include copy-clipboard.html %}
~~~ sql
> ALTER TABLE users ADD COLUMN name STRING;
~~~

{% include copy-clipboard.html %}
~~~ sql
> INSERT INTO users VALUES (1, 'Max Roach');
~~~

{% include copy-clipboard.html %}
~~~ sql
> EXECUTE prep1;
~~~

~~~
+----+-----------+
| id |   name    |
+----+-----------+
|  1 | Max Roach |
+----+-----------+
(1 row)
~~~

It's therefore recommended to **not** use `SELECT *` in queries that will be repeated, via prepared statements or otherwise.

Also, a prepared [`INSERT`](insert.html), [`UPSERT`](upsert.html), or [`DELETE`](delete.html) statement acts inconsistently when the schema of the table being written to is changed before the prepared statement is executed:

- If the number of columns has increased, the prepared statement returns an error but nonetheless writes the data.
- If the number of columns remains the same but the types have changed, the prepared statement writes the data and does not return an error.

### `INSERT ON CONFLICT` vs. `UPSERT`

When inserting/updating all columns of a table, and the table has no secondary indexes, we recommend using an [`UPSERT`](upsert.html) statement instead of the equivalent [`INSERT ON CONFLICT`](insert.html) statement. Whereas `INSERT ON CONFLICT` always performs a read to determine the necessary writes, the `UPSERT` statement writes without reading, making it faster.

This issue is particularly relevant when using a simple SQL table of two columns to [simulate direct KV access](frequently-asked-questions.html#can-i-use-cockroachdb-as-a-key-value-store). In this case, be sure to use the `UPSERT` statement.

### Using `\|` to perform a large input in the SQL shell

In the [built-in SQL shell](use-the-built-in-sql-client.html), using the [`\|`](use-the-built-in-sql-client.html#sql-shell-commands) operator to perform a large number of inputs from a file can cause the server to close the connection. This is because `\|` sends the entire file as a single query to the server, which can exceed the upper bound on the size of a packet the server can accept from any client (16MB).

As a workaround, [execute the file from the command line](use-the-built-in-sql-client.html#execute-sql-statements-from-a-file) with `cat data.sql | cockroach sql` instead of from within the interactive shell.

### New values generated by `DEFAULT` expressions during `ALTER TABLE ADD COLUMN`

When executing an [`ALTER TABLE ADD COLUMN`](add-column.html) statement with a [`DEFAULT`](default-value.html) expression, new values generated:

- use the default [search path](sql-name-resolution.html#search-path) regardless of the search path configured in the current session via `SET SEARCH_PATH`.
- use the UTC time zone regardless of the time zone configured in the current session via [`SET TIME ZONE`](set-vars.html).
- have no default database regardless of the default database configured in the current session via [`SET DATABASE`](set-vars.html), so you must specify the database of any tables they reference.
- use the transaction timestamp for the `statement_timestamp()` function regardless of the time at which the `ALTER` statement was issued.

### Load-based lease rebalancing in uneven latency deployments

When nodes are started with the [`--locality`](start-a-node.html#flags-changed-in-v2-0) flag, CockroachDB attempts to place the replica lease holder (the replica that client requests are forwarded to) on the node closest to the source of the request. This means as client requests move geographically, so too does the replica lease holder.

However, you might see increased latency caused by a consistently high rate of lease transfers between datacenters in the following case:

- Your cluster runs in datacenters which are very different distances away from each other.
- Each node was started with a single tier of `--locality`, e.g., `--locality=datacenter=a`.
- Most client requests get sent to a single datacenter because that's where all your application traffic is.

To detect if this is happening, open the [Admin UI](admin-ui-access-and-navigate.html), select the **Queues** dashboard, hover over the **Replication Queue** graph, and check the **Leases Transferred / second** data point. If the value is consistently larger than 0, you should consider stopping and restarting each node with additional tiers of locality to improve request latency.

For example, let's say that latency is 10ms from nodes in datacenter A to nodes in datacenter B but is 100ms from nodes in datacenter A to nodes in datacenter C. To ensure A's and B's relative proximity is factored into lease holder rebalancing, you could restart the nodes in datacenter A and B with a common region, `--locality=region=foo,datacenter=a` and `--locality=region=foo,datacenter=b`, while restarting nodes in datacenter C with a different region, `--locality=region=bar,datacenter=c`.

### Overload resolution for collated strings

Many string operations are not properly overloaded for [collated strings](collate.html), for example:

{% include copy-clipboard.html %}
~~~ sql
> SELECT 'string1' || 'string2';
~~~

~~~
+------------------------+
| 'string1' || 'string2' |
+------------------------+
| string1string2         |
+------------------------+
(1 row)
~~~

{% include copy-clipboard.html %}
~~~ sql
> SELECT ('string1' collate en) || ('string2' collate en);
~~~

~~~
pq: unsupported binary operator: <collatedstring{en}> || <collatedstring{en}>
~~~

### Max size of a single column family

When creating or updating a row, if the combined size of all values in a single [column family](column-families.html) exceeds the max range size (64MiB by default) for the table, the operation may fail, or cluster performance may suffer.

As a workaround, you can either [manually split a table's columns into multiple column families](column-families.html#manual-override), or you can [create a table-specific zone configuration](configure-replication-zones.html#create-a-replication-zone-for-a-table) with an increased max range size.

### Simultaneous client connections and running queries on a single node

When a node has both a high number of client connections and running queries, the node may crash due to memory exhaustion. This is due to CockroachDB not accurately limiting the number of clients and queries based on the amount of available RAM on the node.

To prevent memory exhaustion, monitor each node's memory usage and ensure there is some margin between maximum CockroachDB memory usage and available system RAM. For more details about memory usage in CockroachDB, see [this blog post](https://www.cockroachlabs.com/blog/memory-usage-cockroachdb/).

### SQL subexpressions and memory usage

Many SQL subexpressions (e.g., `ORDER BY`, `UNION`/`INTERSECT`/`EXCEPT`, `GROUP BY`, subqueries) accumulate intermediate results in RAM on the node processing the query. If the operator attempts to process more rows than can fit into RAM, the node will either crash or report a memory capacity error. For more details about memory usage in CockroachDB, see [this blog post](https://www.cockroachlabs.com/blog/memory-usage-cockroachdb/).

### Query planning for `OR` expressions

Given a query like `SELECT * FROM foo WHERE a > 1 OR b > 2`, even if there are appropriate indexes to satisfy both `a > 1` and `b > 2`, the query planner performs a full table or index scan because it can't use both conditions at once.

### Privileges for `DELETE` and `UPDATE`

Every [`DELETE`](delete.html) or [`UPDATE`](update.html) statement constructs a `SELECT` statement, even when no `WHERE` clause is involved. As a result, the user executing `DELETE` or `UPDATE` requires both the `DELETE` and `SELECT` or `UPDATE` and `SELECT` [privileges](privileges.html) on the table.

### `cockroach dump` does not support cyclic foreign key references

{% include known_limitations/dump-cyclic-foreign-keys.md %}