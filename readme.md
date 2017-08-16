# Logical Database Design Anti-Patterns

## Multi-Valued Attribute
### **Store each value in its own column and row:**     
Storing a list of IDs as a VARCHAR/TEXT column can cause performance and data integrity
problems. Querying against such a column would require using pattern-matching
expressions. It is awkward and costly to join a comma-separated list to matching rows.
This will make it harder to validate IDs. Think about what is the greatest number of
entries this list must support? Instead of using a multi-valued attribute,
consider storing it in a separate table, so that each individual value of that attribute
occupies a separate row. Such an intersection table implements a many-to-many relationship
between the two referenced tables. This will greatly simplify querying and validating
the IDs.

## Recursive Dependency
### **Avoid recursive relationships:**   
It's common for data to have recursive relationships. Data may be organized in a
treelike or hierarchical way. However, creating a foreign key constraint to enforce
the relationship between two columns in the same table lends to awkward querying.
Each level of the tree corresponds to another join. You will need to issue recursive
queries to get all descendants or all ancestors of a node.
A solution is to construct an additional closure table. It involves storing all paths
through the tree, not just those with a direct parent-child relationship.
You might want to compare different hierarchical data designs -- closure table,
path enumeration, nested sets -- and pick one based on your application's needs.

## Primary Key Does Not Exist

### **Consider adding a primary key:**
A primary key constraint is important when you need to do the following:
prevent a table from containing duplicate rows,
reference individual rows in queries, and
support foreign key references
If you don't use primary key constraints, you create a chore for yourself:
checking for duplicate rows. More often than not, you will need to define
a primary key for every table. Use compound keys when they are appropriate.

## Generic Primary Key

### **Skip using a generic primary key (id):**   
Adding an id column to every table causes several effects that make its
use seem arbitrary. You might end up creating a redundant key or allow
duplicate rows if you add this column in a compound key.
The name id is so generic that it holds no meaning. This is especially
important when you join two tables and they have the same primary
key column name.

## Foreign Key Does Not Exist

### **Consider adding a foreign key:**   
Are you leaving out the application constraints? Even though it seems at
first that skipping foreign key constraints makes your database design
simpler, more flexible, or speedier, you pay for this in other ways.
It becomes your responsibility to write code to ensure referential integrity
manually. Use foreign key constraints to enforce referential integrity.
Foreign keys have another feature you can't mimic using application code:
cascading updates to multiple tables. This feature allows you to
update or delete the parent row and lets the database takes care of any child
rows that reference it. The way you declare the ON UPDATE or ON DELETE clauses
in the foreign key constraint allow you to control the result of a cascading
operation. Make your database mistake-proof with constraints.


## Entity-Attribute-Value Pattern


### **Dynamic schema with variable attributes:**  
Are you trying to create a schema where you can define new attributes
at runtime.? This involves storing attributes as rows in an attribute table.
This is referred to as the Entity-Attribute-Value or schemaless pattern.
When you use this pattern,  you sacrifice many advantages that a conventional
database design would have given you. You can't make mandatory attributes.
You can't enforce referential integrity. You might find that attributes are
not being named consistently. A solution is to store all related types in one table,
with distinct columns for every attribute that exists in any type
(Single Table Inheritance). Use one attribute to define the subtype of a given row.
Many attributes are subtype-specific, and these columns must
be given a null value on any row storing an object for which the attribute
does not apply the columns with non-null values become sparse.
Another solution is to create a separate table for each subtype
(Concrete Table Inheritance). A third solution mimics inheritance,
as though tables were object-oriented classes (Class Table Inheritance).
Create a single table for the base type, containing attributes common to
all subtypes. Then for each subtype, create another table, with a primary key
that also serves as a foreign key to the base table.
If you have many subtypes or if you must support new attributes frequently,
you can add a BLOB column to store data in a format such as XML or JSON,
which encodes both the attribute names and their values.
This design is best when you can't limit yourself to a finite set of subtypes
and when you need complete flexibility to define new attributes at any time.

## Metadata Tribbles

### **Store each value with the same meaning in a single column:**   
Creating multiple columns in a table indicates that you are trying to store
a multivalued attribute. This design makes it hard to add or remove values,
to ensure the uniqueness of values, and handling growing sets of values.
The best solution is to create a dependent table with one column for the
multivalue attribute. Store the multiple values in multiple rows instead of
multiple columns. Also, define a foreign key in the dependent table to associate
the values to its parent row.

### **Breaking down a table or column by year:**   
You might be trying to split a single column into multiple columns,
using column names based on distinct values in another attribute.
Each year, you will need to add one more column or table.
You are mixing metadata with data. You will now need to make sure that
the primary key values are unique across all the split columns or tables.
The solution is to use a feature called sharding or horizontal partitioning.
```(PARTITION BY HASH ( YEAR(...) )```. With this feature, you can gain the
benefits of splitting a large table without the drawbacks.
Partitioning is not defined in the SQL standard, so each brand of database
implements it in their own nonstandard way.
Another remedy for metadata tribbles is to create a dependent table.
Instead of one row per entity with multiple columns for each year,
use multiple rows. Don't let data spawn metadata.

# Physical Database Design Anti-Patterns

## Imprecise Data Type

### Use precise data types:   
Virtually any use of FLOAT, REAL, or DOUBLE PRECISION data types is suspect.
Most applications that use floating-point numbers don't require the range of
values supported by IEEE 754 formats. The cumulative impact of inexact 
floating-point numbers is severe when calculating aggregates.
Instead of FLOAT or its siblings, use the NUMERIC or DECIMAL SQL data types
for fixed-precision fractional numbers. These data types store numeric values
exactly, up to the precision you specify in the column definition.
Do not use FLOAT if you can avoid it.


## Values In Definition

### Don't specify values in column definition:   
With enum, you declare the values as strings,
but internally the column is stored as the ordinal number of the string
in the enumerated list. The storage is therefore compact, but when you
sort a query by this column, the result is ordered by the ordinal value,
not alphabetically by the string value. You may not expect this behavior.
There's no syntax to add or remove a value from an ENUM or check constraint
you can only redefine the column with a new set of values.
Moreover, if you make a value obsolete, you could upset historical data.
As a matter of policy, changing metadata — that is, changing the definition
of tables and columns—should be infrequent and with attention to testing and
quality assurance. There's a better solution to restrict values in a column:
create a lookup table with one row for each value you allow.
Then declare a foreign key constraint on the old table referencing
the new table.
Use metadata when validating against a fixed set of values.
Use data when validating against a fluid set of values.

## Files Are Not SQL Data Types

### Resources outside the database are not managed by the database:   
It's common for programmers to be unequivocal that we should always
store files external to the database.
Files don't obey DELETE, transaction isolation, rollback, or work well with
database backup tools. They do not obey SQL access privileges and are not SQL
data types.
Resources outside the database are not managed by the database.
You should consider storing blobs inside the database instead of in
external files. You can save the contents of a BLOB column to a file.

## Too Many Indexes

### Don't create too many indexes:   
You benefit from an index only if you run queries that use that index.
There's no benefit to creating indexes that you don't use.
If you cover a database table with indexes, you incur a lot of overhead
with no assurance of payoff.
Consider dropping unnecessary indexes.
If an index provides all the columns we need, then we don't need to read
rows of data from the table at all. Consider using such covering indexes.
Know your data, know your queries, and maintain the right set of indexes.

## Index Attribute Order

### Align the index attribute order with queries:   
If you create a compound index for the columns, make sure that the query
attributes are in the same order as the index attributes, so that the DBMS
can use the index while processing the query.
If the query and index attribute orders are not aligned, then the DBMS might
be unable to use the index during query processing.

#### Example

```
CREATE INDEX TelephoneBook ON Accounts(last_name, first_name)   
SELECT * FROM Accounts ORDER BY first_name, last_name
```

# Query Anti-Patterns
## SELECT *

### Inefficiency in moving data to the consumer:   
When you SELECT *, you're often retrieving more columns from the database than
your application really needs to function. This causes more data to move from
the database server to the client, slowing access and increasing load on your
machines, as well as taking more time to travel across the network. This is
especially true when someone adds new columns to underlying tables that didn't
exist and weren't needed when the original consumers coded their data access.   

### Indexing issues:   
Consider a scenario where you want to tune a query to a high level of performance.
If you were to use *, and it returned more columns than you actually needed,
the server would often have to perform more expensive methods to retrieve your
data than it otherwise might. For example, you wouldn't be able to create an index
which simply covered the columns in your SELECT list, and even if you did
(including all columns [shudder]), the next guy who came around and added a column
to the underlying table would cause the optimizer to ignore your optimized covering
index, and you'd likely find that the performance of your query would drop
substantially for no readily apparent reason.   

### Binding Problems:   
When you SELECT *, it's possible to retrieve two columns of the same name from two
different tables. This can often crash your data consumer. Imagine a query that joins
two tables, both of which contain a column called \ID\. How would a consumer know
which was which? SELECT * can also confuse views (at least in some versions SQL Server)
when underlying table structures change -- the view is not rebuilt, and the data which
comes back can be nonsense. And the worst part of it is that you can take care to name
your columns whatever you want, but the next guy who comes along might have no way of
knowing that he has to worry about adding a column which will collide with your
already-developed names.   

## NULL Usage


### Use NULL as a Unique Value:   
NULL is not the same as zero. A number ten greater than an unknown is still an unknown.
NULL is not the same as a string of zero length.
Combining any string with NULL in standard SQL returns NULL.
NULL is not the same as false. Boolean expressions with AND, OR, and NOT also produce
results that some people find confusing.
When you declare a column as NOT NULL, it should be because it would make no sense
for the row to exist without a value in that column.
Use null to signify a missing value for any data type.

## NOT NULL Usage

### Use NOT NULL only if the column cannot have a missing value:   
When you declare a column as NOT NULL, it should be because it would make no sense
for the row to exist without a value in that column.
Use null to signify a missing value for any data type.

## String Concatenation

### Use COALESCE for string concatenation of nullable columns:   
You may need to force a column or expression to be non-null for the sake of
simplifying the query logic, but you don't want that value to be stored.
Use COALESCE function to construct the concatenated expression so that a
null-valued column doesn't make the whole expression become null.

#### Example

```
SELECT first_name || COALESCE(' ' || middle_initial || ' ', ' ') || last_name
AS full_name FROM Accounts
```

## GROUP BY Usage

### Do not reference non-grouped columns:   
Every column in the select-list of a query must have a single value row
per row group. This is called the Single-Value Rule.
Columns named in the GROUP BY clause are guaranteed to be exactly one value
per group, no matter how many rows the group matches.
Most DBMSs report an error if you try to run any query that tries to return
a column other than those columns named in the GROUP BY clause or as
arguments to aggregate functions.
Every expression in the select list must be contained in either an
aggregate function or the GROUP BY clause.
Follow the single-value rule to avoid ambiguous query results.

## ORDER BY RAND Usage

### Sorting by a nondeterministic expression (RAND()) means the sorting cannot benefit from an index:   
There is no index containing the values returned by the random function.
That's the point of them being ran- dom: they are different and
unpredictable each time they're selected. This is a problem for the performance
of the query, because using an index is one of the best ways of speeding up
sorting. The consequence of not using an index is that the query result set
has to be sorted by the database using a slow table scan.
One technique that avoids sorting the table is to choose a random value
between 1 and the greatest primary key value.
Still another technique that avoids problems found in the preceding alternatives
is to count the rows in the data set and return a random number between 0 and
the count. Then use this number as an offset when querying the data set.
Some queries just cannot be optimized consider taking a different approach.

## Pattern Matching Usage

### Avoid using vanilla pattern matching:   
The most important disadvantage of pattern-matching operators is that
they have poor performance. A second problem of simple pattern-matching using LIKE
or regular expressions is that it can find unintended matches.
It's best to use a specialized search engine technology like Apache Lucene, instead of SQL.
Another alternative is to reduce the recurring cost of search by saving the result.
Consider using vendor extensions like FULLTEXT INDEX in MySQL.
More broadly, you don't have to use SQL to solve every problem.

## Spaghetti Query Alert

### Split up a complex spaghetti query into several simpler queries:   
SQL is a very expressive language—you can accomplish a lot in a single query or statement.
But that doesn't mean it's mandatory or even a good idea to approach every task with the
assumption it has to be done in one line of code.
One common unintended consequence of producing all your results in one query is
a Cartesian product. This happens when two of the tables in the query have no condition
restricting their relationship. Without such a restriction, the join of two tables pairs
each row in the first table to every row in the other table. Each such pairing becomes a
row of the result set, and you end up with many more rows than you expect.
It's important to consider that these queries are simply hard to write, hard to modify,
and hard to debug. You should expect to get regular requests for incremental enhancements
to your database applications. Managers want more complex reports and more fields in a
user interface. If you design intricate, monolithic SQL queries, it's more costly and
time-consuming to make enhancements to them. Your time is worth something, both to you
and to your project.
Split up a complex spaghetti query into several simpler queries.
When you split up a complex SQL query, the result may be many similar queries,
perhaps varying slightly depending on data values. Writing these queries is a chore,
so it's a good application of SQL code generation.
Although SQL makes it seem possible to solve a complex problem in a single line of code,
don't be tempted to build a house of cards.

## Reduce Number of JOINs

### Reduce Number of JOINs:   
Too many JOINs is a symptom of complex spaghetti queries. Consider splitting
up the complex query into many simpler queries, and reduce the number of JOINs

## Eliminate Unnecessary DISTINCT Conditions

### Eliminate Unnecessary DISTINCT Conditions:   
Too many DISTINCT conditions is a symptom of complex spaghetti queries.
Consider splitting up the complex query into many simpler queries,
and reduce the number of DISTINCT conditions
It is possible that the DISTINCT condition has no effect if a primary key
column is part of the result set of columns

## Implicit Column Usage

### Explicitly name columns:   
Although using wildcards and unnamed columns satisfies the goal
of less typing, this habit creates several hazards.
This can break application refactoring and can harm performance.
Always spell out all the columns you need, instead of relying on
wild-cards or implicit column lists.

## HAVING Clause Usage

### Consider removing the HAVING clause:   
Rewriting the query's HAVING clause into a predicate will enable the
use of indexes during query processing.

#### Example

```
SELECT s.cust_id,count(s.cust_id) FROM SH.sales s GROUP BY s.cust_id
HAVING s.cust_id != '1660' AND s.cust_id != '2'
```
can be rewritten as:   
```
SELECT s.cust_id,count(cust_id) FROM SH.sales s WHERE s.cust_id != '1660'
AND s.cust_id !='2' GROUP BY s.cust_id
```

## Nested sub queries

### Un-nest sub queries:   
 Rewriting nested queries as joins often leads to more efficient
execution and more effective optimization. In general, sub-query unnesting
is always done for correlated sub-queries with, at most, one table in
the FROM clause, which are used in ANY, ALL, and EXISTS predicates.
A uncorrelated sub-query, or a sub-query with more than one table in
the FROM clause, is flattened if it can be decided, based on the query
semantics, that the sub-query returns at most one row.

#### Example

```
SELECT * FROM SH.products p WHERE p.prod_id = (SELECT s.prod_id FROM SH.sales
s WHERE s.cust_id = 100996 AND s.quantity_sold = 1 )
```
can be rewritten as:   
```
SELECT p.* FROM SH.products p, sales s WHERE p.prod_id = s.prod_id AND
s.cust_id = 100996 AND s.quantity_sold = 1
```

## OR Usage

### Consider using an IN predicate when querying an indexed column:   
The IN-list predicate can be exploited for indexed retrieval and also,
the optimizer can sort the IN-list to match the sort sequence of the index,
leading to more efficient retrieval. Note that the IN-list must contain only
constants, or values that are constant during one execution of the query block,
such as outer references.

#### Example

```
SELECT s.* FROM SH.sales s WHERE s.prod_id = 14 OR s.prod_id = 17
```  
can be rewritten as:   
```
SELECT s.* FROM SH.sales s WHERE s.prod_id IN (14, 17)
```

## UNION Usage

### Consider using UNION ALL if you do not care about duplicates:   
Unlike UNION which removes duplicates, UNION ALL allows duplicate tuples.
If you do not care about duplicate tuples, then using UNION ALL would be
a faster option.

## DISTINCT & JOIN Usage

### Consider using a sub-query with EXISTS instead of DISTINCT:   
The DISTINCT keyword removes duplicates after sorting the tuples.
Instead, consider using a sub query with the EXISTS keyword, you can avoid
having to return an entire table.

#### Example

```
SELECT DISTINCT c.country_id, c.country_name FROM SH.countries c,
SH.customers e WHERE e.country_id = c.country_id
```
can be rewritten to:   
```
SELECT c.country_id, c.country_name FROM SH.countries c WHERE  EXISTS
(SELECT 'X' FROM  SH.customers e WHERE e.country_id = c.country_id)
```

# Application Development Anti-Patterns  

## Readable Passwords

### Do not store readable passwords:   
It's not secure to store a password in clear text or even to pass it over the
network in the clear. If an attacker can read the SQL statement you use to
insert a password, they can see the password plainly.
Additionally, interpolating the user's input string into the SQL query in plain text
exposes it to discovery by an attacker.
If you can read passwords, so can a hacker.
The solution is to encode the password using a one-way cryptographic hash 
function. This function transforms its input string into a new string,
called the hash, that is unrecognizable.
Use a salt to thwart dictionary attacks. Don't put the plain-text password
into the SQL query. Instead, compute the hash in your application code,
and use only the hash in the SQL query.


Source: https://github.com/jarulraj/sqlcheck
