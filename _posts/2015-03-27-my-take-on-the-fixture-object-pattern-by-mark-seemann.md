---
title: My take on the Fixture Object pattern by Mark Seemann
date: 2015-03-27 11:09:51.000000000 +00:00
categories: []
tags: []
---
A fair while back, whilst discussing and experimenting with unit testing, I had read about this particular pattern. The Fixture Object pattern allows you to delegate complex `arrange` logic within your tests in order to improve readability and maintainability of your code.

First of all, now that I have finally found the link (I have been talking about it, but I could never find the link), let me link it here: <http://blog.ploeh.dk/2009/03/16/FixtureObject/>

In the post above, Mark shows how a fixture can be used to improve your tests readability by hiding the composition and leaving a nice to read method. Another thing that he has shown, is how we can encapsulate the construction of the subject under test (SUT). The construction of the subject under test is not what we are testing and it is also something that we do for all of our tests. By encapsulating it behind a factory method, if the constructor were to change, we would only need to fix the construction in one place per test file.

To help understand the pattern myself, I like to think of it as a _sort of stateful_ SUT factory. The key word here is stateful; the dependencies of your SUT will most likely have to be mocks or stubs that you will bend to your will in order to influence the outcome. The fixture can create and alter these dependencies on your behalf (but on command) in order to keep the arrange section of your tests lean and readable. Another thing to note, is that the fixture could help you set up complex method parameters also; this is why I made an emphasis on _sort of_.

Before we continue, a quick disclaimer on the tools I will be using below; I use NUnit as my testing framework and Moq for mocking. Let's look at an example. Imagine a test as follows:

```csharp
[TestFixture]
public class CounterTests
{
    public void CalculateTotal_BasketContainsValidCouponFor10PercentOff_TotalIncludesA10PercentDiscount()
    {
        const string couponCode = "OXAD";
        var couponRepository = new Mock<ICouponRepository>();
        couponRepository.Setup(repo => repo.GetCoupon(couponCode))
            .Returns(new Coupon()
            {
                Active = true,
                Code = couponCode,
                PercentOff = 0.1m
            });
        
        var sut = new TotalCalculator(couponRepository.Object);
        var basket = new Basket()
        {
            Products = new List<Product>()
            {
                new Product() { Price = 50, Id = 1 },
                new Product() { Price = 50, Id = 2 },
                new Product() { Price = 30, Id = 3 }
            },
            CouponCode = couponCode
        };
        var total = sut.CalculateTotal(basket);
        Assert.AreEqual(117.0m, total);
    }
}
```
So, to summarise:
1. We create a mock coupon code.
2. Then we set up our coupon repository to create a valid coupon when asked about our mock coupon code.
3. We create our basket and ensure that it has products that we can discount.
4. We make sure our basket has our mock coupon code.
5. Finally, we run our test and verify the output.

Out of these steps, we only really care about certain aspects:
- We want our basket to contain a coupon code.
- We want our repository to allow a 10% discount based on this code.
- We need products, but we don't care how many or in what proportions.

The last point is quite important. Adding two products of £50.00, and one of £30.00 allows us to know we have £130.00 worth of products in our basket, and this allows us to assume that a 10% will give us a total of £117.00. So we need to specify the value of our basket within our test in order to maintain readability, but we don't need to keep the details of how we compose the basket.

Another point that needs to be raised, is that we are constructing our SUT within our tests and therefore, making our tests more fragile than it needs to be. Finally, we set up our repository using a very generic API which can sometimes become large and complicated looking; the important information about what specifically we are setting up is hidden inside a fair amount of setup and construction logic.

Meet the fruition of the fixture object (FYI, this example tests for a 20% discount instead of 10%):
```csharp
[TestFixture]
public class CounterTests
{
    public void CalculateTotal_BasketContainsValidCouponFor20PercentOff_TotalIncludesA20PercentDiscount()
    {
        const string couponCode = "OXAD";
        var basket = Fixture.CreateBasketWithCoupon(productSum: 130m, coupon: couponCode);
        Fixture.AddValidCouponToRepo(code: couponCode, discount: 0.2m);
        
        var sut = Fixture.TestSubject;
        var total = sut.CalculateTotal(basket);
        
        Assert.AreEqual(104.0m, total);
    }
}
```
There is certainly less code, but did it achieve what it set out to?
1. The first thing we do is to create a basket. The basket should have a product sum of £130.00 and should include a coupon code applied with a value of 'OXAD'.
2. We then add a valid coupon to our repository with the same coupon code as used before, and with a discount of 0.2 (20%).
3. Finally, we get our SUT before running our test and asserting that our discount was applied.

Whether it reads better or is easier to maintain will be up for interpretation of the reader, but for me, this reads much better. The next thing we should look at is the Fixture object; but just before, I would like to show you how I wire up my fixture.
```csharp
[TestFixture]
public class CounterTests
{
    internal class TestFixture
    {
        //...
    }

    private TestFixture Fixture { get; set; }
    [SetUp]
    public void Setup()
    {
        Fixture = new TestFixture();
    }
}
```

So the first thing you should notice, is that I create a nested class with name TestFixture. This allows me to re-implement the pattern in multiple tests without worrying about conflicting class names. It is also appropriate, as each TestFixture is unique to its parent test class. I create a Fixture property for my tests to use, and I ensure that just before each test, the Fixture object is recreated so that any state within this class will be irrelevant for the next test helping to achieve test isolation.

The fixture class can get quite large because of the many different methods and properties. One technique could be to split your class into two using the partial keyword and define the nested class in a separate file.
```csharp
internal class TestFixture
{
    private TotalCalculator _sut;
    private Mock<ICouponRepository> _couponRepoMock;
    public TotalCalculator TestSubject
    {
        get { return _sut ?? CreateSut(); }
    }

    public ICouponRepository CouponRepo
    {
        get { return _couponRepo ?? CouponRepoMock }
        set { _couponRepo = value; }
    }

    private Mock<ICouponRepository> CouponRepoMock
    {
        get { return _couponRepoMock ?? CreateCouponRepoMock(); }
    }
    
    private TotalCalculator CreateSut()
    {
        return _sut = new TotalCalculator(CouponRepo);
    }

    private Mock<ICouponRepository> CreateCouponRepoMock()
    {
        _couponRepoMock = new Mock<ICouponRepository>();
        return _couponRepoMock;
    }

    public Basket CreateBasketWithCoupon(decimal productTotal, string coupon)
    {
        return new Basket()
        {
            Products = CreateManyProducts(productTotal),
            CouponCode = coupon
        };
    }
    
    private List<Product> CreateManyProducts(decimal productTotal)
    {
        const int many = 3;
        var third = Math.Floor(productTotal/many * 100) / 100;
        var remainder = productTotal - (third*3);
        return new List<Product>()
        {
            new Product() {Id = 1, Price = third},
            new Product() {Id = 1, Price = third},
            new Product() {Id = 1, Price = third + remainder},
        };
    }

    public void AddValidCouponToRepo(string code, decimal discount)
    {
        CouponRepoMock.Setup(repo => repo.GetCoupon(code))
            .Returns(new Coupon()
            {
                Active = true,
                Code = code,
                PercentOff = discount
            });
    }
}
```
Above is the TestFixture. As you can see, the public method names are written to be easy to read and the class will keep a reference to each of the dependencies that it needs so that any modification made on it will not be lost. There are a few refactoring steps missing here, I went through a bunch of variations of the fixture pattern before settling, and even now, I am adjusting it as I need to.

One thing you may notice is the lazy loading happening almost everywhere. The idea here, is that we have a default construction method that creates the objects that the SUT needs in a barebones way.

Imagine that when we set up the coupon repository, we used a custom Fake class instead of using Moq. In our test, instead of calling AddValidCouponToRepo(), we would set the CouponRepo property on our fixture before retrieving our SUT. When we retrieve our SUT, the following chain is followed:

1. The TestSubject's 'get' method is invoked.
2. The method, in turn, invokes the 'get' method of the CouponRepo property.
3. The CouponRepo 'get' method will check to see if the private field is null.
4. As the field is not null, it returns the existing fake object that we created before.
5. The CouponRepository fake is passed into the TestSubject's constructor
6. The TestSubject is returned.

On the flip side, if we do not do anything special to our couponRepo, then we just want a basic stub object that makes sure it behaves well enough to allow your code to run its course (as stubs do). If we do not set the CouponRepo property, and we don't set up any behaviours, then when the CouponRepo 'get' method checks the private field, the field will be null, then it will check to see if a mock has been created, if not, it will create one and finally return the mock object which will act as our stub.

One of the pitfalls that I think I have already come across, is that the fixture object can hide ugly looking, and complicated setups. Sometimes it is better to use a custom fake instead of writing complex setup logic at all. Another thing to keep in mind, is that whenever you find yourself repeating yourself in your fixtures, consider extracting it out into another class that can be shared between your tests; DRY still applies to a certain extent in your unit tests, the trick is to find the right balance between code de-duplication and readability.

In some cases, it is better to write two different methods that create or set up an object using similar logic. Technically, the methods are doing something very similar, but in terms of semantics, it will be drastically different. For example, `CreateALowIncomeBankAccount()` and `CreateAHighIncomeBankAccount()` can be more descriptive than `CreateABankAccount(20.00m)` and `CreateABankAccount(100000.00m)`. If you don't care about the amount in your tests any more than "it should be a high amount or a low amount", you should consider if you can make your method easier to read **without** parameterising it. You can always create a private parameterised method to adhere to DRY in the background.

Finally, use the YAGNI principle here as much as you can; it can be very tempting to make our methods more and more generic, but the more we do so, the less descriptive our method names can be. Don't add parameters to your methods if none of your tests use them yet, don't try to cater for scenarios in your codebase that haven't yet occurred (for example, the lazy loading is not actually useful in this simple scenario, but I kept it in for demonstration purposes). In general, try not to think about your next test until you get there; its easier to refactor a method to cater for a new test once you have usages in your code, than to attempt to design a method that will work for all tests before you have written your tests.

Here is an example of the fixture object without lazy loading:
```csharp
internal class TestFixture
{
    private Mock<ICouponRepository> _couponRepoMock;
    public TestFixture()
    {
        CouponRepoMock = new Mock<ICouponRepository>();
        TestSubject = new TotalCalculator(_couponRepoMock.Object);
    }

    public TotalCalculator TestSubject { get; private set; }

    private Mock<ICouponRepository> CouponRepoMock { get; set; }

    public Basket CreateBasketWithCoupon(decimal productTotal, string coupon)
    {
        return new Basket()
        {
            Products = CreateManyProducts(productTotal),
            CouponCode = coupon
        };
    }

    private List<Product> CreateManyProducts(decimal productTotal)
    {
        const int many = 3;
        var third = Math.Floor(productTotal/many * 100) / 100;
        var remainder = productTotal - (third*3);

        return new List<Product>()
        {
            new Product() {Id = 1, Price = third},
            new Product() {Id = 1, Price = third},
            new Product() {Id = 1, Price = third + remainder},
        };
    }

    public void AddValidCouponToRepo(string code, decimal discount)
    {
        CouponRepoMock.Setup(repo => repo.GetCoupon(code))
            .Returns(new Coupon()
            {
                Active = true,
                Code = code,
                PercentOff = discount
            });
    }
}
```
This fixture doesn't try to over think any what-if scenarios, the only part of this that makes me worry a little, is that the SUT is created right at the beginning and using the CouponRepository before any setups have been defined on the mock. Moq does not require that mocks were set up before exporting the object, but it does require that the object is not used before a setup has been made. Even so, I think that exporting the object before setting up the mock can cause confusion and so in my final implementation, I would leave only the SUT lazy loaded.
