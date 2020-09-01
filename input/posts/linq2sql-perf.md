Title: LINQ2SQL Query Performance Tuning
Published: 12 May 2011
Tags:
  - performance
  - LINQ2SQL
---
Quite often I will look at a block of code and think that a certain refactor or modification might make a worthwhile performance or scalability improvement, but I don't have the time or means to actually test and verify my idea.

Well I recently had the good fortune to be performance tuning a web application with the assistance of a consultant on hand with HP LoadRunner to simulate high load and monitor the site in real time.

This was very satisfying because it allowed me to immediately see the benefits of a change when the site was being visited by a simulated 1000 concurrent users.

I had one very successful period of about two hours where a whole bunch of things just seemed to leap out at me and they all had the desired affect.

The starting point is when a function in the site is identified as having slow response time. Locate the source code that implements this function (e.g. the controller - action in an ASP.NET MVC site).

First, I reviewed the overall code structure with a critical eye. Unless you've pair-programmed or peer-reviewed every inch of the way preceding this point in time, it's amazing what you can find. I am lucky enough to work among a team of exceedingly capable and diligent fellow programmers, but mistakes still find their way of creeping in to our code. On this occassion I found (and fixed) two things:

1. The code to query the database was being executed twice.
2. The code to cache the response data was silently failing, thereby requiring the DB fetch to be repeated the next time the user visited the same page.

We use LINQ to SQL for database access. My next step was to look really closely at all LINQ to SQL queries executed as part of the slow responding function. Consider how the queries will relate to the underlying database table. In this case I noticed three things:

1. The query was joining to a table that it didn't need data from (the column it did need was available on another table in the query); I removed the join.
2. The query was ordering by a created date field. However I knew that ordering by the primary key ID field would acheive the same result because rows are inserting chronologically, as the data is created. Since the SQL Server table was not indexed on the created date column, I changed the ordering to use the primary key.
3. The query included nested queries in the where part, something like this:
```
    let teamTrackedIds = db.MemberTrackTeams.Where(tt => tt.MemberID == memberId).Select(tt => tt.TeamID)
    from n in db.Notifications where teamTrackedIds.Contains(n.SourceTeamID)
    select n
```

I happen to know that this will map to a SQL "in" condition within the where clause. I also learnt (from an Informix DBA many years ago) that databases much prefer to do joins than nested in clauses.

So I rewrote this query to use a join, like this:
```
from n in db.Notifications
join tt in db.MemberTrackTeams on n.SourceTeamId equals tt.TeamId
select n
```

In fairness, there were a lot of other parts to the actual query I was looking at. My example above has been simplified to make the change very easy to follow, but it also might make you think that it was silly to write it the nested way in the first place. That wasn't so obvious in the original code.

The amazing thing about this sequence of finds was that each one reduced the response time by a significant meaurable amount. Combined together, in our maximum load scenario (tens of thousands of users concurrently hitting the site), the slowest page's response time dropped from about 2.5 seconds to 1 second and the second slowest page went from 2 seconds to just fractionally over 1.

A big win for users of the site, and a great day for load-testing! 