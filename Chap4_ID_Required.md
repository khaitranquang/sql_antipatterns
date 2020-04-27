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

