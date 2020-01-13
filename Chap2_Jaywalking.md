## Chap 2: Jaywalking

Programmer commonly use command-separated lists to avoid creating and intersection table for many-to-many relationship. This anti-pattern is called Jaywalking, because jaywalking is also an act of avoiding an intersection.

### 2.1 Object: Store Multivalue Attributes

When a column in a table has single value, the design is straightforward: choose an SQL type to represent a single instance of the value.
But, how do we can store a collection of related values in a column?

=> We create a many-to-one (or one-to-many) relationship
=> But...
=> As our project matures, we realize that we need to support a many-to-may relationship instead of many-to-one relationship...
=> Holly sh**

Example: 
We might associate a product with a contact using an integer column. Each contact may have many products, so we create a many-to-one relationship between products and accounts. Then, we realize that we also need support a one-to-many relationship from products to accounts

### 2.2 Antipattern: Format Comma-Separated Lists

To minimize changes to the database structure, we decide to redefine the account_id as a VARCHAR so you can list multiple account IDs in that column, separated by commas.
```
CREATE TABLE Products (
	product_id SERIAL PRIMARY KEY,
	product_name VARCHAR(1000),
	account_id VARCHAR(100), -- comma-separated list
	-- . . .
);

INSERT INTO Products (product_id, product_name, account_id)
VALUES (DEFAULT, 'Visual TurboBuilder', '12,34');		-- comma-separated list
```

This seems like the problem is resolved. But...

#### Querying Products for a Special Account

Queries are difficult:
```SELECT * FROM Products WHERE account_id REGEXP '[[:<:]]12[[:>:]]';```

=> SQL code is not vender-neutural

#### Querying Accounts for a Given Products
Likewise:
```
SELECT * FROM Products AS p JOIN Accounts AS a
	ON p.account_id REGEXP '[[:<:]]' || a.account_id || '[[:>:]]'
WHERE p.product_id = 123;
```

#### Making Aggregate Queries

#### Updating Accounts for a Specific Product

#### Validating Product IDs

#### Choosing a Separator Character
What happen if account id in separated lists contains comma???

#### List length limitions ???
How do we do if we want to limit number of accounts for each product???

=>>>>  This anti-pattern seems many issues...

### 2.3 How to recognize the Antipattern
* “What is the greatest number of entries this list must support?
* “Do you know how to match a word boundary in SQL?”
* “What character will never appear in any list entry?”

### 2.4 Legitimate uses of the Antipattern

Some cases, **if the database was normalized**, this antipattern permits your application code to be more flexible, and it allows your databse to help preserve data 

### 2.5 Solution: 
Haha, we need **create an intersection table**
That mean we implements a many-to-many relationship










