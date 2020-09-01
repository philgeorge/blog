Title: Moving to Dapper
Published: 14/10/2015
Tags:
  - sql
  - LINQ2SQL
  - dapper
  - C#
  - design
---

![Dapper man with cane](/posts/img/man-with-cane.png){align="right"}
Towards the end of 2008, when we started building our current code base, I recommended that the team use a new data access technology that Microsoft were shipping with Visual Studio: LINQ to SQL (or LINQ2SQL). It felt good to be getting on the ORM bandwagon with an "out of the box" Microsoft framework. Unfortunately, within a year, Microsoft had shifted their focus to Entity Framework and it became apparent the LINQ2SQL was a dead end product.

Despite some early performance issues, due to [my own misunderstandings](/posts/when-to-dispose-linq2sql-datacontext), it has actually served us well. On  occassions, I'd considered whether we ought to switch to Entity Framework, just to be "up to date" with current Microsoft recommendations, however it never appeared to have any distinct advantages that warranted the effort that would be involved.

Over the last year, a number of things have been frustrating me with our LINQ2SQL implementation:

## Maintaining the DBML file

In order to reference tables and columns in your C# code, those tables must be dragged and dropped on to a "DBML" design surface. Visual Studio can then generate data model classes and data access code.

However, if you combine a moderately large number of tables (more than 50) with a remote hosted database (we switched to using Azure SQL Database in 2012), the DBML designed becomes frustratingly slow to use. It got to the point where somebody adding a new table to the DBML was the signal for a team tea break (one person forced to wait, the others to commiserate).

Also every change to the DBML seemed to rewrite connection strings in the settings file, however careful we were about all team members setting things up the same. This makes a mess of version control history. Currently the config file contains this block (all five connection strings would be exactly the same, but rest assured it will change again with every commit/check-in):
```
<connectionStrings>
    <add name="Gtwm.Data.Properties.Settings.gcc_devConnectionString" ...
    <add name="Gtwm.Data.Properties.Settings.gcc_devConnectionString17" ...
    <add name="Gtwm.Data.Properties.Settings.gcc_devConnectionString1" ...
    <add name="Gtwm.Data.Properties.Settings.gcc_devConnectionString16" ...
    <add name="Gtwm.Data.Properties.Settings.gcc_devConnectionString10" ...
</connectionStrings>
```

## Comprehension of complicated queries

While LINQ2SQL query syntax is nice for simple queries, it can get very messy for larger ones. Outer joins and aggregation are two things where the syntax is less intuitive and more verbose than native SQL syntax.

So if you have a query that joins multiple tables, does some aggregation and perhaps throw in some nested subqueries too, then you can have a beast of a LINQ2SQL query that fills your screen. If you have to revisit such a query months later, or need to modify one written by another developer, then you inevitably end up spending most of your time trying to translate it into raw SQL just in order to be sure what it's doing and to reason about it. This is a bad sign - the abstraction that makes life easier in the simple cases is now making things harder!

## Performance of Updates

One of my golden rules for application performance and scalability is to [minimise round trips between your app code and your database](/posts/minimising-db-round-trips). But due to the ORM nature of LINQ2SQL, simple updates always require two round trips. For example, if I want to update a user's weight unit preference (kilograms or pounds), I'd do this (assuming I already have reference to the DataContext object, `db`):
```
var profile = db.MemberProfiles.FirstOrDefault(m => m.MemberID == memberId); // first DB hit here to fetch the row
profile.DefaultWeightKg = true;
db.SubmitChanges(); // second DB hit here to update the row
```
However, I can save myself the query round trip if I simply execute the SQL directly:
```
db.ExecuteCommand("update MemberProfile set WeightUnitKg = 1 where MemberID = {0}", memberId);
```

I know that there are concurrency reasons for the ORM first fetching the row back. It can then assert that nobody else has modified this same row when it does the update. However, in our application that is not a real concern. Our security model ensures that users can only update their own data, and if they had two concurrent browser sessions doing that, well, it doesn't really matter because it's just a profile flag. For us, overall throughput for thousands of concurrent users is much more important.

The upshot of this is that we adopted a strategy of writing all performance-sensitive updates using the direct SQL approach.

# Moving to Dapper

Given that the DBML modelling was painful and we were either thinking or actually writing in native SQL for much of our data access code, it made sense to look at the various Micro-ORM libraries that have sprung up in the .NET ecosystem. Dapper stood out due to its [active development](https://github.com/StackExchange/dapper-dot-net) and track record of [use in Stack Overflow](http://stackoverflow.com/tags/dapper/info).

So we tried it out for the next area of data access code that we were working on. These were the benefits:

- We need to know SQL anyway, so we're leveraging an existing developer skill rather than learning a new one
- If we need to optimise a SQL query, we do it entirely in SQL and that is what gets run
- Less lines of code in our data access methods
- Easier to intercept and wrap with our own helper function to do retries, [as recommended for Azure SQL DB](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-connectivity-issues)
- Most performant by default (e.g. minimal SQL, single round trip, etc)
- No more waiting for a DBML designer surface to do its thing before we can start writing code

There were only really two disadvantages:

1. We lose intellisense on database table and columns. This had made it easier to write code without checking the table structure, and gave us compile errors if a table changed (or at least if somebody changed it in the DBML). It also meant that you could find usages of a particular table or column using normal C# tooling (Visual Studio or Resharper).
2. We now have a mix of two data access techniques. Developers need to know which way is preferred for new code and also how to work with both in existing code.

For me, it was a no-brainer that we should proceed with Dapper moving forward. The intellisense benefits were rapidly waning as more code had been written to bypass the actual LINQ2SQL table objects.

However, it was important to pay attention to implications of having multiple data access technologies in our code. Jimmy Bogard explains it very clearly [here in a post on the Lava Layer anti-pattern](https://lostechies.com/jimmybogard/2015/01/15/combating-the-lava-layer-anti-pattern-with-rolling-refactoring/).
Taking this on board, we are committing to convert existing LINQ2SQL code to Dapper whenever we touch it for any other reason. I also know that we ought not introduce any other new data access technology to the code base until this migration is complete. But given LINQ2SQL lasted as our primary approach for nearly 7 years, I am confident that we will achieve this!

![Dapper emoji shrug](img/shrug.png){align="center"}
