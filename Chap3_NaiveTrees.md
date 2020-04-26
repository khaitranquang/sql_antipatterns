## Chap 3: Naive Trees

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

### 3.1 Object: Store and Query Hierarchies
It is common for data to have recursive relationships. Data may be organized in a treelike or hierarchical way.

### 3.2 Antipattern: Always depend on one's parent
The naive solution commonly is to add a column *parent_id*. This column references another comment in the same table. This design is called *Adjacency List*. It's probably the most common design software developers use to store hierarchical data.

Example data:

| comment_id | parent_id | author | comment |
|------------|-----------|--------|---------|
| 1 | NULL | Fran | What is the cause of this bug? |
| 2 | 1 | Ollie | I think it is a null pointer |
| 3 | 2 | Fran | No. I checked for that |
| 4 | 1 | Kukla | We need to check for valid input |
| 5 | 4 | Ollie | Yes. That is a bug. |
| 6 | 4 | Fran | yes, please add a check |
| 7 | 6 | Kukla | That fixed it |


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
=> The number of joins in an SQL query must be fixed:
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

The strength of Adjacency List design is retrieving the direct parent or child of a given node. It is also easy to insert rows. If those operators are all we need to do with our hierarchical data, then Adjacency List can work well for us.

Some brands of RDBMS support extensions to SQL to support hierarchical stored in Adjacency List format. Microsoft SQL Server 2005, Oracle 11g, IBM DB2, and PostgreSQL 8.4 support **recursive queries** using common table expressions, as shown earlier.
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

There are several alternatives to the Adjacency List model of storing hierarchical data, including *Path Enumeration, Nested Sets, Closure Table*

#### Path Enumeration
One weakness of Adjacency List is that it is expensive to retrieve ancestors of a given node in the tree. In Path Enumeration, this is solved by storing the string of ancestors as an attribute of each node.

We can see a form of Path Enumeration in directory hierarchies.

In *Comments* table, instead of the parent_id column, define a column called *path* as a long *VARCHAR*. The string stored in this column is sequence of ancestors of the current row in order from the top of the tree down, just like UNIX path. We can choose / as a separator character.

| comment_id | path | author | comment|
|------------|------|--------|--------|
| 1 | 1/ | Fran | What’s the cause of this bug?|
| 2 | 1/2/ | Ollie | I think it’s a null pointer. |
| 3 | 1/2/3/ | Fran | No, I checked for that. |
| 4 | 1/4/ | Kukla | We need to check for invalid input. |
| 5 | 1/4/5/ | Ollie | Yes, that’s a bug.|
| 6 | 1/4/6/ | Fran | Yes, please add a check.|
| 7 | 1/4/6/7/ | Kukla | That fixed it|

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

An easy way to assign these values is by following a depth-first traversal of the tree, assigning *nsleft* numbers incrementally as you descend a branch of the tree and assigning *nsright* numbers as you ascend back up the branch.

| comment_id | nsleft | nsright | author | comment|
|------------|--------|---------|--------|--------|
| 1 | 1 | 14 | Fran | What’s the cause of this bug?|
| 2 | 2 | 5 | Ollie | I think it’s a null pointer. |
| 3 | 3 | 4 | Fran | No, I checked for that. |
| 4 | 6 | 13 | Kukla | We need to check for invalid input. |
| 5 | 7 | 8 | Ollie | Yes, that’s a bug.|
| 6 | 9 | 12 | Fran | Yes, please add a check.|
| 7 | 10 | 11 | Kukla | That fixed it|

Once we have assigned each node with these numbers, we can use them to find ancestors and descendants of any given node. For example, we can retrieve comment #14 and its descendants by searching for node whose numbers are between the current node's nsleft and nsright

```
SELECT c2.*
FROM Comments AS c1
    JOIN Comments as c2
        ON c2.nsleft BETWEEN c1.nsleft AND c1.nsright
WHERE c1.comment_id = 4;
```

One chief strength of the Nested Sets design is that when we delete a nonleaf node, its descendants are automatically considered direct children of the deleted node's parents.

However, some queries that are simple in the Adjacency List design, such as retrieving the immediate child or immediate parent, are more complex in the Nested Sets design.

For example, to find the immediate parent of comment #6, do this:
```
SELECT parent.*
FROM Comment AS c
    JOIN Comment AS parent
        ON c.nsleft BETWEEN parent.nsleft AND parent.nsright
    LEFT OUTER JOIN Comment AS in_between
        ON c.nsleft BETWEEN in_between.nsleft AND in_between.nsright
        AND in_between.nsleft BETWEEN parent.nsleft AND parent.nsright
WHERE c.comment_id = 6
    AND in_between.comment_id IS NULL;
```

Manipulations of the tree, inserting and moving nodes, are generally more complex in Nested Sets design than they are in other models. When you insert a new node, you need to recalculate all the left and right values greater than the left value of the new node.

The Nested Sets model is best when it’s more important to perform queries for subtrees quickly and easily, rather than operations on individual nodes. Inserting and moving nodes is complex, because of the requirement to renumber the left and right values. If your usage of the tree involves frequent insertions, Nested Sets isn’t the best choice.

#### Closure Table

The Closure Table solution is a simple and elegant way of storing hierarchies. It involves storing all paths through the tree, not just those with a direct parent-child relationship.

In addition to a plain Comments table, create another table TreePaths, with two columns, each of which is a foreign key to the Comments table.

```
CREATE TABLE Comments (
    comment_id  SERIAL PRIMARY KEY,
    bug_id BIGINT UNSIGNED NOT NULL,
    author BIGINT UNSIGNED NOT NULL,
    comment_date DATETIME NOT NULL,
    comment TEXT NOT NULL,
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id),
    FOREIGN KEY (author) REFERENCES Accounts(account_id)
);

CREATE TABLE TreePaths (
    ancestor BIGINT UNSIGNED NOT NULL,
    descendant BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY(ancestor, descendant),
    FOREIGN KEY (ancestor) REFERENCES Comments(comment_id),
    FOREIGN KEY (descendant) REFERENCES Comments(comment_id)
);
```

Instead of using the Comments table to store information about the tree structure, use the TreePaths table. Store one row in this table for each pair of nodes in the tree that shares an ancestor/descendant relationship, even if they are separated by multiple levels in the tree. Also add a row for each node to reference itself.

| ancestor | descendant | ancestor | descendant | ancestor| descendant|
|------------|--------|---------|--------|--------|-------------|
| 1 | 1 | 1 | 7 | 4 | 6 |
| 1 | 2 | 2 | 2 | 4 | 7|
| 1 | 3 | 2 | 3 | 5 | 5 |
| 1 | 4 | 3 | 3 | 6 | 6 |
| 1 | 5 | 4 | 4 | 6 | 7 |
| 1 | 6 | 4 | 5 | 7 | 7 |

The queries to retrieve ancestors and descendants from this table are even more straightforward than those in the Nested Sets solution. To retrieve descendants of comment #4, match rows in TreePaths where the ancestor is 4:
```
SELECT c.*
FROM Comments AS c
JOIN TreePaths AS t ON c.comment_id = t.descendant
WHERE t.ancestor = 4;
```

To insert a new leaf node, for instance a new child of comment #5, first insert the self-referencing row. Then add a copy of the set of rows in TreePaths that reference comment #5 as a descendant (including the row in which comment #5 references itself), replacing the descendant with the number of the new comment:

```
INSERT INTO TreePaths (ancestor, descendant)
    SELECT t.ancestor, 8
    FROM TreePaths AS t
    WHERE t.descendant = 5
UNION ALL
    SELECT 8, 8;
```
To delete a leaf node, for instance comment #7, delete all rows in TreePaths that reference comment #7 as a descendant :
```
DELETE FROM TreePaths WHERE descendant = 7;
```

Notice that if you delete rows in TreePaths , this doesn’t delete the comments themselves.


#### Which Design Should You Use?

| Design | Tables | Query Child | Query Tree | Insert| Delete | Ref. Integ.|
|------------|--------|---------|--------|--------|-------------|-----------|
| Adjacency List   | 1 | Easy | Hard | Easy | Easy | Yes |
| Recursive Query  | 1 | Easy | Easy | Easy | Easy | Yes |
| Path Enumeration | 1 | Easy | Easy | Easy | Easy | No |
| Nested Sets      | 1 | Hard | Easy | Hard | Hard | No |
| Closure Table    | 2 | Easy | Easy | Easy | Easy | Yes |

Each of the designs has its own strengths and weaknesses. Choose the design depending on which operations we need to be most efficient.

* *Adjacency List* is the most conventional design, and many software developers recognize it.
* *Recursive Queries* using WITH or CONNECT BY PRIOR make it more efficient to use the Adjacency List design, provided you use one of the database brands that supports the syntax.
* *Path Enumeration* is good for breadcrumbs in user interfaces, but it’s fragile because it fails to enforce referential integrity and stores information redundantly.
* *Nested Sets* is a clever solution—maybe too clever. It also fails to support referential integrity. It’s best used when we need to query a tree more frequently than we need to modify the tree.
* *Closure Table* is the most versatile design and the only design in this chapter that could allow a node to belong to multiple trees. It requires an additional table to store the relationships. This design also uses a lot of rows when encoding deep hierarchies, increasing space consumption as a trade-off for reducing computing.