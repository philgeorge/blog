Title: API First Design - can we do it?
Published: 09/10/14
Tags:
  - ASP.NET
  - design
  - architecture
---

Our current code base contains a mix of both ASP.NET MVC controllers serving up web pages and also Web API controllers for use by our iPhone and Android mobile apps. We try and avoid code duplication by making the action methods in both of these entry points call through to shared "service layer" classes.

It is common advice to [design your applications "API First"](https://www.google.com.au/search?q=api+first+design) so I often think about whether we could adopt this approach and remove the need to have two different sets of controllers. Our project has several hundred controller classes, so this is important!

However, there are important differences to the expectation we have of API functions when compared to rendering HTML web pages.

- A web page works one HTTP request at a time, and is optimised by only getting the data necessary for that one page view.
- Conversely, a mobile app holds state, and is therefore optimised by getting as much data as possible in one request (assuming that data does not go stale). Then the app user is able to navigate around screens without further calls to API.

So typically one web API GET action method encompasses multiple MVC GET action methods. POST action methods are likely to be more equivalent in the action they take, but not in what they return. The POST for a web API call usually need only return a success or failure indicator. Our web page POST action methods need to return an HTTP redirect in the case of success or the full original page HTML (with error messages) in the case of failure.

What you may have gleaned from all this is that we have quite a traditional HTML web application - it is based heavily around request/response across multiple pages. We need to do this because we have large corporate clients that still have their employees (our end users) on outdated browsers (e.g. IE8)

Therefore, I have concluded that API First design - at least in so far as utilising the same API for our web site and apps - is not appropriate for us at this point in time. My hope is that in a few years time (when everyone is using "evergreen" browsers), we will be able to move to a SPA (single page app) web site. This would then open the door to sharing the same interface to the server as our mobile apps currently do.
