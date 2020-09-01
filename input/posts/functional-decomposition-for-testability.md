Title: Functional Decomoposition For Testability
Published: 02/04/2017
Tags:
  - functional
  - testing
---

![test tubes of separate colours](/posts/img/test-tubes.jpg){align="right"}
In this article, I  describe a very simple way to break down your business logic code, in a way that makes testing easier.

Consider you are required to write some code to display a chart of spending habits for a particular day. You alreay have the raw data stored and accessible through a repository class that implements this interface:

```
public class DataPoint
{
    public int Hour; // 0..23
    public double Dollars;
}

public interface IDataRepository
{
    IList<DataPoint> GetData(int userId, DateTime day);
}
```

The requirement for the chart is to display a data point for every hour of the day, even if there was no dollar spend recorded in the stored data.

Your initial thought for how to implement this requirement might be something like this:
```
public class ChartMaker
{
    private readonly IDataRepository _repository;

    public ChartMaker(IDataRepository repository)
    {
        _repository = repository;
    }

    public ICollection<DataPoint> GetChartData(int userId, DateTime day)
    {
        var data = _repository.GetData(userId, day);
        var array = new DataPoint[24];
        //todo: fill array with either points from the repository or points with $0
        return array;
    }
}
```

However, if you think about how to test the GetChartData function, then there are twp obvious pain points:
1. You would have to create a mock object to return the repository data and set it up differently for each test
1. You'd have to supply a userId and day to every test that merely gets passed through to the repository

Overall, just one test would look something like this (using [nunit](//www.nunit.org) and [Moq](https://github.com/moq/moq)):
```
[TestFixture]
public class ChartMakerTests
{
    [Test]
    public void ChartContains24Hours()
    {
        var mockRepo = new Mock<IDataRepository>();
        mockRepo.Setup(r => r.GetData(It.IsAny<int>(), It.IsAny<DateTime>())).Returns(new List<DataPoint>());
        var maker = new ChartMaker(mockRepo.Object);
        var data = maker.GetChartData(100, DateTime.MinValue);
        Assert.AreEqual(24, data.Count);
    }
}
```

With these pain points in mind, "extract method" is an appropriate re-factor to apply:
```
public ICollection<DataPoint> GetChartData(int userId, DateTime day)
{
    var data = _repository.GetData(userId, day);
    return MakeArray(data);
}

public static ICollection<DataPoint> MakeArray(IList<DataPoint> data)
{
    var array = new DataPoint[24];
    //todo: fill array with either points from the repository or points with $0
    return array;
}
```

Note that I make the extracted MakeArray method public so that it can be accessed by the test class. As a result of this extraction the method has become static, which shows that no dependencies exist and mocking is no longer necessary.

Consequently, the same unit test originally shown above, is significantly simpler:
```
[Test]
public void ChartContains24Hours()
{
    var data = ChartMaker.MakeArray(new List<DataPoint>());
    Assert.AreEqual(24, data.Count);
}
```

What we have done here is decompose the original implementation into separate functions, isolating the key business logic of transforming the data. That transformation (ChartMaker.MakeArray) is the only thing we need to unit test (assuming that our overall system has some higher level integration tests that verify everything is wired up correctly).

Given that in a complete implementation, we might have 5 or more unit tests to verify the business logic, this decomposed design will save us plenty of typing test setup code and considerably simplify that test class.

