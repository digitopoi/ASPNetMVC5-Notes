# Essential Tools for MVC

This chapter looks at three MVC tools:

  1. Dependency Injection Container (Ninject)

  2. A unit test framework (built-in)

  3. A mocking tool (Moq)

## Using Ninject

The idea behind **dependency injection** is to decouple the components in an MVC application with a combination of interfaces and DI container that creates instances of objects by creating implementations of the interfaces they depend on and injecting them into the constructor. 

Currently, the ShoppingCart class is tightly coupled to the LinqValueCalculator and the HomeController is tightly coupled to both ShoppingCart and LinqValueCalculator:

ShoppingCart.cs
```c#
public class ShoppingCart
{
    private LinqValueCalculator calc;

    public ShoppingCart(LinqValueCalculator calcParam)      //    <====
    {
        calc = calcParam;
    }

    public IEnumerable<Product> Products { get; set; }

    public decimal CalculateProductTotal()
    {
        return calc.ValueProducts(Products);
    }
}
```

HomeController.cs
```c#
public class HomeController : Controller
{
    private Product[] products =
    {
        new Product { Name = "Kayak", Category = "Watersports", Price = 275M },
        new Product { Name = "Lifejacket", Category = "Watersports", Price = 48.95M },
        new Product { Name = "Soccer ball", Category = "Soccer", Price = 19.50M },
        new Product { Name = "Corner flag", Category = "Soccer", Price = 34.95M },
    };

    // GET: Home
    public ActionResult Index()
    {
        LinqValueCalculator calc = new LinqValueCalculator();                 //  <====

        ShoppingCart cart = new ShoppingCart(calc) { Products = products };   //  <====

        decimal totalValue = cart.CalculateProductTotal();

        return View(totalValue);
    }
}
```

If I want to replace LinqValueCalculator, I have to locate and then change the references in the class that are tightly coupled to it.

Not a problem in such a small application, but in a larger one it would cause problems (tedious and error-prone).

## Applying an Interface

Part of the problem can be solved by using a C# interface to abstract the definition of the calculator functionality from its implementation.



