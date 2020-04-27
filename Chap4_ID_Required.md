## Chap 4: ID Required

In the content management database, he stored articles for publishing on a website. He used an intersection table for a many-to-many association between a table of articles and a table of tags.

```
CREATE TABLE ArticleTags (
	id SERIAL PRIMARY KEY,
	article_id BIGINT UNSIGNED NOT NULL,
	tag_id BIGINT UNSIGNED NOT NULL,
	FOREIGN KEY (article_id) REFERENCES Articles (id),
	FOREIGN KEY (tag_id) REFERENCES Tags (id)
);
```

Now, we knew that there were only five articles with the "economy" tag, but the query was telling us there were seven.
```
SELECT tag_id, COUNT(*) AS articles_per_tag FROM ArticleTags WHERE tag_id = 327;
```
When we queired all the rows: three rows showed the same association, although we had different values for *id*

This table had a primary key, but that primary key wasn’t preventing duplicates in the columns that mattered. One remedy might be to create a UNIQUE constraint over the other two columns, but given that, **why is the id column needed at all?**

### 4.1 Object: Establish Primary Key Conventions

The objetive is to make sure every table has a primary key, but confusion about the nature of a primary key has resulted in an antipattern.

A primary key is guaranteed to be unique over all rows in the table, so this is the logical mechanism to address individual rows and to prevent duplicate rows from being stored. A primary key is also referenced by foreign keys to create table associations.

A primary key must be unique. A new column is needed in such tables to sotre an artificial value that has no meaning in the domain modeled by the table. This column is used as the primary key, so we can address rows uniquely white allowing any other attribute column to contain duplicates. This type of primary key column is sometimes called a *pseudokey* or a * surrogate key*. Pseudokeys are a useful feature, by they are not only solution for declaring a primary key.

### 4.2 Antipattern: One Size Fits All

Books, articles and programming frameworks have established a cultural convention that every database table must have a primary key column with the following characteristics:

* The primary key's column name is id
* Its data type is a 32-bit or 64-bit integer
* Unique values are generated automaticlly.

Adding an *id* column to every table cuses several effects that make its use seem arbitrary.

#### Making a Redundant key
We might see an id column defined as the primary key simply for the sake of tradition, even when another cilumn in the same table could be used as the natural primary key. The other column may even be defined with a UNIQUE constraint.

```
CREATE TABLE Bugs (
	id SERIAL PRIMARY KEY,
	bug_id VARCHAR(10) UNIQUE,
	description VARCHAR(1000),
	-- . . .
);
INSERT INTO Bugs (bug_id, description, ...)
	VALUES ('VIS-078', 'crashes on save', ...);
```

#### Allowing duplicate rows

A compound key consists of multiple columns. One typical use for a compound key is in an intersection table like BugsProducts. The primary key should ensure that a given combination of values for bug_id and product_id appears only once in the table, even though each value may appear many times in different pairings.

However, when you use the mandatory id column as the primary key, the constraint no longer applies to two columns that should be unique.

```
CREATE TABLE BugsProducts (
	id SERIAL PRIMARY KEY,
	bug_id BIGINT UNSIGNED NOT NULL,
	product_id BIGINT UNSIGNED NOT NULL,
	FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id),
	FOREIGN KEY (product_id) REFERENCES Products(product_id)
);
INSERT INTO BugsProducts (bug_id, product_id)
	VALUES (1234, 1), (1234, 1), (1234, 1); -- duplicates are permitted
```

Duplicates in this intersection table cause unintended results when you use the table to match Bugs to Products. To prevent duplicates, you could declare a UNIQUE constraint over the two columns besides id:

```
CREATE TABLE BugsProducts (
	id SERIAL PRIMARY KEY,
	bug_id BIGINT UNSIGNED NOT NULL,
	product_id BIGINT UNSIGNED NOT NULL,
	UNIQUE KEY (bug_id, product_id),
	FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id),
	FOREIGN KEY (product_id) REFERENCES Products(product_id)
);
```

But if you need a unique constraint over those two columns anyway, the id column is superfluous.

#### Obscuring the Meaning of the Key

The name *id* is so generic that it holds no meaning.

#### Using USING

We usually use SQL syntax *JOIN ... ON*:

```
SELECT * FROM Bugs AS b JOIN BugsProducts AS bp ON (b.bug_id = bp.bug_id);
```

SQL also supports a more concise suntax for expressing a join between two tables. We can rewrite:
```
SELECT * FROM Bugs JOIN BugsProducts USING (bug_id);
```

However, if all tables are required to define a pseudokey primary key name *id*, then a foreign key column in a dependent table can never using the same name as the primary key it references. Instaed, 

