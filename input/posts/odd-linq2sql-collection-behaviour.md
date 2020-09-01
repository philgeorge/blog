Title: Odd LINQ2SQL behaviour when filtering by a Collection
Published: 30/01/2013
Tags:
  - LINQ2SQL
  - C#
---

I recently experienced a couple of cases of unexpected outcomes when LINQ2SQL query syntax I'd written was translated to SQL.
Both are specific cases where I was trying to filter results from a DB table by a known collection of integers, `myIds`, using the following expression:

```
from n in Notifications
where myIds.Contains(n.Id)
select n
```

Can you guess what would happen in each of the following special cases?

1. If `myIds` is of type `IList<int>`

This results in a run time error: *Method 'Boolean Contains(Int32)' has no supported translation to SQL*.
The fix is easy: if you reference the same integer list as `IEnumerable<int>` then it works just fine!

2. If `myIds` is declared as type `IEnumerable<int>`, but is actually an empty list

This results in a query being executed, but the actual resulting SQL (as evidenced using Microsoft SQL Server Profiler) is this:
```
select a,b,c from Notification where 1=0
```
i.e. it will always return no rows from the database query.
(Note, I have simplified the literal SQL that was generated for readability - as we know, LINQ2SQL puts in lots of t1/t2/etc table aliases.)

This led me to recommend to our team of developers that we check wherever possible to avoid unecessary round trips to the database, i.e. write the C# as follows:
```
if (myIds != null && myIds.Any())
{
  var query = from n in Notifications where myIds.Contains(n.Id) select n;
  return query.ToList();
}
// ... return empty list or throw exception as appropriate
```
