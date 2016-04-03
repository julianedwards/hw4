# Homework 4

* Assigned: 4/5
* Due: 8:40 AM on 4/14, [via Courseworks](https://courseworks2.columbia.edu/courses/6866/assignments/29438)
* Worth 3.75% of your grade


In this homework, you will practice causing SQL injections, and study PostgreSQL's 
cost estimation and critique its choices. Please record your answers in a plain text file, and either paste your answers into [the Courseworks text box, or upload a plain text file](https://courseworks2.columbia.edu/courses/6866/assignments/29438).

Some useful references include

* [A presentation about PostgreSQL's EXPLAIN command](http://jberkus.github.io/explain_explained/index.html)
* [the PostgreSQL EXPLAIN documentation](http://www.postgresql.org/docs/9.3/static/sql-explain.html)
* [the PostgreSQL row estimation documentation](http://www.postgresql.org/docs/8.3/static/row-estimation-examples.html)

# 1. SQL Injection

*Q1.1: (5 points):* Consider the following code in a web server:

```python
def buggy_sanitize_1(input):
  return input.replace(';', '\;')

input = request.form['input']
query = "SELECT 1 WHERE 1 = 0 and %s;" % buggy_sanitize_1(input)
cursor = conn.execute(query)
print cursor.rowcount
```

Note: `buggy_sanitize_1` replaces semicolons with backslash followed by a semicolon, in an attempt to ensure correct behavior.

*Construct a value for `input` that would cause the query to return 1 row.*


*Q2.2: (5 points):* Consider the following code in a web server:

```python
def buggy_sanitize_2(input):
  return input.replace(';', '\;').replace("\'", "\\'")

input = request.form['input']
query = "SELECT 1 WHERE 'foo' = '%s';" % buggy_sanitize_2(input)
cursor = conn.execute(query)
print cursor.fetchall()[0][0]
```

Note: `buggy_sanitize_2` replaces semicolons with backslash followed by a semicolon, then replaces single quotes with backslash followed by a single quote. The extra backslashes are there to escape quotes and semicolons, in an attempt to ensure correct behavior.

*Construct a value for `input` that will cause the `cursor` to return the value `2` instead of `1`.*


# 2.  Indexing

**(3 points each, 21 points total)**

In this part of the problem set, you will examine query plans that PostgreSQL uses to execute queries, and try to understand
why it produces the plan it does for a certain query.

The data set you will use has the same schema as the `iowa` dataset. Rather than using SQLite,
you will be connecting to our PostgreSQL server, since PostgreSQL produces more interesting
query plans than SQLite.  To connect to the server we host, run the following command:

    psql -U <your uni> -h w4111db.eastus.cloudapp.azure.com iowa

**NOTE: The iowa table is fairly large with lots of rows, so please try not to run too many generic queries like “SELECT * FROM iowa”. They take a long time to execute, and slow down the database for everyone else. Hit Ctrl-C if you accidentally ran a query like this. EXPLAINs are fine since they don't actually execute the queries. When running a query, always use LIMIT clauses and/or selection filters to reduce the number of rows produced.**

To understand what query plan is being used, PostgreSQL includes the `EXPLAIN` command. 
It prints the plan for a query, including all of the physical operators and access methods being used. 
For example, the following SQL command displays the query plan for the SELECT:

    EXPLAIN SELECT  * FROM iowa WHERE vendor = '';

The result is a textual representation of the query plan 

                                        QUERY PLAN
    ------------------------------------------------------------------------------------
    Bitmap Heap Scan on iowa  (cost=100.97..12753.11 rows=3683 width=261)
      Recheck Cond: ((vendor)::text = ''::text)
      ->  Bitmap Index Scan on iowa_vendor  (cost=0.00..100.05 rows=3683 width=0)
            Index Cond: ((vendor)::text = ''::text)


For example, this is a query plan with no branches.  It first runs a `Bitmap Index Scan` using
the index `iowa_vendor_tree`, which is a Btree index, and the condition `vendor = ''` The results
are then fed into a `Bitmap Heap Scan`, which gathers all the tuple ids from the index scan together,
sorts the tuple ids by the pages the tuples are stored in, and reads the data pages as a single scan 
while rechecking the vendor condition.

Don't worry about the heap scan too much. We mainly care that the query uses the `iowa_vendor_tree` index.
You should also keep in mind that leaves of the BTree index do not store actual tuples (i.e. it is a 
secondary index, not a primary index).

[Click here for a pretty good presentation about EXPLAIN](http://jberkus.github.io/explain_explained/index.html)

We have created a number of additional indexes in the database.  You can see by typing `\d iowa` 
in PostgreSQL:

    Indexes:
      "iowa_cat_hash" hash (category)
      "iowa_date" btree (date)
      "iowa_dt_store_item_vendor_tree" btree (date, store, item, vendor)
      "iowa_store_hash" hash (store)
      "iowa_store_item_vendor_dt_tree" btree (store, item, vendor, date)
      "iowa_store_tree" btree (store)
      "iowa_vendor_hash" hash (vendor)
      "iowa_vendor_tree" btree (vendor)
      "iowa_zip_hash" hash (zipcode)
      "iowa_zip_tree" btree (zipcode)

We can see that several attributes have both a tree and hash index. We will try to explore when each is picked. 
Note that all of these indexes are secondary. There is no primary index here. 
You will want to use [the PostgreSQL EXPLAIN documentation](http://www.postgresql.org/docs/9.3/static/sql-explain.html).

1. **Q3**: Run `EXPLAIN` on the following query and explain in your own words (in a few sentences) the query plan that PostgreSQL picked (we are expecting something similar to the given example above).

    EXPLAIN SELECT  * FROM iowa WHERE store = 1000;

1. **Q4**: Explain why PostgreSQL picked that particular plan, based on statistics and our discussion in lecture (Hint: run `set enable_bitmapscan = 0` in PostgreSQL and run the explain command again. What changed? Why does using the hash index cost more than using the BTree index? Don’t forget to run `set enable_bitmapscan = 1` to re-enable bitmap scan after you're done with this question).

  A little bit of background: Bitmap Scans are very different from plain Index Scans. An Index Scan simply scans through an index and fetches tuples that satisfy a condition one by one. Thus each tuple would incur a separate access to a data page. On the other hand, a Bitmap Scan collects all pointers to tuples matching a condition, sorts them by the data pages they are stored in, and then fetches those tuples in one go. This improves the locality of page accesses. For some reason, PostgreSQL does not allow Bitmap Scans with hash indexes (you can assume this without explanation). As a result, when Bitmap Scans are disabled, using the hash index outperforms using the BTree index.

1. **Q5**: What did PostgreSQL estimate the number of resulting rows to be and what is the actual number of rows?  
   Why is there a difference?

1. **Q6**: Run `EXPLAIN` on the slightly different query below.  What index does the query use and why is
   it the same or different than the result of Q3?

    EXPLAIN SELECT  * FROM iowa WHERE store = 1000 LIMIT 1;

1. **Q7**: If we increase the `LIMIT` clause from **Q6** to a higher number (e.g., 200), 
    what type of index does PostgreSQL use now and why (Hint: this might be related to Q4)?

    EXPLAIN SELECT  * FROM iowa WHERE store = 1000 LIMIT 200;

1. **Q8**: Run `EXPLAIN` on the following slightly different query.  Why does the database choose this plan? Also what would have happened if PostgreSQL chose to use the BTree index?

    EXPLAIN SELECT  * FROM iowa WHERE store > 1000;

1. **Q9**: The following query _should_ use the same plan as Q6:

    EXPLAIN SELECT  * FROM iowa WHERE 1000 <= store AND store < 1001 LIMIT 1;

  However it doesn't and instead uses a BTree index.   Why do you think this is the case? What would the optimizer do if it were smarter?
