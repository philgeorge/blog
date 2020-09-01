Title: Paging in ASP.NET MVC
Published: 10/01/2015
Tags:
  - ASP.NET
  - functional
  - html
  - C#
  - design
---

One thing that really bugs me is to find blocks of code that have been copied and pasted all over a code base. Of course this violates the DRY principle, so we should always look for refactorings that can remove the duplication. A case that I came across recently was copied HTML paging UI components - the *next*, *previous* links that live at the bottom of any list view page. It doesn't seem like a lot of code to copy, but there's the little rules such as "only show the *next* link if you're not already on the last page" which you really don't want coded separately everywhere. The important thing was that the URLs needed for the *next* or *previous* links were different for each list page (e.g. list of customers versus list of products).

We already had a fairly nice generic class for paging data, called `PagedList<T>`.
Therefore I have attempted to design a paging system in ASP.NET MVC that meets the following criteria:

- the paging UI controls should be a single, re-usable partial view
- makes no assumptions about URL/route format, other than that searches are using HTTP GET

The remainder of this post describes the solution I came up with.

## The Model

For illustration here, let's assume the data objects I am displaying a paged list of are of this very simple Product class:
```
public class Product
{
    public int ProductId;
    public string Name;
    public double Price;
}
```

And here's a view model that I can create from my MVC controller easily:
```
public class ProductViewModel
{
    public string NameFilter;
    public int? MaxPrice;

    public IPagedList<Product> Products;

    public ProductViewModel(string nameFilter, int? maxPrice, int? pageNumber)
    {
        NameFilter = nameFilter;
        MaxPrice = maxPrice;
        const int pageSize = 20;
        Products = ProductRepository.Get(nameFilter, maxPrice, pageNumber ?? 1, pageSize);
    }
}
```

I've created this dummy repository that creates and filters a list of 1000 random products, using the handy little [Faker.NET.Portable nuget library](https://github.com/AdmiringWorm/Faker.NET.Portable/):

```
public class ProductRepository
{ 
    private static readonly List<Product> List = GenerateFakeProdcuts();

    public static IPagedList<Product> Get(string name, int? maxPrice, int pageNumber, int pageSize)
    {
        var products = List as IEnumerable<Product>;
        if (!string.IsNullOrEmpty(name))
        {
            products = products.Where(p => p.Name.IndexOf(name, StringComparison.OrdinalIgnoreCase) >= 0);
        }
        if (maxPrice != null)
        {
            products = products.Where(p => p.Price <= maxPrice.Value);
        }
        products = products.OrderBy(p => p.Name);
        return new PagedList<Product>(products, pageNumber, pageSize);
    }

    private static List<Product> GenerateFakeProdcuts()
    {
        var list = new List<Product>();
        for (var i = 0; i < 1000; i++)
        {
            list.Add(new Product
            {
                ProductId = i,
                Name = Faker.Company.Name() + " " + Faker.App.Name(),
                Price = Faker.RandomNumber.Next(1, 1000)
            });
        }
        return list;
    }
}
```

Where the repository creates a `PagedList<Product>` it is using this class that holds the page of actual list items, plus all the related paging data such as page size, current page number, etc.:
```
[Serializable]
public class PagedList<T> : IPagedList<T>
{
    [NonSerialized]
    private Func<int, string> _getUrlForPageFunc; // must be a private field (instead of auto property) so that serialization can be supressed
    public int TotalItems { get; private set; }
    public int TotalPages { get; private set; }
    public int PageNumber { get; private set; }

    public int PageSize { get; private set; }

    public string GetPageUrl(int page)
    {
        if (GetUrlForPageFunc == null)
        {
            throw new NotImplementedException("After creating a PagedList<T> of items, you must implement the GetUrlForPageFunc callback in your controller in order for next/previous page links to work");
        }
        var url = GetUrlForPageFunc(page);
        return url;
    }

    [JsonIgnore]
    public Func<int, string> GetUrlForPageFunc
    {
        get { return _getUrlForPageFunc; }
        set { _getUrlForPageFunc = value; }
    }

    public int NextPageNumber
    {
        get { return PageNumber < TotalPages ? PageNumber + 1 : TotalPages; }
    }

    public int PreviousPageNumber
    {
        get { return PageNumber > 1 ? PageNumber - 1 : 1; }
    }

    public bool IsLastPage
    {
        get { return NextPageNumber <= PageNumber; }
    }
    public bool IsFirstPage
    {
        get { return PreviousPageNumber >= PageNumber; }
    }

    public ICollection<T> CurrentPage { get; set; }

    /// <summary>
    /// Create a paged list with the current page only selected from the given query.
    /// </summary>
    /// <param name="query">The query that would fetch all items (not just a single page)</param>
    /// <param name="pageNumber"></param>
    /// <param name="pageSize"></param>
    public PagedList(IQueryable<T> query, int pageNumber, int pageSize)
    {
        Initialise(pageNumber, pageSize, query);
    }

    /// <summary>
    /// Create a paged list with the given collection representing all items, and the current page filtered from this.
    /// </summary>
    /// <param name="items">All the items</param>
    /// <param name="pageNumber"></param>
    /// <param name="pageSize"></param>
    public PagedList(IEnumerable<T> items, int pageNumber, int pageSize)
    {
        Initialise(pageNumber, pageSize, items.AsQueryable());
    }

    /// <summary>
    /// Create a paged list with the current page of items as the given collection, and the overall size of the list specified.
    /// </summary>
    /// <param name="items">The items for just the current page</param>
    /// <param name="pageNumber"></param>
    /// <param name="pageSize"></param>
    /// <param name="totalItems"></param>
    public PagedList(IEnumerable<T> items, int pageNumber, int pageSize, int totalItems)
    {
        Initialise(pageNumber, pageSize, totalItems, items);
    }

    /// <summary>
    /// Create a single page of all the items in the given collection.
    /// </summary>
    public PagedList(IEnumerable<T> items)
    {
        PageSize = 0;
        TotalPages = 1;
        PageNumber = 1;
        CurrentPage = items.ToArray();
        TotalItems = CurrentPage.Count;
    }

    private void Initialise(int pageNumber, int pageSize, IQueryable<T> query)
    {
        ValidatePageSize(pageSize);
        PageSize = pageSize;
        TotalItems = query.Count();
        TotalPages = GetTotalPages();
        PageNumber = GetPageNumber(pageNumber);
        CurrentPage = query.Skip((PageNumber - 1) * PageSize).Take(PageSize).ToArray();
    }

    private void Initialise(int pageNumber, int pageSize, int totalItems, IEnumerable<T> items)
    {
        ValidatePageSize(pageSize);
        PageSize = pageSize;
        TotalItems = totalItems;
        TotalPages = GetTotalPages();
        PageNumber = GetPageNumber(pageNumber);
        CurrentPage = items.ToList();
    }

    private int GetPageNumber(int pageNumber)
    {
        return pageNumber > TotalPages ? TotalPages : (pageNumber <= 0 ? 1 : pageNumber);
    }

    private int GetTotalPages()
    {
        return (TotalItems - 1) / PageSize + 1;
    }

    private static void ValidatePageSize(int pageSize)
    {
        if (pageSize <= 0) throw new ArgumentOutOfRangeException("pageSize", "pageSize must be greater than 0");
    }
}
```

The PagedList<T> class gives us several ways of constructing a PagedList, where we can provide all the items and it will filter out the current page for us, or just the items for the current page where we tell it how many items in total there would have been. I am using the first of these options, as seen earlier in my ProductRepository.

PagedList<T> implments the IPagedList<T> interface, which, in turn, implments the IPageable interface.
- IPagedList<T> is the interface that each view uses to display the actual list of items
- IPageable is the interface is that the re-useable Paging UI component will uses

```
public interface IPageable
{
    int TotalItems { get; }
    int TotalPages { get; }
    int PageNumber { get; }
    int PageSize { get; }
    /// <summary>
    /// Set this property if you want to display and use a drop down list of alternate page sizes, e.g. 10,50,100
    /// </summary>

    int PreviousPageNumber { get; }
    int NextPageNumber { get; }
    bool IsLastPage { get; }
    bool IsFirstPage { get; }

    string GetPageUrl(int page);
    /// <summary>
    /// Set this property to a callback function in your controller to allow the Pager UI compmonent to render hyperlinks for next/previous etc.
    /// </summary>
    Func<int, string> GetUrlForPageFunc { get; set; }
}

public interface IPagedList<T> : IPageable
{
    ICollection<T> CurrentPage { get; set; }
}
```

Next up is the MVC controller that creates a view model of Products given the user's search filters:
```
public class ProductController : Controller
{
    public ActionResult Index(string nameFilter, int? maxPrice, int? pageNumber)
    {
        var model = new ProductViewModel(nameFilter, maxPrice, pageNumber);
        model.Products.GetUrlForPageFunc = page => Url.Action("Index", new { nameFilter, maxPrice, pageNumber = page });
        return View(model);
    }
}
```

Notice here how we set a callback function that generates the URL for any given page (using the MVC UrlHelper class). This is important because it keeps the knowledge of generating URLs in the correct layer of code (i.e. the controllers) instead of either hard-coded in the model. Effectively, the Pager will be asking the controller what is the URL for a particular page number as it needs them. This means that the we will work with any format of URL routes (e.g. either "/customer/list/2?nameFilter=smith" or "/customer/list?page=2&nameFilter=smith") and it's easy for the URL to retain context of any search filters from one page to the next.

Let's take a look at the MVC View for this controller action:
```
@model ExamplePagingWebSite.Models.ProductViewModel
@{
    ViewBag.Title = "Product List";
}
<h3>Filter</h3>
@using (Html.BeginForm("Index", "Product", FormMethod.Get))
{
    <fieldset>
        @Html.LabelFor(m => m.NameFilter, "Name")
        @Html.TextBoxFor(m => m.NameFilter)
    </fieldset>
    <fieldset>
        @Html.LabelFor(m => m.MaxPrice, "Max Price")
        @Html.TextBoxFor(m => m.MaxPrice, new {type = "number", min = "1", max = "1000"})
    </fieldset>
    <button type="submit">Search</button>
}
<h3>@Model.Products.TotalItems matching products</h3>
<table cellpadding="4">
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Price</th>
        </tr>
    </thead>
    <tbody>
        @foreach (var product in Model.Products.CurrentPage)
        {
        <tr>
            <td>@product.ProductId</td>
            <td>@product.Name</td>
            <td>@product.Price.ToString("C")</td>
        </tr>
            }
        <tr></tr>
    </tbody>
</table>
@Html.Partial("_Pager", Model.Products)
```

This view can access all it needs from IPagedList<Product> to render to the current page of the list, including (as shown) the total matching products.
The Pager partial included at the bottom is as follows:
```
@model ExamplePagingWebSite.Models.IPageable
<nav class="pagination">
    @if (Model.PageNumber > 1)
    {
        <span>
            <a href="@Model.GetPageUrl(Model.PreviousPageNumber)" id="prev">
                <
            </a>
        </span>
    }
    <span>page @Model.PageNumber of @Model.TotalPages</span>
    @if (Model.PageNumber < Model.TotalPages)
    {
        <span>
            <a href="@Model.GetPageUrl(Model.NextPageNumber)" id="next">
                >
            </a>
        </span>
    }
    <div>
        <select id="pageNumber" name="pageNumber" onchange="window.location.href = this.options[this.selectedIndex].value">
            @for (var i = 1; i <= Model.TotalPages; i++)
            {
                <option value="@Model.GetPageUrl(i)" @(Model.PageNumber == i ? "selected='selected'" : "")>@i</option>
            }
        </select>
    </div>
</nav>
```

This page only accesses what it needs from the IPageable interface. Again, notice the callback to Model.GetPageUrl() in number of spots for links.

This exact same partial can now be included on any paged list view, without modifications, thus achieving my original goal of DRY, not copied, code.
