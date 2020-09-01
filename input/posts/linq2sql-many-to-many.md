Title: LINQ2SQL query with more than one many-to-many entity relationships
Published: 30/06/14
Tags:
  - LINQ2SQL
  - performance
---

![resource-locale-category entity relationship diagram](/posts/img/resource-erd.png){align="right"}
Imagine we want to retrieve a list of rows from the Resource table, each of which has many related LocaleIds and many related CategoryIds. That is, we have many-to-many relationships between Resource and both of the other tables.

We can write the following, in LINQ2SQL query syntax:
```
var query = from r in db.Resources
            select new Resource
            {
                ResourceId = r.ResourceID,
                Title = r.Title,
                FileSize = r.FileSize,
                PhysicalFileName = r.PhysicalFileName,
                LocaleIds = db.ResourceLocales.Where(rl => r.ResourceID == rl.ResourceID).Select(rl => rl.LocaleID).ToList(),
                CategoryIds = db.ResourceCategories.Where(rc => rc.ResourceID == r.ResourceID).Select(rc => rc.ResourceCategoryID).ToList()
              };

var resources = query.ToList();
```

If we trace the SQL generated from this query, the first nested select maps into an outer join and nested count, but the second becomes multiple separate SQL queries that are executed in a loop after the initial query.

Clearly this is something that will kill the performance of the code in question. Also it's kind of unexpected because LINQ2SQL queries can express the multiple many-to-many relationships, but plain old SQL cannot - at least not in a way that the LINQ2SQL engine is happy to generate.

So having discovered this little surprise, what can we do about it?

One option is not to call `.ToList()` on the nested queries. This would leave them as IQueryable<T> properties, to be lazy loaded whenever the property is accessed. However, that would require the DataContext to remain in scope, and [we have previously found that to be a bad idea](when-to-dispose-linq2sql-datacontext). I want all the required data to be completely populated on my list of resources by the time this function exits.

Therefore my preferred solution is to accept that we cannot do it all in one single query, so instead we explicitly execute two queries and then stitch the results back together:

```
var query = from r in db.Resources
            select new Resource
            {
                ResourceId = r.ResourceID,
                Title = r.Title,
                FileSize = r.FileSize,
                PhysicalFileName = r.PhysicalFileName,
                LocaleIds = db.ResourceLocales.Where(rl => r.ResourceID == rl.ResourceID).Select(rl => rl.LocaleID).ToList()
                // don't attempt to populate CategoryIds here
              };
var resources = query.ToList();

var categoryQuery = from rc in db.ResourceCategories
                    select new { rc.ResourceID, rc.ResourceCategoryID };
var categories = categoryQuery.ToList();

foreach (var resource in resources)
{
  resource.CategoryIds = categories.Where(c => c.ResourceID == resource.ResourceId).ToList();
}
```

This results in exactly two round trips to the database, followed by a little in memory object/list creation within the foreach loop.
