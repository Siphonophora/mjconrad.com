---
layout: single
classes: wide
title:  "Easily Compare any SQL Server Queries or Tables"
date:   2020-09-21 09:00:00 -0400
permalink: blog/easily-compare-any-sql-server-queries-or-tables
---

I'm excited to announce that this last week I have released a new project ([source code][project]), which is [hosted here][appURL]. Its a free utility that generates SQL Server scripts to compare the results of any two queries. You then run that comparison script from your client (SQL Server Management Studio, Azure Data Studio...) and are presented with a detailed report of all identical or discrepant records. Notice, that's **queries** we are comparing and not just tables or views. This allows for some interesting capabilities, which differ from how the major commercial data comparison programs work. A few use cases for this tool are outlined below, along with a basic usage example.

* **Table or View Comparison** - This is the bread and butter for any data comparison tool. Just plug in a select statement for any two tables or views, on the same or different databases or servers, and you can see matched, extra, missing and discrepant rows. You have probably made comparisons scripts like this by hand many times, and don't have to anymore. An example of this type of comparison is shown below. 
* **Validating a Refactored Query** - Safely refactoring complex queries for performance or clarity can be daunting. Once you introduce temp tables, CTEs and nested subqueries the code can become difficult to reason about. *In fact, working on a complex refactoring was what originally motivated this project.* So long as the output columns are the same after refactoring, then the queries will be comparable, regardless of the complexity of the syntax. The queries being compared can even be parameterized. You may find that doing this kind of refactoring aided by this tool, can save quite a bit of time, as you can iterate over changes to the test query and more easily find and resolve errors than by doing any visual comparison of results.
* **Validating Schema Changes** - I recently working on a change to a set of tables which were not fully normalized. As part of the change they needed to become normalized. As you may know de-normalized data can have discrepancies, which is why they are often avoided. For example, if a table of books stores author names instead of a foreign key to an authors table, then you can easily end up with inconsistencies if the author's name was updated on one but not all of the book rows. So, when I was working on normalizing those tables I needed to confirm if the script to normalize the tables was not making unexpected changes. This turns out to be simple, by comparing a query that selected all the columns from all the affected tables in a production database, and another query selecting the now rearranged columns in a dev database.

## A Simple Data Comparison

As a simple example, lets create two tables to compare. As you can see these have the same structure, but have some deliberately discrepant data.

To generate a comparison script:
1. Create these tables and data in a test database.

``` sql
-- Sample Data Source
Create Table Books
( Id int not null
 ,Author varchar(50) not null
 ,Title varchar(50) not null)

Create Table SciFiBooks
( Id int not null
 ,Author varchar(50) not null
 ,Title varchar(50) not null)

Insert into Books (Id, Author, Title)
Values (1, 'Martha Wells', 'All Systems Red')
      ,(2, 'Terry Mancour', 'Spellmonger')
      ,(3, 'Yahtzee Croshaw', 'Will Save the Galaxy for Food')
      ,(4, 'Andrew Rowe', 'Sufficiently Advanced Magic')
      ,(5, 'Brandon Sanderson', 'The Emperor''s Soul')
      ,(7, 'Lois McMaster Bujold', 'The Vor Game')

Insert into SciFiBooks (Id, Author, Title)
Values (1, 'Martha Wells', 'All Systems Red')
      ,(3, 'Yahtzee Croshaw', 'Will Save the Galaxy for Food')
      ,(4, 'Andrew Rowe', 'Sufficiently Advanced Magic')
      ,(5, 'Lois McMaster Bujold', 'The Warrior''s Apprentice')
      ,(6, 'Larry Niven', 'Ringworld')
      ,(7, 'Lois McMaster Bujold', 'Ceteganda')
```

2. Open the [web app][appURL].
3. Then paste the `Assert` and `Test` SQL into the app.

``` sql
-- Assert
Select Id, Author, Title
From Books
-- Test
Select Id, Author, Title
From SciFiBooks
```

4. Select the `Id` column as a key.

 ![results](/images/2020/sql-data-compare/book_query_in_app.PNG)

5. Copy the comparison script.

 ![results](/images/2020/sql-data-compare/book_query_in_app_copy.PNG)

6. Run the query in the test database. The output below will be produced.

As we can see, the query has correctly found the following:
* 3 rows are identical.
* Spellmonger is in `Books` but not `SciFiBooks`.
* Ringworld is in `SciFiBooks` but not `Books`.
* Two rows have discrepancies. The column specific discrepancy tables `Discrepant__Author` and `Discrepant__Title` show one and two discrepancies respectively. 

 ![results](/images/2020/sql-data-compare/book_comparison.PNG)

## Advanced Comparisons

I will go into examples of more complex comparisons in later posts. For now, you can see examples in the [usage guide][usage].

If you have any feedback, issues, or requests, please visit the project on [GitHub][project].

[project]: https://github.com/Siphonophora/SqlDataCompare
[appURL]: https://sqldatacompare.mjconrad.com/
[usage]: https://github.com/Siphonophora/SqlDataCompare/blob/master/docs/usage_guide.md
