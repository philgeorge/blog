Title: The most common async/await pitfall
Published: 08/09/2019
Tags:
  - C#
  - async
---

![threads](/posts/img/threads.jpg){align="right"}
The .NET async/await pattern for writing non-blocking code is great, but there's lots of way you can get into trouble with it. In many cases, either the compiler or a static analysis tool like Resharper can tell you that you have forgotton something.

For example, if you call an async method in the same class, the compiler will give you this warning:
`warning CS4014: Because this call is not awaited, execution of the current method continues before the call is completed. Consider applying the 'await' operator to the result of the call.`

This prompts you to add the await, the warning goes away, and you code does the right thing.

However, what if the async method that you are calling is in another class _and_ behind an interface, like this:
```
    public interface IDummyService
    {
        Task DoUpdate();
    }

    public class DummyService : IDummyService
    {
        public async Task DoUpdate()
        {
            await Task.Delay(2000);
        }
    }
```
This is the calling code:
```
    [HttpPost]
    public int Post([FromBody] string value)
    {
        var id = Create(value);
        _service.DoUpdate(); // no warning!
        return id;
    }
```

In this case, you don't get any warning and the code compiles (and seemingly runs) just fine.
This is because the interface method is declared to return a Task. The calling code doesn't know at compile time that what it's calling is an async method. Therefore it is awaitable, but it cannot be certain you should await it.

So what would be the consequences in this case? The controller action always returns and appears to have completed successfully. The implementation of IDummyService.DoUpdate() is getting called, but it just starts a Task and then returns. I have found that in the case where a similar method performed a database insert, about 30% of the time the insert never completed. However because there were no compile time or run time errors, this mistake went unnoticed for several months before something else alerted me to the missing data!

The lesson here is that the compiler (or other developer tools such as Resharper) won't always be there to save you. Keep your human eyes open for this pitfall in code reviews or pairing!