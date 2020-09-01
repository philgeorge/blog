Title: When to dispose your LINQ2SQL DataContext
Published: 27/10/2009
Tags:
  - LINQ2SQL
  - C#
---

*This post only relates to LINQ to SQL when used as an ORM for a website.*


Simple LINQ to SQL examples tend to be of the form:

```
public IEnumerable<Categories> GetAll()
{
	using (MyDataContext db = new MyDataContext())
	{
		var q = from c in db.Categories where c.Deleted == false select c;
		return q.ToList();
	}
}
```

Once your web page becomes a bit more complex (i.e. realistic), you will typically need to display more than one piece of data on it.
For example, there might be user details in a header, product categories in a menu, and a main area with product details.
In a well-factored data access layer, you will have separate methods (possibly even on separate classes) for fetching each of these different sets of data.

So how do you control the lifetime of your DataContext then? We have read that the DataContext is cheap to create and discard,
but it still seems wasteful to do so when we know that our code will be doing 3 or 4 consecutive data access calls.
And say you know that the same entity will be queried more than once in this sequence of calls:
for example a user entity is retrieved to show the logged in user's name and then again to check their preferences for displaying metric or imperial measurements.
In this case we clearly want to keep using the same DataContext so that it can track and return the same instance of the user entity to the second request.

Having realised this, the temptation arises to define the lifetime of your DataContext object to be aligned with the HttpContext.
ASP.NET creates you a fresh HttpContext object for each web request that it handles.
It makes sense that any data access work done to render the page for this request could use the same LINQ to SQL DataContext:
this would guarantee to avoid any repeated database hits for queries of the same entity.
Most IOC containers make scoping an object to HttpContext very easy. And this will work - up to a point.

There is a problem with the HttpContext scoped DataContext, and it took major performance issues on a production web site for me to understand it.
If your LINQ to SQL data model is quite large (i.e. more than about 25 tables), and you make 3 or 4 queries on it to render a page,
then the DataContext can start to occupy a significant amount of memory.
Not huge, but if used on a website that might be receiving 30-50 concurrent page views, it adds up.
Our production server quickly ran out of memory when put under load, and analysis showed that most of the objects in memory where DataContext entities and collections.

We used a number of techniques to reduce the memory consumption, such as removing unimportant data from popular pages,
increased caching of data and turning off object tracking on the DataContext for read-only data access.
However, one coding change that resulted in a big reduction in memory usage was to adopt a pattern of destroying our DataContext objects as soon as possible.
That is, don't wait until the HttpContext is completely finished with:
that leaves the DataContext hanging around whilst ASP.NET is still firing page events, generating ViewState and rendering HTML.
And, I believe crucially, in a high-load web site, further HTTP requests will be coming in while these latter stages of the ASP.NET lifecycle are going on,
so if you've been able to free up memory by then, so much the better.

So my recommendation is not to use HttpContext scoped DataContext, unless you are sure that you will not have to support a high volume of page requests.
Instead, design your code so that data access is explicit and the timing of it is controlled by you.
The MVC pattern is a great help here: you are guided to assemble all your data up front before control is passed off to the view.
Contrast that with an unstructured WebForms approach where it is common to just dip into your data access layer as necessary from various events:
PageLoad, Click, PreRender, etc.
