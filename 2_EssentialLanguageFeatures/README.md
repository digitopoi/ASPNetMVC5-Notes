# Essential Language Features

## Using Automatically Implemented Properties

The regular C# property feature lets you expose a piece of data from a class in a way that decouples that data from how it is set and retrieved

REGULAR PROPERTY (VERBOSE):
```c#
public class Product
{
    private string _name;

    public string Name
    {
        get { return name; }
        set { name = value; }
    }
}
```

Using properties is preferred to using fields because you can change the statements in the get and set blocks without needing to change the classes that depend on the property.

AUTO-IMPLEMENTED PROPERTIES:
```c#
public class Product
{
    public int ProductID { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public decimal Price { get; set; }
    public string Category { get; set; }
}
```

Bodies of the getter and setter and the fields aren't defined

C# compiler implements the getter and setter and fields behind the scenes

Using automatic properties saves typing, creates code that is easier to read, and preserves the flexibility that a property provides.

Can change the implementation of a property:

```c#
public class Product 
{
    private string _name;

    public int ProductID { get; set; }

    public string Name
    {
        get
        {
            return ProductID + name;
        }
        set 
        {
            name = value;
        }
    }

    public string Description { get; set; }
    public decimal Price { get; set; }
    public string Category { get; set; }
}
```

## Using Object and Collection Initializers

Another tiresome programming task is constructing a new object and then assigning values to the properties:

```c#
public ViewResult CreateProduct()
{
    //  Create a new Product object
    Product myProduct = new Product();

    //  Set the property values
    myProduct.ProductID = 100;
    myProduct.Name = "Kayak";
    myProduct.Description = "A boat for one person";
    myProduct.Price = 275M;
    myProduct.Category = "Watersports";

    return View("Result", 
        (object)String.Format("Category: {0}", myProduct.Category));
}
```

Using **object initializer** feature:

```c#
public ViewResult CreateProduct() 
{
    //  Create and populate a new Product object
    Product myProduct = new Product
    {
        ProductID = 100, Name = "Kayak",
        Description = "A boat for one person",
        Price = 275M, Category = "Watersports"
    };

    return View("Result",
        (object)String.Format("Category: {0}", myProduct.Category));
}
```

The braces form the initializer - in which you can supply values to the parameters as part of the construction process

## Using Extension Methods

Extention methods allow you to add methods to classes you do not own and cannot modify directly. 

```c#
public class ShoppingCart
{
    public List<Product> Products { get; set; }
}
```

Perhaps the above class came from a third party, and I cannot modify it. 

I want to determine the total value of the Product object in the ShoppingCart class.

Use extension method to add the functionality:

```c#
namespace LanguageFeatures.Models
{
    public static decimal TotalPrices(this ShoppingCart cartParam)
    {
        decimal total = 0;

        foreach(Product prod in cartParam.Products)
        {
            total += prod.Price;
        }

        return total;
    }
}
```

The *this* keyword in front of the first parameter marks TotalPrices as an extension method.

The first parameter tells .NET which class the extension method can be applied to (ShoppingCart in this case)

Apply the extension:

```c#
public ViewResult UseExtension()
{
    //  Create and populate a new Product object
    ShoppingCart cart = new ShoppingCart 
    {
        Products = new List<Product>
        {
            new Product { Name = "Kayak", Price = 275M },
            new Product { Name = "Lifejacket", Price = 48.95M },
            new Product { Name = "Soccer ball", Price = 19.50M },
            new Product { Name = "Corner flag", Price = 34.95M },
        }
    };

    //  Get the total value of the products in the cart
    decimal cartTotal = cart.TotalPrices();     // <====

    return View("Result",
        (object)String.Format("Total: {0:c}", cartTotal));
    
}
```

