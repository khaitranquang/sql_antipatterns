## Chap 6: Entity - Attribute - Value

### 6.1 Objective: Support Variable Attributes

Extensibility is frequently a goal of software projects. 

A conventional table consists of attribute columns that are relevant for every row in the table, since every rtow represents an instance of a similar object. A different set of attributes represents a different type of object, so it belongs in a different table.

In modern object-oriented programming modes, howerver, dofferent object types can be related, for instance, by extending the same base type. In object-object-oriented design, these object are considered instances of the same base type, as well as instances of their respective subtypes. We would like to store objects as rows in a single database table to simplify. comparations and calculations over multiple objects. But we also need to allow objects of each subtype to sotre their respective attribute columns, which may not apply to the base type or to other subtypes.

### 6.2 Antipattern: Use a Generic Attribute Table

The solution that appeals to some programmer when they need to support variable attributes is to create a second table, storing attributes as rows. Each row in this attribute table has three columns:

* The **Entity**: typically this is a foreign key to a parent table that has one row per entity.
* The *Attribute*: this is simply the name of a column in a conventional table, but in this new design, we have to identify the attribute on each given row.
* The *Value*: each entity has a value for each of its attributes.

The desing is called Entity-Attribute-Value, or EAV for short. It's also sometimes called *open schema*, *schemaless* or *name-value pairs*.

For example, a given bug is an entity we identify by its primary key value 1234. It has an attribute called *status*. The value of that attribute for bug 1234 is *NEW*

```
CREATE TABLE Issues (
	issue_id SERIAL PRIMARY KEY
);
INSERT INTO Issues (issue_id) VALUES (1234);

CREATE TABLE IssueAttributes (
	issue_id BIGINT UNSIGNED NOT NULL,
	attr_name VARCHAR(100) NOT NULL,
	attr_value VARCHAR(100),
	PRIMARY KEY (issue_id, attr_name),
	FOREIGN KEY (issue_id) REFERENCES Issues(issue_id)
);

INSERT INTO IssueAttributes (issue_id, attr_name, attr_value)
VALUES
	(1234, 'product', '1'),
	(1234, 'date_reported', '2009-06-01'),
	(1234, 'status', 'NEW'),
	(1234, 'description', 'Saving does not work'),
	(1234, 'reported_by', 'Bill'),
	(1234, 'version_affected', '1.0'),
	(1234, 'severity', 'loss of functionality'),
	(1234, 'priority', 'high');
```

By adding one additional table, yoi seem to gain the following benefits:

* Both tables have few columns
* The number of columns does not need to grow to suppoert new attributes,
* We avoid a clutter of columns that contain null in rows where the attribute is inapplicable.

This appears to be an improved design. However, the simple database structure doesn't make up for the difficulty of using it.

#### Querying an Attribute

We need to run a report of the bugs reported per day. In a conventional table design, the *Issues* table would have a simple attribute column such as *date_reported*. To query all bugs with their report dates, we could use a simple query like this:

```
SELECT issue_id, date_reported FROM Issues;
```

To get the same information as the previous query using the EAV desing, we need to fetch rows from the *IssueAttributes* table that sores an attribute named by the string *date_Reported*. This query is more verbose but less clear:

```
SELECT issue_id, attr_value AS "date_reported"
FROM IssueAttributes
WHERE attr_name = 'date_reported';
```

#### Supporting Data Integrity

When we use EAV, we sacrifice many advantages that a conventional database design would have given us

##### We Can't Make Mandatory Attributes

To help your boss generate accurate project reports, you should also require that the date_reported attribute has a value. In a conventional database design, it would be simple to enforce a mandatory column by declaring the column NOT NULL.

In the EAV design, each attribute corresponds to a row in the IssueAttributes table, not a column. You would need a constraint that checks that a row exists for each issue_id value, and the row must have the string date_reported in its attr_name column. 

However, SQL doesn’t support a constraint that can do this. So, you must write application code to enforce it. If you do find a bug with no reported date, should you add a value for this attribute? What value should you give it? If you make a guess or use some default value for a missing attribute, how does that affect the accuracy of your boss’s reports?

##### We Can't use SQL Data Types


##### We Can't Enforce Referential Integrity

##### We Can't Make Up Attribute Names

##### Reconstructing a Row


