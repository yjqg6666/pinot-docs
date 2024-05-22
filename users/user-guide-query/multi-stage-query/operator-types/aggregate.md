---
description: >-
  Describes the aggregate relation operator in the multi-stage query engine.
---

# Aggregate relational operator

The aggregate operator is used to perform calculations on a set of rows and return a single row of results.

This page describes the aggregate operator defined in the relational algebra used by multi-stage queries.
This operator is generated by the multi-stage query engine when you use _aggregate functions_ in a query either with or
without a `group by` clause.

## Implementation details
Aggregate operations may be expensive in terms of memory, CPU and network usage. 
As explained in [understanding stages](../understanding-stages.md), the multi-stage query engine breaks down the query 
into multiple stages and each stage is then executed in parallel on different _workers_.
Each worker processes a subset of the data and sends the results to the _coordinator_ which then aggregates the results.
When possible, the multi-stage query engine will try to apply a divide-and-conquer strategy to reduce the amount of data
that needs to be processed in the _coordinator_ stage.

For example if the aggregation function is a sum, the engine will try to sum the results of each worker before sending
the partial result to the _coordinator_, which would then sum the partial results in order to get the final result.
But some aggregation functions, like `count(distinct)`, cannot be computed in this way and require all the data to be
processed in the _coordinator_ stage.

In Apache Pinot 1.1.0, the multi-stage query engine always keeps the data in memory.
This means that the amount of memory used by the engine is proportional to the number of groups generated by the
`group by` clause and the amount of data that needs to be kept for each group (which depends on the aggregation 
function).

Even when the aggregation function is a simple `count`, which only requires to keep a long for each group in memory,
the amount of memory used can be high if the number of groups is high.
This is why the engine limits the number of groups.
By default, this limit is 100.000, but this can be changed by providing hints.

### Blocking nature
The aggregate operator is a blocking operator. It needs to consume all the input data before emitting the result.

## Hints

### num_groups_limit
Type: Integer

Default: 100.000

Defines the max number of groups that can be created by the `group by` clause. 
If the number of groups exceeds this limit, the query will not fail but will stop the execution.
  
Example:
```sql
SELECT
/*+  aggOptions(num_groups_limit='10000000') */
    col1, count(*)
FROM table GROUP BY col1
```

### is_partitioned_by_group_by_keys
Type: Boolean 

Default: false

If set to true, the engine will consider that the data is already partitioned by the `group by` keys.
This means that the engine will not need to shuffle the data to group them by the `group by` keys and the
_coordinator_ stage will be able to compute the final result without needing to merge the partial results.

{% hint style="warning" %}
Caution: This hint should only be used if the data is already partitioned by the `group by` keys. 
There is no check to verify that the data is indeed partitioned by the `group by` keys and using this hint when the data
is not partitioned by the `group by` keys will lead to incorrect results.
{% endhint %}

Example:
```sql
SELECT 
/*+ aggOptions(is_partitioned_by_group_by_keys='true') */
    a.col3, a.col1, SUM(b.col3)
FROM a JOIN b ON a.col3 = b.col3 
GROUP BY a.col3, a.col1
```

### is_skip_leaf_stage_group_by
Type: Boolean

Default: false

If set to true, the engine will not push down the aggregate into the leaf stage.
In some situations, it could be wasted effort to do group-by on leaf, eg: when cardinality of group by column is very 
high.

Example:
```sql
SELECT 
/*+ aggOptions(is_skip_leaf_stage_group_by='true') */ 
    a.col1, SUM(a.col3) 
FROM a
WHERE a.col3 >= 0 AND a.col2 = 'a' 
GROUP BY a.col1
```

### max_initial_result_holder_capacity
Type: Integer

Default: 10.000

Defines the initial capacity of the result holder that stores the intermediate results of the aggregation.
This hint can be used to reduce the memory usage of the engine by setting a value close to the expected number of 
groups.
It is usually recommended to not change this hint unless you know that the expected number of groups is much lower
than the default value.

Example:
```sql
SELECT 
/*+ aggOptions(max_initial_result_holder_capacity='10') */ 
    a.col1, SUM(a.col3) 
FROM a
WHERE a.col3 >= 0 AND a.col2 = 'a' 
GROUP BY a.col1
```

## Stats
### executionTimeMs
Type: Long

The summation of time spent by all threads executing the operator.
This means that the wall time spent in the operation may be smaller that this value if the parallelism is larger than 1.
Remember that this value is affected by the number of received rows and the complexity of the aggregation function.

### emittedRows
Type: Long

The number of groups emitted by the operator.
A large number of emitted rows can indicate that the query is not well optimized.

Remember that the number of groups is limited by the `num_groups_limit` hint and a large number of groups can lead to
high memory usage and slow queries.

### numGroupsLimitReached
Type: Boolean

This stat is set to true when the number of groups exceeds the limit defined by the `num_groups_limit` hint.
In that case, the query will not fail but will return partial results, which will be indicated by the global 
`partialResponse` stat.

## Explain attributes
The aggregate operator is represented in the explain plan as a `LogicalAggregate` explain node.

Remember that these nodes appear in pairs: First in one stage where the aggregation is done in parallel and then in the
upstream stage where the partial results are merged.

### group
Type: List of Integer

The list of columns used in the `group by` clause.
These numbers are 0-based column indexes on the virtual row projected by the upstream.

For example the explain plan:
```
LogicalAggregate(group=[{6}], agg#0=[COUNT()], agg#1=[MAX($5)])
  LogicalTableScan(table=[[default, userAttributes]])
```

Is saying that the `group by` clause is using the column with index 6 in the virtual row projected by the upstream.
Given in this case that row is generated by a table scan, the column with index 6 is the 7th column in the table
as defined in its schema.

### agg#N
Type: Expression

The aggregation functions applied to the columns.
There may be multiple `agg#N` attributes, each one representing a different aggregation function.

For example the explain plan:
```
LogicalAggregate(group=[{6}], agg#0=[COUNT()], agg#1=[MAX($5)])
  LogicalTableScan(table=[[default, userAttributes]])
```

Has two aggregation functions: `COUNT()` and `MAX()`.
The second is applied to the column with index 5 in the virtual row projected by the upstream.
Given in this case that row is generated by a table scan, the column with index 5 is the 6th column in the table
as defined in its schema.

## Tips and tricks

### Try to not use aggregation functions that cannot be parallelized
For example, it is recommended to use one of the different hyperloglog flavor instead of `count(distinct)` when
the cardinality of the data or their size.

For example, it is cheaper to execute `count(distinct)` on an int column with 1000 distinct values than on a column that
stores very long strings, even if the number of distinct values is the same.

<!-- ### Use the `group by` clause and partition -->