Title: Minimising DB round trips in LINQ2SQL
Published: 2 April 2013
Tags:
  - performance
  - LINQ2SQL
---

![Server talking to database](/posts/img/db-round-trips.png){align="right"}
A well-known good practice for good website performance is to minimise the number of round trips between your web server and the database server. The classic example of how things go bad if you don't have an eye on this is the ["select n+1 problem"](http://stackoverflow.com/questions/97197/what-is-the-n1-selects-issue). 

I recently spotted a much more subtle case of unecessary database round trips in our C# LINQ2SQL code:

```
var steps = db.StepsTargets.Any(s => s.MemberID == memberId);
var stepTarget = steps ? db.StepsTargets.First(s => s.MemberID == memberId) : new StepsTarget();
```

The code above makes logical sense: check if there's any rows in the StepsTarget table for the given MemberID, if there is then fetch it, otherwise create a new one.
If you are familiar with LINQ2SQL, you will know that a call to the database is made on the Any() method and also on the First() method. This means that in the common case where a row *does* already exists, there will be 2 separate database hits. When you think about it in these terms, we can re-write the code in this way:

```
var stepTarget = db.StepsTargets.FirstOrDefault(s => s.MemberID == memberId) ?? new StepsTarget();
```

This removes the separate check for existance of rows and simply attempts the fetch, using a new empty row object if nothing was found. So there's never more than 1 database hit.

We can take a similar approach when doing an insert/update, i.e. attempt to fetch existing row by its key and then create a new one if not found:

```
using (var db = GetDatabaseForUpdate())
{
    var stepTarget = db.StepsTargets.FirstOrDefault(s => s.MemberID == memberId);
    if (stepTarget == null)
    {
        stepTarget = new StepsTarget {MemberID = memberId};
        db.StepsTargets.InsertOnSubmit(stepTarget);
    }
    stepTarget.Q1Target = q1Target ?? stepTarget.Q1Target; // only replace value already in DB if it has been provided
    stepTarget.Q2Target = q2Target ?? stepTarget.Q2Target;
    stepTarget.Q3Target = q3Target ?? stepTarget.Q3Target;
    stepTarget.Q4Target = q4Target ?? stepTarget.Q4Target;
    db.SubmitChanges();
}
```

Note that InsertOnSubmit can be called at the point you create the entity, doesn't have to be after you've set its values. This allows us to keep it cleanly in the one if statement as shown above.

