Title: A Functional Approach With C#
Published: 24/09/2014
Tags:
  - functional
  - C#
---

I have fond memories of Scheme, Lisp and Standard ML being amongst the first programming
languages taught to me as an undergraduate Computer Science student at [Southampton University](http://www.ecs.soton.ac.uk).

With the recent upsurge in popularity of functional languages in mind, I've been thinking about how to apply functional programming techniques
to help us write better C# code in our day to day work.

Here's my top 3 recommendations for better C# code that arise from functional practices:

## Don't re-use classes for different purposes

The context here is returning objects from a SQL repository query method.

Unlike most functional languages, declaring model classes in C# is quite a hassle (separate file, class declaration, etc), so it's tempting to re-use classes from elsewhere.

For example, you may have a fully-featured Customer model object with a dozen properties for editing a Customer's details, but now you need a customer search page which displays a list of customer's names as search results. The easy option is to use your existing Customer object in an IList or IEnumerable collection, after all, it's logically the same thing, right?

But wait! If you return search results from a SQL query into a list of full Customer model objects, you would likely end up with many fields left unset, at their default values. This would result in misleading code - objects could be picked out of this list and passed to other functions/methods that take a Customer object but assume it is a fully populated object.

My recommendation is to always make the effort to declare and use a separate object, e.g. CustomerListItem, which has just the one or two properties needed to display in the search results. The benefit of this is that it enforces a separation of concerns so that if a changes are made to one of the two aforementioned pages, it has less chance of unexpectedly impacting the other.


## Return data in instances of classes of exactly the right shape for the data you have

Let's switch to a different example: a UserProfile object like this:

```
public class UserProfile
{
    public int UserId { get; set; }
    public string Email { get; set; }
    public DateTime JoinedDate { get; set; }
    public int RankId { get; set; }

    public string RankDescription { get; set; }
    pulibc bool IsMyProfile { get; set; }
}
```

The first four properties come from your database table, but the other two are contextual, relevant to the current user logged in to the system:
- RankDescription is a description of this user's "rank", e.g. novice/expert/etc. and is localised (English, French, Japanese, etc.) for the language preference of whoever is viewing the page
- IsMyProfile is a boolean flag that indicates whether the logged in user is viewing their own profile, or that of another user

My first inclination here is to write a SQL query to populate the object with it's first 4 properties, then write further C# code that figures out values for the other 2.
Most likely, populating the inital 4 fields would occur in a repository or data access function that returns the object. Then a function in another layer of code would modify those remaining 2 properties and return it up to the view.

However, if we take on the immutable object mindset of a functional programmer, instead we can create two separate classes, as follows:
```
public class UserProfileData
{
    public int UserId { get; set; }
    public string Email { get; set; }
    public DateTime JoinedDate { get; set; }
    public int RankId { get; set; }
}

public class UserProfile
{
    public UserProfileData Data { get; set; }
    public string RankDescription { get; set; }
    public bool IsMyProfile { get; set; }
}
```

Then our repository method returns the UserProfileData object. This never gets modified, and it can be safely cached without any personalised or localised extras.
The next layer of code creates its own new UserProfile object that has the data object contained within, plus the extras.
As well as the ability to cache the DB data safely and correctly, this means (similarly to recommendation 1 above) at no point does a function return a partially populated object, which could lead to mistaken attempts of reuse.


## Use lambdas / functions to specialize behaviour

Here's an example of code where I needed to loop through a list of geographic locations (from a file), but take different actions in two different use cases.
1. Check if the locations are valid and print out all errors where they're not.
2. Import the locations into a database table.

The most naive implementation might be to code up a function for case 1, then copy and paste it into a function for case 2. Obviously, this violates the [Don't Repeat Yourself](https://en.wikipedia.org/wiki/Don't_repeat_yourself) principle, which reduces maintainability.

So to keep our code DRY, we might have one function that loops through the locations, but it takes a boolean flag indicating whether this is an actual import or just a validation. Then within the function, we check this flag and branch into alternate code as necessary.
This approach is an improvement, but can lead to long, hard to read functions.

If we use the approach of composing one function from other functions then the code naturally breaks down into more separate, readable, testable units, plus it avoids the flag checking and branching code. The .NET generic types Action<> and Func<> are excellent for this purpose.

Here's what I ended up with:

```
        // case 1
        public ErrorCollection<ImportLocationError> ValidateLocations(IList<Location> locations)
        {
            var errors = new ErrorCollection<ImportLocationError>();
            ProcessLocations(locations, e => { errors.Add(e); }, l => { });
            return errors;
        }

        // case 2
        public void ImportLocations(int courseId, Member member, IList<Location> locations)
        {
            var memberId = member.MemberId;
            ProcessLocations(locations,
                e => { throw new Exception(e.ToString()); },
                l => { _locationRepository.Create(courseId, l.Location, l.MapData, memberId); }
                );
        }

        private void ProcessLocations(IList<Location> locations, Action<ImportLocationError> errorHandler, Action<Location> newLocationHandler)
        {
            foreach (var location in locations)
            {
                var geoNamesResponse = _geoNamesService.Get(location.MapData.Longitude, location.MapData.Latitude);
                if (geoNamesResponse.Error == null)
                {
                    var country = _countryService.GetCountryIncludingCourseOnlyByCode(geoNamesResponse.CountryCode, (int)LocaleId.English);

                    if (country == null)
                    {
                        errorHandler(ImportLocationError.UnableToFindCountry);
                        continue;
                    }
                    else
                    {
                        location.CountryId = country.CountryId;
                    }
                }
                else
                {
                    errorHandler(ImportLocationError.GeoLookupFailed);
                    continue;
                }

                if (newLocationHandler != null)
                {
                    newLocationHandler(location);
                }
            }
        }        
```
