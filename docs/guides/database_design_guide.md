## Table Naming
-	Table names should be concise and descriptive. Two to three words is the sweet spot.
-	Do not prefix ‘Bunzl’ to the table name by default. If the table exists in the same database with a duplicate name, odds are the table name can be more specific or accurately descriptive. It is okay to have a table with the same name in a different database, even if there is a single ADF job interacting with both tables.
-	Choose the casing that is consistent with all other tables in the database. Otherwise, use the convention recommended by the database provider (i.e. Postgres, MySQL, etc.).
Columns
-	Every table must have audit columns for storing who created the record and when. These include ‘CreatedOn’, ‘UpdatedOn’, ‘CreatedBy’, UpdatedBy’ and must be named accordingly.
-	‘CreatedOn’ and’ ModifiedOn’ column type must be ‘datetime’ and populated with the current UTC time.
-	‘CreatedOn’ and ‘CreatedBy’ columns must have a not-null constraint
-	‘UpdatedOn’ and ‘UpdatedBy’ columns must be nullable (NOT have a not-null constraint). These columns should only be populated when a row is updated and left as null when a row is created.
-	Include an ‘IsActive’ Boolean type (not-null) column for tables that are exposed by APIs that may have DELETE operation that acts on the table.
-	Avoid using DateOnly types for columns with dates. Instead use DateTime type.

## Primary Keys
When choosing what to use for a table’s primary key, security, performance, scalability and readability must all be considered. See this document for more details on using UUIDs and incremental integers.
In summary, use incremental/serial integers as the default primary key type for most internal and performance-sensitive systems but be mindful of the data that may be exposed by APIs. Only switch to UUIDs in the following scenarios: 
    - The primary key will be exposed externally via APIs that expose private or sensitive data, anything that should be hidden from other users. An example of this would be a User table that exposes PII data like email and address information or an Order table that exposes order numbers. 
        o	NOTE: this scenario does NOT include tables that store generic data like type or status values. In those cases, incremental integers may still be used. An example of this would be an OrderStatus table that exposes the potential statuses an order can have. Even if this is exposed by an API, integer is still the preferred type for that table’s primary key
    - The system is distributed, requiring independent ID generation.
    - The data architecture requires merging or synchronization across multiple databases.

## Other Best Practices
-	Use join table where necessary to normalize the database as much as possible. A common example of this is when there is a many-to-many relationship between two tables. A third table can be used to join the data between those two tables. Columns being populated with a string of comma-separated list of values is usually a sign that the database can be normalized. (see here for resource).
-	Consider scale when designing database schemas. Have a good idea of how many records will be created for a given operation and how this may impact all related tables. Consider the business domain use-cases. Have a good idea of how many records in total will eventually be created and queried. 
-	Avoid having too many ‘value’ columns in a single column. Having more than 10 columns in a table may be a sign that the design can be further optimized.
-	Consider maintainability: Create a separate table for storing a short list of static values for things like resource types or statuses. For example, consider an Order table that has a status of the order. Instead of populating with limited potential values ‘pending’, ‘paid’, ‘shipped’, etc., create a separate table for OrderStatus that contains a row for each value with a value and id column. The Order table can then reference the status value by including an id to the OrderStatus table. 
