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

I can solve this by implementing a C# interface to abstract the definition of the calculator functionality from its implementation:

IValueCalculator.cs
```c#
public interface IValueCalculator
{
    decimal ValueProducts(IEnumerable<Product> products);
}
```

This interface can then be implemented in the LinqValueCalculator class:

LinqValueCalculator.cs
```c#
public class LinqValueCalculator : IValueCalculator
{
  ...
}
```

The interface allows me to break the tight coupling between ShoppingCart and LinqValueCalculator

ShoppingCart.cs
```c#
public class ShoppingCart 
{
  private IValueCalculator calc;

  public ShoppingCart(IValueCalculator calcParam)
  {
    calc = calcParam;
  }

  ...
}
```

C# still requires me to specify the implementation class for an interface during instantiation

I still have a problem in the HomeController:

HomeController.cs
```c#
public ActionResult Index()
{
  IValueCalculator calc = new LinqValueCalculator();

  ...

}
```

I now need to use Ninject to reach the point where I can specify that I want to instantiate an implementation of the IValueCalculator interface, but the details of which implementation is required are not part of the code in the HomeController.

Need to tell Ninject that LinqValueCalculator is the implementation of IValueCalculator interface and update HomeController class so that it obtains its objects via Ninject, rather than by using the *new* keyword.

## Adding Ninject to the Visual Studio Project

```bash
Install-Package Ninject -version 3.0.1.10
Install-Package Ninject.Web.Common -version 3.0.0.7
Install-Package Ninject.MVC3 -Version 3.0.0.6
```

### The first step is to prepare Ninject for use. 

1. Create an instance of Ninject **kernel**- the object that is responsible for resolving dependencies and creating new objects.

Ninject can be extended and customized to use different kinds of kernel, but usually (almost always) you only need the StandardKernel.

```c#
IKernel ninjectKernel = new StandardKernel();
```

2. Configure the Ninject kernel so that it understands which implementation objects I want to use for each interface I am working with

Ninject uses C# type parameters to create a relationship: I set the interface I want to work with as the type parameter for the Bind() method and call the To() method on the result it returns. I set the implementation class I want instantiated as the type parameter for the To() method. This tells Ninject that dependencies on the IValueCalculator interface should be resolved by creating an instance of the LinqValueCalculator class.

```c#
ninjectKernel.Bind<IValueCalculator>().To<LinqValueCalculator>();
```

3. Use Ninject to create an object - through the kernel Get() method

The type parameter for the Get() method tells Ninject which interface I am interested in and the result from this method is an instance of the implementation type I specified with the To() method

```c#
IValueCalculator calc = ninjectKernel.Get<IValueCalculator>();
```

HomeController.cs
```c#
public ActionResult Index()
{
    IKernel ninjectKernel = new StandardKernel();                           //  [1]
    ninjectKernel.Bind<IValueCalculator>().To<LinqValueCalculator>();       //  [2]

    IValueCalculator calc = ninjectKernel.Get<IValueCalculator>();          //  [3]

    ShoppingCart cart = new ShoppingCart(calc) { Products = products };

    decimal totalValue = cart.CalculateProductTotal();

    return View(totalValue);
}
```

## Setting up MVC Dependency Injection

So far, the Ninject linkage remains in HomeController - so it is still tightly coupled to the LinqValueCalculator class.

Next step is to embed Ninject at the heart of the MVC application - will allow to simplify the controller, expand Ninject's influence so it works across the app, and move the configuration out of the controller.

## Creating the Dependency Resolver

**custom dependency resolver**- The MVC Framework uses a dependency resolver to create instances of the classes it needs to service requests. By creating a custom resolver, I can ensure that the MVC Framework uses Ninject whenever it creates an object-including instance of controllers, for example.

To set up the resolver, I need a new folder called Infrastructure - folder to put classes that do not fit the other folders in an MVC application. 

And, add a class called NinjectDependencyResolver:

```c#
public class NinjectDependencyResolver : IDependencyResolver
{
    private IKernel kernel;

    public NinjectDependencyResolver(IKernel kernelParam)
    {
        kernel = kernelParam;
        AddBindings();
    }

    public object GetService(Type serviceType)
    {
        return kernel.TryGet(serviceType);
    }

    public IEnumerable<object> GetServices(Type serviceType)
    {
        return kernel.GetAll(serviceType);
    }

    private void AddBindings()
    {
        kernel.Bind<IValueCalculator>().To<LinqValueCalculator>();
    }
}
```

The NinjectDependencyResolver class implements the IDependencyResolver interface (from System.Mvc namespace) - MVC Framework uses it to get the objects it needs.

The MVC Framework will call the GetService() or GetServices() methods when it needs an instance of a class to service an incoming request.

The job of a dependency resolver is to create that instance - performed by Ninject TryGet() and GetAll() methods. 

The TryGet() method works like the Get() method, but returns null when there is no suitable binding rather than throwing an exception.

The GetAll() method supports multiple bindings for a single type, which is used when there are several different implementation objects available.

The dependency resolver class is also where Ninject binding is set up. In the AddBindings() method, the Bind() and To() methods are used to configure the relationship between the IValueCalculator interface and the LinqValueCalculator class.

## Register the Dependency Resolver

It's not enough to simply create an implementation of the IDependencyResolver interface - also need to tell MVC Framework that I want to use it.

Ninject created a file called NinjectWebCommon.cs in the App_Start folder - defines methods called automatically when the application starts, in order to integrate into the ASP.NET request lifecycle. (This is to provide the **scopes** feature). 

In the RegisterServices() method, I add a statement that creates an instance of the NinjectDependencyResolver class to register the resolver with the MVC Framework.

NinjectWebCommon.cs
```c#
private static void RegisterServices(IKernel kernel)
{
  System.Web.Mvc.DependencyResolver.SetResolver(new
    EssentialTools.Infrastructure.NinjectDependencyResolver(kernel));
}
```

## Refactoring the Home Controller

The final step is to refactor the HomeController so that it takes advantage of the facilities set up previously.

The main change is to add a class constructor that accepts and implementation of the IValueCalculator interface. Ninject will provide an object that implements the interface when it creates an instance of the controller, using the configuration setup in the NinjectDependencyResolver class.



```c#
public class HomeController : Controller
{
    private IValueCalculator calc;                              //  <====

    private Product[] products =
    {
        new Product { Name = "Kayak", Category = "Watersports", Price = 275M },
        new Product { Name = "Lifejacket", Category = "Watersports", Price = 48.95M },
        new Product { Name = "Soccer ball", Category = "Soccer", Price = 19.50M },
        new Product { Name = "Corner flag", Category = "Soccer", Price = 34.95M },
    };

    public HomeController(IValueCalculator calcParam)           //  <====              
    {
        calc = calcParam;
    }

    // GET: Home
    public ActionResult Index()
    {
        ShoppingCart cart = new ShoppingCart(calc) { Products = products };

        decimal totalValue = cart.CalculateProductTotal();

        return View(totalValue);
    }
}
```



