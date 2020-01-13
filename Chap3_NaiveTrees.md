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