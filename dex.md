## Project description

The goal is establish a unified approach for extending, composing and accelerating data analysis capabilities derived from different data models and computing paradigms.

Part I: Extending SQL databases with in-memory computing.

Part II: Cross-engine hybrid query API, processing engine and orchestration middleware.

Part III: Hybrid query optimization using machine learning and program synthesis(?).

## Motivating examples for query push down

### Query 1

```
select a.trunc, count(*) from rand_seq a join rand_seq b on a.trunc = b.trunc where a.id < b.id group by a.trunc;
```

AST of this statement:

```
SELECT
  targetList
    column ('a', 'trunc')
    function
      count
  fromClause
    join
      left
        relation ('rand_seq' as 'a')
      right
        relation ('rand_seq' as 'b')
      quals
        =
          lexpr
            column ('a', 'trunc')
          rexpr
            column ('b', 'trunc')
  whereClause
    <
      lexpr
        column ('a', 'id')
      rexpr
        column ('b', 'id')
  groupClause
    column ('a', 'trunc')

END
```

To offload this query to a remote system while preseving all the original declarative semantics, we need to

1) identify the input data to be migrated or imported to the remote system

2) migrate the input data to the backend system in the compatible format

3) rename the input, construct an identical query using the target query language

4) send the result data back

To execute the above query in Spark, we could take the following four steps:

The remote system module should implement the data rename and migration interface.

1) the only input data involved in the query is rand_seq

2) in reponse of the renaming and migration request , the Spark adapter constructs a JDBCRDD for rand_seq, creates temp view by renaming it as `rs`

COMMAND (MIGRATE-RENAME, brand_seq', 'rs')

there are two data migration styles: push and pull. For lazy execution systems like Spark, a pull style data migration is more plausible than push migration. Data pulling happens after the remote query starts. 

3) 

COMMAND (QUERY, "a = alias(rs);
                 b = alias(rs);
                 v1 = join(a, b);
                 v2 = filter(v1, 'a.trunc = b.trunc');
                 v3 = filter(v2, 'a.id < b.id');
                 v4 = group(v3, 'a.trunc', count);
                 v5 = format(v4, 'a.trunc', 'count'")

Consider building an intermediate language to facilitate the query translation between the base and remote systems.

