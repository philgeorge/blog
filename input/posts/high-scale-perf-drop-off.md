Title: Diagnosing high-scale performance drop off
Published: 13 April 2014
Tags:
  - performance
  - ASP.NET
---

Once per year, in late May, we experience very high load on our [web site](http://www.gettheworldmoving.com) when hundreds of thousands of users all log in and start their 100-day journey experience. To make sure we are able to cope with this peak load, we carry out extensive load testing in the month beforehand. Every year, without fail, this gives us some new insights into how our code truly behaves at run time, and enables us to fine tune it.

![Memory chips](/posts/img/memory-chips.jpg){align="right"}
During the latest block of testing, we stressed the site to double our predicted load (tens of thousands concurrent users). We saw response times suddenly drop from an average of 2 secondss to 10 seconds. Clearly something had been pushed beyond its limit and snapped!

We looked at the usual monitoring points:
- no locks or excessively slow transactions in the database
- [NewRelic](http://www.newrelic.com) showed most time was spent in our application code
- our distibuted cache processes showed suspiciously high average memory usuage

This led me to suspect either too much was being stored in distributed cache, or else something in there was unreasonably large.
So I added diagnostic code to log all distributed cache inserts. 

```
var msg = GetSizeInfo(key, item, ids); // ids are the cache key parameters to uniquely identify each item, e.g. key="country", id="aus"
Trace.TraceInformation(msg);
// go on to put item in distributed cache...

private string GetSizeInfo<T>(string key, T item, params object[] ids)
{
    var size = GetObjectSize(item);
    var idString = string.Join(",", ids);
    var sizeKb = Math.Round(((double)size)/1024);
    var msg = string.Format("{0}({1})={2}KB", key, idString, sizeKb);
    if (sizeKb > 100)
    {
        Trace.TraceWarning("cache item > 100KB: " + msg);
    }
    return msg;
}

private static long GetObjectSize(object obj)
{
    long length;

    var stream1 = new MemoryStream();
    using (var writer = XmlDictionaryWriter.CreateBinaryWriter(stream1))
    {
        var serializer = new NetDataContractSerializer();
        serializer.WriteObject(writer, obj);
        length = stream1.Length;
    }

    if (length == 0)
    {
        var stream2 = new MemoryStream();
        var bf = new BinaryFormatter();
        bf.Serialize(stream2, obj);
        length = stream2.Length;
    }

    return length;
}
```

This meant that I could run the code locally and view in the trace output window every object that was going into distributed cache.
Nothing stood out here that shouldn't have been cached at all, but the special warning log for items over 100K *did* occur.

In fact I found an object that had crept up to 400K. It was an array of a couple of hundred geographical locations, but each location was repeated multiple times for each supported language (English, French, Japanese, etc) and extra translations had recently been added so that we now had about 16 different languages. When this code was first written we only supported 4 langauges!

Now, to render a particular page, this cached data was being pulled from cache multiple times. This was a popular page too, which pretty much every one of those tens of thousands of users would be viewing. So this 400K array would have been flooding the network between web servers and cache servers.

Having identified the cause of the perfomance problem, there were two solutions:

1. Split the cached object up so that the calling code only pulls out what it needs (e.g. just one location, and/or just one language).
2. Don't store this object in distributed cache at all.

In this case, I opted for the second option, because the cached data almost never changes, therefore an in-Process cache on each web server instance makes perfect sense and required no code changes at all.

Hey presto, response times went back down to 2 secs and we can now scale beyond our previous limit!

Thanks to [JDS](http://www.jds.net.au) for their expertise in the HP Loadrunner product and executing the tests with us.

*At the time of writing this article, we used the Microsoft Azure AppFabric component for distributed cachine, deployed in a cloud service worker role. Since then, we have migrated our distributed cache to use the MS Azure managed Redis service, which has the benefit of requiring zero build and deployment by our team. The diagnostic code shown is still around though, because we are just as likely to need to monitor what we are putting into Redis.*
