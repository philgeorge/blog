Title: ASP.NET MVC Partial View gotcha
Published: 22/03/2018
Tags:
  - ASP.NET
  - partial view
  - debugging
---

![russian dolls](/posts/img/russian-doll.jpg){align="right"}
In our large MVC project, we have quite often decomposed views into pages containing partial views. For example, if a profile widget is used on multiple different pages, then it becomes a partial view (MiniProfile.cshtml). We would then have the model object used by the partial view as a child property on the model object of the main view. For example:

```
public class MyPageViewModel
{
    public string Title { get; set; }
    public MiniProfileViewModel MiniProfile { get; set; }
}

public class MiniProfileViewModel
{
    public string Name { get; set; }
    public string AvatarUrl { get; set; } 
}
```

The child partial is then included in the main view using `@Html.RenderPartial("MiniProfile", Model.MiniProfile)` directive. 

 Today I had an interesting debugging experience when I received this error message on viewing an ASP.NET MVC page:
`The model item passed into the dictionary is of type 'PhilTest.MyPageViewModel', but this dictionary requires a model item of type 'PhilTest.ChildViewModel'.`

In this case the error message was actually a bit misleading. The view model passed into the partial view wasn't the wrong type at all, it was just `null`.
So when the child object is null, ASP.NET seems to think that the partial view has been passed the type of the parent object.

I had assumed that it would be OK to have the child object being null, as long as I checked for this in the child view:
```
@Model MiniProfileViewModel
@if (Model != null)
{
    <div class="mini-profile">
        <span class="name">@Model.Name</span>
        <img src="@Model.AvatarUrl">
    </div>
}
```
However, it seems ASP.NET will raise the exception before it gets to render the view, therefore the solution is to simply move the check into the main parent view:
```
@if (Model.MiniProfile != null)
{
	@Html.RenderPartial("MiniProfile", Model.MiniProfile)
}
```
Unfortunately that means you may have to duplicate this "not null" check across multiple views. Perhaps this is a code smell that is telling us to avoid nullable properties, also referred to by me as ["partially inflated objects"](/posts/functional-approach-with-csharp), in the first place?
