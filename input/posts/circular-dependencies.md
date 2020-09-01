Title: Circular Dependencies (and how to avoid them)
Published: 01/01/2019
Tags:
  - C#
  - IOC
  - circular dependencies
  - dependency injection
---

![infinity loop](/posts/img/infinity-loop.jpg){align="right"}

When you are using an IOC container to manage dependency injection of service classes, it becomes very easy to have a lot of dependencies for any given class. While this is a good indicator that the class is not observing the Single Responsibility Principle, I'm going to focus this post on one of the more concrete problems that can arise in this situation: circular dependencies. That is, class A needs to be given an instance of class B when it's created, but class B needs an instance of class A for it to be created. It's the classic chicken and egg problem - you can't create one without the other!

I'll walk through a simple example of how a circular dependency is created, and then how you can resolve it. My example is very contrived, because when you have simple small classes, it's really easy to do the right thing and not tangle up your method calls. In reality, I've only experienced the accidental introduction of a circular dependency in much more complicated scenarios, where classes may have 5 to 10 dependencies each and those dependencies work through many layers. So, class A depends on class B, which depends on class C, which depends on class D, in which a developer decides they need to use a method in class A. Thus, you have your circle: A-B-C-D-A.

That said, I'll work through the simple, contrived example, because the same basic principles are at play - they're just much more obvious to see.

Imagine an ASP.NET Core Web API that returns a list of foods in a meal, plus the total calorific value.
So `GET api/meal` returns `{"foods":["Hot Chips","Battered Flake","Peas"],"calories":783}`

We put our code to build this data object into a service class, `MealMaker`:
```
    public interface IMealMaker
    {
        Meal GetMeal();
    }

    public class MealMaker : IMealMaker
    {
        public Meal GetMeal()
        {
            var foods = new List<string> { "Hot Chips", "Battered Flake", "Peas" };
            var calories = GetCalories(40, 4, 15);
            calories += GetCalories(20, 25, 24);
            calories += GetCalories(14, 5, 0);
            return new Meal(foods, calories);
        }

        private int GetCalories(int carbs, int protein, int fat)
        {
            return 4 * carbs + 4 * protein + 9 * fat;
        }
    }
```

`MealMaker` has to be registerd in `Startup.cs`:
```
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddControllersWithViews();
            services.AddScoped<IMealMaker, MealMaker>();
        }
```

It can then be used in an API controller:

```
    public class MealController : ControllerBase
    {
        private readonly IMealMaker _mealMaker;

        public MealController(IMealMaker mealMaker)
        {
            _mealMaker = mealMaker;
        }

        [HttpGet]
        public ActionResult Get()
        {
            return new JsonResult(_mealMaker.GetMeal());
        }
    }
```

All well and good up to this point. Later on, we need to extend the food data returned to include a dessert in addition to the fish and chips main course.
It seems like a good idea to put the code for building the dessert data into its own class, like so:
```
    public interface IDessertMaker
    {
        public Meal GetDessert();
    }

    public class DessertMaker : IDessertMaker
    {
        public Meal GetDessert()
        {
            var foods = new List<string> { "Ice Cream", "Wafer" };
            var calories = 300;
            return new Meal(foods, calories);
        }
    }
```

Now we need our meal, including dessert, to be returned to the API, so we update `MealMaker` to depend on `IDessertMaker` and combine the data:
```
    public class MealMaker : IMealMaker
    {
        private readonly IDessertMaker _dessertMaker;
        public MealMaker(IDessertMaker dessertMaker)
        {
            _dessertMaker = dessertMaker;
        }

        public Meal GetMeal()
        {
            var foods = new List<string> { "Hot Chips", "Battered Flake", "Peas" };
            var calories = GetCalories(40, 4, 15);
            calories += GetCalories(20, 25, 24);
            calories += GetCalories(14, 5, 0);

            var dessert = _dessertMaker.GetDessert();
            foods.AddRange(dessert.Foods);
            calories += dessert.Calories;

            return new Meal(foods, calories);
        }

        public int GetCalories(int carbs, int protein, int fat)
        {
            return 4 * carbs + 4 * protein + 9 * fat;
        }
    }
```

Of course, the ASP.NET IOC container needs to know about our new service class, so we add this line to `Startup.ConfigureServices()`:
```
services.AddScoped<IDessertMaker, DessertMaker>();
```

This works fine, we now get an API response of: `{"foods":["Hot Chips","Battered Flake","Peas","Ice Cream", "Wafer"],"calories":1083}`

Next, someone points out that our dessert calories is an estimate, and it ought to be calculated properly from the nutritional content.
We remember that there's a handy method in `MealMaker` that already does this!
So we take a dependency on that class, and `DessertMaker` becomes:
```
    public class DessertMaker : IDessertMaker
    {
        private readonly IMealMaker _mealMaker;
        public DessertMaker(IMealMaker mealMaker)
        {
            _mealMaker = mealMaker;
        }

        public Meal GetDessert()
        {
            var foods = new List<string> { "Ice Cream", "Wafer" };
            var calories = _mealMaker.GetCalories(50, 10, 30);
            calories += _mealMaker.GetCalories(8, 1, 1);
            return new Meal(foods, calories);
        }
    }
```

But when we run the code now we get a run-time exception:
`System.AggregateException: 'Some services are not able to be constructed'` and `A circular dependency was detected.`

With this trivial example, it's fairly easy to see where we went wrong: `MealMaker` depends on `DessertMaker` but `DessertMaker` also depends on `MealMaker`. The smallest possible circle of dependency!

At each step there were other decisions we could have made that would have avoided the circular dependency. We could have:
* put all the dessert making code into the existing `MealMaker` class
* made the controller depend on `MealMaker` and `DessertMaker` separately, and combined the two food lists in the action method

However, both of those decisions would have made existing classes bigger and more complex.

The best way out of this is not to combine classes, but to break things down even further. Move the calorie calculating method `GetCalories()` into its own class:
```
    public interface ICalorieCalculator
    {
        int GetCalories(int carbs, int protein, int fat);
    }
    public class CalorieCalculator : ICalorieCalculator
    {
        public int GetCalories(int carbs, int protein, int fat)
        {
            return 4 * carbs + 4 * protein + 9 * fat;
        }
    }
```

Now each of `MealMaker` and `DessertMaker` must depend upon `ICalorieCounter`.

Here's how this looks in `DessertMaker`. The `IMealMaker` dependency is gone and replaced by a simpler `ICalorieCounter` dependency:
```
    public class DessertMaker : IDessertMaker
    {
        private readonly ICalorieCalculator _calculator;
        public DessertMaker(ICalorieCalculator calculator)
        {
            _calculator = calculator;
        }

        public Meal GetDessert()
        {
            var foods = new List<string> { "Ice Cream", "Wafer" };
            var calories = _calculator.GetCalories(50, 10, 30);
            calories += _calculator.GetCalories(8, 1, 1);
            return new Meal(foods, calories);
        }
    }
```

`MealMaker` itself now has two dependencies, but the logic code to build the meal is still simple:
```
    public class MealMaker : IMealMaker
    {
        private readonly IDessertMaker _dessertMaker;
        private readonly ICalorieCalculator _calculator;
        public MealMaker(IDessertMaker dessertMaker, ICalorieCalculator calculator)
        {
            _dessertMaker = dessertMaker;
            _calculator = calculator;
        }

        public Meal GetMeal()
        {
            var foods = new List<string> { "Hot Chips", "Battered Flake", "Peas" };
            var calories = _calculator.GetCalories(40, 4, 15);
            calories += _calculator.GetCalories(20, 25, 24);
            calories += _calculator.GetCalories(14, 5, 0);

            var dessert = _dessertMaker.GetDessert();
            foods.AddRange(dessert.Foods);
            calories += dessert.Calories;

            return new Meal(foods, calories);
        }
    }
```

Don't forget to add `ICalorieCalculator` to `Startup.ConfigureServices()` and then run the API app  again.... Success!
Now we get our expected response: `{"foods":["Hot Chips","Battered Flake","Peas","Ice Cream","Wafer"],"calories":1338}`

Even in real life situations, where there are many more dependencies and layers of method calls involved, circular dependencies can always be resolved this way:
* Identify whatever method or methods you needed to use and caused you to add the extra dependency that became circular
* Then extract that method from the class it is on into a new class
* You may find that  methods need to come with it, or are similar and naturally belong on the new class
* You've created a smaller, more focussed class
* Update the dependencies of the two original classes that used the method to depend on the new class

With smaller classes, this technique will therefore always move you towards more adherence to the Single Responsibility Principle (SRP).
Conversely if you can keep SRP front of mind when you are writing code in the first place, you are much less likely to run into problems like circular dependencies at all!
