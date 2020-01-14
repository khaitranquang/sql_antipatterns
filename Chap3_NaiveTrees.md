## Chap 3: Navie Trees

Suppose we need develop a comment feature. Readers can contribute comments and even reply to each other (like Reddit). We choose a simple solution to track these reply chains: Each comment references the comment to which it replies:
```
CREATE TABLE Comments (
comment_id SERIAL PRIMARY KEY,
parent_id BIGINT UNSIGNED,
comment TEXT NOT NULL,
FOREIGN KEY (parent_id) REFERENCES Comments(comment_id)
);
```

It soon becomes clear. However, that it's hard to retrieve a long chain of replies in a single query. If the thread can have an *unlimited* depth. Hmm...

The other idea we have is to retrieve *all* comments and assemble them in to tree data structures in application memory. But the website publish very very many articles every day and each article can have hundreds comments. Sorting through millions of comments every time someone views the website is impractical.

=> We need find the better way to store the threads of comments simply and efficiently.

### 3.1 Object: Store and Query Hierachies
It is common for data to have recursive relationships. Data may be organized in a treelike or hierachical way.

### 3.2 Antipattern: Always depend on one's parrent
The naive solution commonly is to add a colummn *parent_id*. This colummn references another comment in the same table. This design is called *Adjacency List*.

There are some issues:
#### Query a Tree with Adjacency List
Commonly, we query:
```
SELECT c1.*, c2.*
FROM Comments c1 LEFT OUTER JOIN Comments c2
ON c2.parent_id = c1.comment_id;
```
=> Only two levels of the tree

To extend to any depth, we need compute the COUNT() of comments in the thread or the SUM() of the cost of parts in mechanical assembly.
=> The number of joins in an SQL query musst be fixed:
```
SELECT c1.*, c2.*, c3.*, c4.*
	FROM Comments c1 					-- 1st level
LEFT OUTER JOIN Comments c2
	ON c2.parent_id = c1.comment_id 	-- 2nd level
LEFT OUTER JOIN Comments c3
	ON c3.parent_id = c2.comment_id 	-- 3rd level
LEFT OUTER JOIN Comments c4
	ON c4.parent_id = c3.comment_id; 	-- 4th level'
```

#### Maintaining a Tree with Adjacency List

Deleting a node from a tree is complex. If we want to delete an entire subtree, we have to issue multiple queries to find all descendants. Then move the descendants from the lowest level up to satisfy the foreign key integrity.


### 3.3 How to recognize the Antipattern

* “How many levels do we need to support in trees?”

* “I dread ever having to touch the code that manages the tree data
structures.”

* “I need to run a script periodically to clean up the orphaned rows
in the trees.”

### 3.4 Legitimate Uses of the Antipattern

The strength of Adjacency List design is retrieving the direct parent or child of a given node. It is also easy to insert rows. If those operators are all we need to do with our hierachical data, then Adjacency List can work well for us.

Some brands of RDBMS support extensions to SQL to support hierachical stored in Adjacency List format: Microsoft SQL Server 2005, Oracle 11g, IBM DB2, and PostgreSQL 8.4.
```
WITH CommentTree
	(comment_id, bug_id, parent_id, author, comment, depth)
AS (
	SELECT *, 0 AS depth FROM Comments WHERE parent_id IS NULL
	UNION ALL
		SELECT c.*, ct.depth+1 AS depth FROM CommentTree ct
		JOIN Comments c ON (ct.comment_id = c.parent_id)
)
SELECT * FROM CommentTree WHERE bug_id = 1234;
```

MySQL, SQLite, and Informix are examples of database brands that don't support this syntax yet.

### 3.5 Solution: Use Alternative Tree Models

There are several alternatives to the Adjacency List model of storing hierachical data, including *Path Enumeration, Nested Sets, Closure Table*

#### Path Enumeration
One weakness of Adjacency List is that it is expensive to retrieve ancestors of a given node in the tree. In Path Enumeration, this is solved by storing the string of ancestors as an atrribute of each node.

We can see a form of Path Enumeration in directory hierachies.

In *Comments* table, instead of the parent_id colummn, define a colummn called *path* as a long *VARCHAR*. The string stored in this colummn is sequence of ancestors of the current row in order from the top of the tree down, just like UNIX path. We can choose / as a separator character.
```
comment_id 	path 		author 		comment
------------------------------------------------------------------------
1 			1/ 			Fran 		What’s the cause of this bug?
2 			1/2/ 		Ollie 		I think it’s a null pointer.
3 			1/2/3/ 		Fran 		No, I checked for that.
4 			1/4/ 		Kukla 		We need to check for invalid input.
5 			1/4/5/ 		Ollie 		Yes, that’s a bug.
6 			1/4/6/ 		Fran 		Yes, please add a check.
7 			1/4/6/7/ 	Kukla 		That fixed it
```

We can query ancestors by comparing the current row's path to a pattern formed from the path of another row. For example, to find ancestors of comment #7, whose path is *1/4/6/7*, do this:
```
SELECT *
FROM Comments AS c
WHERE '1/4/6/7/' LIKE c.path || '%';
```
This matches the patterns formed from paths of ancestors *1/4/6/%, 1/4/%, 1/%*

=> Drawbacks: **Jaywalking Antipattern**

#### Nested Sets

Each node is given *nsleft* and *nsright* numbers in the following way: the *nsleft* number is less than the numbers of all node's children, whereas the *nsright* is greater than the numbers of all the node's children. These numbers have no relation to comment_id value.



















