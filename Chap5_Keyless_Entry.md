## Chap 5: Keyless Entry

### 5.1 Objective: Simplify Database Architecture

Relational database design is almost as much about relationships between tables as it is aboout the individual tables themselves. *REferential integrity* is an important part of proper database design and operation.

However, some software developers recommend avoiding referantial integrity constraints. Some reasons:

* Your data updates can conflict with the constraints.
* You are using a database design that's so flexible it can't suppport referential integrity constraints.
* You believe that the index the database creates for the foreign key will impact performance
* You use a database brand that doesn't support foreign keys.
* You have to look up the syntax for declaring foreign keys.

### 5.2 Antipattern: Leave Out the Constraints

Even though it seems at first that skipping foreign key constraints makes your database design simpler, more flexible, or speedier, yoi pay for this in other ways. It becomes your responsibility to write code to ensure referential integrity manually.

#### Assuming Flawless Code

Many people's solution for referential integrity is to write application code so that data relationships are always satisfied. Every time we insert a row, make sure that values in foreign key columns reference existing values in the referenced table. Every time we delete a row, make sure that any child tables are also updated appropriately. 

To delete a row, we'd have to make sure no child rows exist:
```
SELECT bug_id FROM Bugs WHERE reported_by = 1;
```

Then you could delete the account:
```
DELETE FROM Accounts WHERE account_id = 1;
```

What if the user with *account_id 1* sneaks in and enters a new bug in the moment ager our query and before we delete that account??? That still leaves a broken reference - a bug reported by an account that no longer exists.

The only remedy is for you to explicitly lock the *Bugs* table while we are checking it and unlock it afterwe have finished deleting the account.


#### Checking for Mistakes

For example, the *Bugs.status* column references the lookup table *BugStatus*. To find bugs with invalid status value, we could use a query like:

```
SELECT b.bug_id, b.status
FROM Bugs b LEFT OUTER JOIN BugStatus s
	ON (b.status = s.status)
WHERE s.status IS NULL;
```

If you find yourself in the habit of checking for broken references like this, your next question is, how often do you need to run these checks? Running hundereds of checks every day, or even more frequntly, becomes quite a chore.

What happens when we do find a broken reference? Can we correct it? We can - sometimes. For instance, we might change an invalid bug status value to a sensible default.

```
UPDATE Bugs SET status = DEFAULT WHERE status = 'BANANA';
```

Inevitably, there are other cases where we can;t synthesize data to correct these kinds of mistakes. For example, the *Bugs.reported_by* column should reference the account of the user who reported the given bug, but if this value is invalid, which user's account should we use as a replacement???


#### "It's not My Fault!"

It’s pretty unlikely that all your code touching the database is perfect. You could easily perform similar database updates in several functions in your application. When you have to change the code, how can you be sure you’ve applied compatible changes to every case in your application?

#### Catch-22 Updates

Many developer avoid foreign key constraints because the constraints make it inconvenient to update related columns in multiple tables. For instance, if we need to delete a row that other rows depend on, we have to delete the child rows first to avoid violating foreign key constraints.

We have to execute multiple statements manually, one for each child table. If we add another child table in a feature enhancement to our database, we have to fix our code to delete from new the new table too. But this problem is solvable.

The unsolvable problem is when we UPDATE a column that child rows depend on. We can't update the child rows before we update the parent, an we can't update the parent before we update the child values that reference it. We need to make both changes simultaneously, but that's impossible using two separate updates. It is a catch-22 scenario.

Some developers find these scenario difficult to manage, so they decide not to use foreign keys at all.


### 5.3 How to recognize the Antipattern

* “How do I query to check for a value that exists in one table and not the other table?”
* “Is there a quick way to check that a value exists in one table as part of my insert to a second table?”
* “Foreign keys? I was told not to use them because they slow down the database.”

### 5.4 Legitimate Uses of the Antipattern

Sometimes we are forced to use a database brand that doesn't support foreign key constraints (for example MySQL's MyISAM storage engine or SQLite prior to version 3.6.19). If that's the case, then we have to find a way to compensate.

There are also some ultra-flexible database designs where foreign keys can’t model the relationships. It should be a strong clue that you’re using another SQL antipattern if we can’t use traditional referential integrity constraints.

### 5.5 Solution: Declare Constraints

Instead of searching for and correcting data integrity mistakes, we can prevent these mistake from entering our database in the first time by using foreign key constraints.

Using foreign keys saves us from writing unnecessary code and ensures that all our code works the same way if we change the database. 


#### Support Multitable changes

Foreign keys have another feature you can’t mimic using application code: **cascading updates**.

```
CREATE TABLE Bugs (
-- . . .
    reported_by BIGINT UNSIGNED NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'NEW' ,
    FOREIGN KEY (reported_by) REFERENCES Accounts(account_id)
        ON UPDATE CASCADE
        ON DELETE RESTRICT,
    FOREIGN KEY (status) REFERENCES BugStatus(status)
        ON UPDATE CASCADE
        ON DELETE SET DEFAULT
);
```






