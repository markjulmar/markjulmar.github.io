---
title: Dynamic type binding in WPF 4.5
date: 2012-03-10T00:00:00.0000000
layout: post
categories: wpf
tags: wpf dotnet
---

One of the new features in WPF 4.5 is data-binding support for [ICustomTypeProvider](http://msdn.microsoft.com/en-us/library/system.reflection.icustomtypeprovider(v=vs.96).aspx "ICustomTypeProvider"). This enables adding dynamic properties to types where the actual shape of the type is not known until runtime. For example, where the data itself is being retrieved from a web service (JSON) or from an XML definition. This feature was also added in Silverlight 5 so we have feature parity here (almost).

ICustomTypeProvider allows you to define the bindable properties at runtime with full type knowledge (something that the .NET 4.0 dynamic/ExpandoObject/DynamicObject support are not capable of providing). That means XAML type coercion continues to work properly which is very nice. However, the support isn't as straight forward as you might think - it requires some plumbing to make it work. I looked around and found that [Alexandra Rusina had blogged about doing this for Silverlight](http://blogs.msdn.com/b/silverlight_sdk/archive/2011/04/25/binding-to-dynamic-properties-with-icustomtypeprovider-silverlight-5-beta.aspx) but nobody had done any samples for WPF. So, I took Alexandra's sample and modified it to work for WPF 4.5 - which involved extending the custom type provider to support all the method calls available in .NET (vs. just the Silverlight subset in the original sample).

When you run the sample, you get a dialog which looks like:

![](/images/CustomTypeProviderWpf45.png "CustomTypeProviderWpf45")

You get the data by clicking the button - I provided two DataGrid sets to show that INotifyPropertyChanged works for both the built-in properties as well as the dynamic properties added to each of the data types.

The key bit of reusable code in the sample is the `CustomTypeHelper` which provides the implementation for the `ICustomTypeProvider` as well as a derived `Type` and `PropertyInfo` class to hold type and custom property data. It has static methods to add new property definitions to a type:

```csharp
public partial class CustomTypeHelper<T> : ICustomTypeProvider, INotifyPropertyChanged
{
    // Methods used to add new properties
    public static void AddProperty(string name);
    public static void AddProperty(string name, Type propertyType);
    public static void AddProperty(string name, Type propertyType, List<Attribute> attributes);

    // Instance methods to get/set values on a specific instance
    public void SetPropertyValue(string propertyName, object value);
    public object GetPropertyValue(string propertyName);
    public PropertyInfo[] GetProperties();

    // Implementation of ICustomTypeProvider
    public Type GetCustomType();

    // Property change notification
    public event PropertyChangedEventHandler PropertyChanged;
    protected void RaisePropertyChanged([CallerMemberName] string propertyName = "");
}
```

This class was almost unmodified from Alexandra's sample. The sample usage shows two techniques for adding properties (both were in the original sample as well), the first is to extend the `CustomTypeHelper`:

```csharp
public class Product : CustomTypeHelper<Product>
{
    private string _name;
    private int _price;

    // Existing known properties.
    public string Name
    {
        get { return _name; }
        set { _name = value; RaisePropertyChanged(); }
    }

    public int Price
    {
        get { return _price; }
        set { _price = value; RaisePropertyChanged(); }
    }
}
```

This is, by far, the easiest approach - you can then add new properties dynamically through the methods exposed on the `CustomTypeHelper` - the new properties must be added prior to creating the object type for this implementation as it does not re-scan the known property list on each set, but instead caches it off up front when the object is instantiated. This could be changed, but it's more performant this way. In the `MainWindow.OnGetData` method, we add the custom property to the `Product` type and then apply a value to the created instance:

```csharp
Product.AddProperty("Available", typeof(bool));

Product product1 = new Product { Name = "Soap", Price = 2 };
product1.SetPropertyValue("Available", true);
```

The second approach is to use encapsulation - where you hold an inner `CustomTypeHelper` type and delegate to it for the implementation. This is done for the `Customer` type:

```csharp
public class Customer: ICustomTypeProvider, INotifyPropertyChanged
{
  /// <summary>
  /// Custom Type provider implementation - delegated through static methods.
  /// </summary>
  private readonly CustomTypeHelper <Customer> _helper = new CustomTypeHelper <Customer> ();

  // Existing known properties
  private string _firstName;
  private string _lastName;

  public string FirstName {
    get {
      return _firstName;
    }
    set {
      _firstName = value;
      RaisePropertyChanged();
    }
  }

  public string LastName {
    get {
      return _lastName;
    }
    set {
      _lastName = value;
      RaisePropertyChanged();
    }
  }

  public Customer() {
    _helper.PropertyChanged += (s, e) =>PropertyChanged(this, e);
  }

  // Methods to support dynamic properties. public static void AddProperty(string name) { CustomTypeHelper<Customer>.AddProperty(name); }
  public static void AddProperty(string name, Type propertyType) {
    CustomTypeHelper < Customer > .AddProperty(name, propertyType);
  }

  public static void AddProperty(string name, Type propertyType, List < Attribute > attributes) {
    CustomTypeHelper < Customer > .AddProperty(name, propertyType, attributes);
  }

  public void SetPropertyValue(string propertyName, object value) {
    _helper.SetPropertyValue(propertyName, value);
  }

  public object GetPropertyValue(string propertyName) {
    return _helper.GetPropertyValue(propertyName);
  }

  public PropertyInfo[] GetProperties() {
    return _helper.GetProperties();
  }

  Type ICustomTypeProvider.GetCustomType() {
    return _helper.GetCustomType();
  }

  public event PropertyChangedEventHandler PropertyChanged = delegate {};
  private void RaisePropertyChanged([CallerMemberName] string propertyName = "") {
    PropertyChanged(this, new PropertyChangedEventArgs(propertyName));
  }
}
```

As you can see, this is more code, but it has the advantage of allowing us to use a different base class (a ViewModel base for example). Notice we have to forward property change notifications with this approach - that's the purpose of hooking into it as part of the constructor. Adding dynamic properties is much the same as before - we use the static methods prior to constructing the object:

```csharp
Customer.AddProperty("Age", typeof(int));
Customer.AddProperty("Married", typeof(bool));

Customer customer1 = new Customer { FirstName = "Julie", LastName = "Smith" };
customer1.SetPropertyValue("Age", 38);
customer1.SetPropertyValue("Married", true);

Customer customer2 = new Customer { FirstName = "Mark", LastName = "Smith" };
customer2.SetPropertyValue("Age", 43);
customer2.SetPropertyValue("Married", true);
```

The modified WPF sample code is available [here](/samples/CustomTypeProviderSample.zip) and requires the VS11 beta and .NET 4.5. Please keep in mind this is a derived work - all the real credit goes to Alexandra Rusina who built the Silverlight version of this code. See that post for more details on how all this works.
