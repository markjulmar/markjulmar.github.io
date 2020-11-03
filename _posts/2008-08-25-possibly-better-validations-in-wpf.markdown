---
title: (Possibly) better validations in WPF
date: 2008-08-25T00:00:00.0000000
layout: post
categories: wpf
tags: wpf dotnet
---

I've never cared much for the built-in validation mechanisms provided by WPF.  I just don't think any of them feel natural to the way we build WPF applications today.  Basically, there are essentially three mechanisms built into WPF for validations:

1. Validation Rules
1. Exceptions
1. `IDataErrorInfo`

Validation rules are checked prior to transferring the value from the bound control to your source (typically the business object).  Exceptions are a form of validation rule - if the property setter throws an exception, it fails validation.  Finally, **IDataErrorInfo** was added with WPF 3.5 to support validations inside the business objects directly.  Often, you will use one or several of these validation techniques in your WPF application to check the input.

It's this last scenario (**IDataErrorInfo**) I want to focus on in this post.  I actually like having validations in the business object -I think it makes the most sense in most scenarios.  However, the interface itself feels clunky to me - it looks like this:

```csharp
interface IDataErrorInfo  
{  
    string Error { get; }  
    string this[string columnName];  
}
```

It's a weird one for sure.  I think the thing I dislike about it is the indexer - it forces your validation code to look like the old 90's style switch statements for Win32 message processing.  WPF calls the indexer just after applying the value to the bound property - the validation method is responsible for checking the value and returning a string indicating the failure, or null/empty if no failure occurred.

In building WPF applications, I've used the above interface many times to validate business objects and I found myself writing the same code over and over - most validations are pretty common:

1. Something must be there
1. It's a certain required length
1. It's a certain required pattern
1. It's a certain range (numeric)

And I thought to myself, "surely there's a better way!"  And so I came up with an attribute-based validation system.  It piggy-backs onto the **IDataErrorInfo** interface, but delegates off to a helper class which looks for attributes applied to the properties.  The attributes them provide the validation for you.  Here's the idea:

```csharp
public interface IValidator  
{  
    string Validate(string name, object value);  
}  
  
public abstract class ValidatorBase : Attribute, IValidator  
{  
    public abstract string Validate(string name, object value);  
}
```

The **IValidator** interface represents the validation to occur.  It gets passed the name of the property and the current value.  We wrap that in an abstract class called ValidatorBase for most attributes to derive from.  Here's a couple of examples:

```csharp
public class RequiredAttribute : ValidatorBase  
{  
    public override string Validate(string name, object value)  
    {  
        return value == null || value.ToString().Length == 0 ? name + " must be supplied." : null;  
    }  
}  
  
public class RequiredLengthAttribute : ValidatorBase  
{  
    public int Minimum { get; set; }  
    public int Maximum { get; set; }  

    public override string Validate(string name, object value)  
    {  
        if (value == null)  
            return null;  

        int len = value.ToString().Length;  
        return (len >= Minimum && len <= Maximum)  
                    ? null  
                    : string.Format("Length of {0} must be between {1} and {2}.", name, Minimum, Maximum);  
    }  
}
```

You could get as complicated as you like - I have in my toolbox a bunch more (Regex validator, range validator, etc.)  Next, we use a validation manager class to do the validations for us, this is what will implement the logic for **IDataErrorInfo**:

```csharp
public static class ValidationManager
{
    public static string Validate(string name, object instance)
    {
        if (instance == null)
            throw new ArgumentNullException("instance");

        Type type = instance.GetType();
        var errorList = new List<string>();

        if (!string.IsNullOrEmpty(name))
        {
            ValidateProperty(name, instance, errorList);
        }
        else
        {
            foreach (PropertyInfo pi in type.GetProperties(BindingFlags.Public | BindingFlags.Instance))
                ValidateProperty(pi.Name, instance, errorList);
        }

        return string.Join(Environment.NewLine, errorList.ToArray());
    }

    static void ValidateProperty(string name, object instance, ICollection<string> errorList)
    {
        Type type = instance.GetType();
        PropertyInfo pi = type.GetProperty(name, BindingFlags.Public | BindingFlags.Instance);
        if (pi != null)
        {
            object value = pi.GetValue(instance, null);
            foreach (Attribute att in pi.GetCustomAttributes(true))
            {
                var iv = att as IValidator;
                if (iv != null)
                {
                    string err = iv.Validate(name, value);
                    if (!string.IsNullOrEmpty(err))
                        errorList.Add(err);
                }
            }
        }
    }
}
```  

It's pretty simple - it just uses reflection to walk through the properties and look for the **IValidator** interface on any attributes.  If it finds any, it executes the Validate logic and returns a string indicating the failure(s).

To use the above code, you add the attributes to your business object properties.  For example, here is a simple Person object which uses the two validators for it's name property:

```csharp
public class Person : INotifyPropertyChanged, IDataErrorInfo
{
    private string _name;

    [Required]
    [RequiredLength(Minimum = 2, Maximum = 10)]
    public string Name
    {
        get { return _name; }
        set
        {
            _name = value;
            OnPropertyChanged("Name");
        }
    }
    ...
}
```

Finally, you implement the IDataErrorInfo by utilizing the ValidationManager class.  If you have a base class for your business objects, it could be done directly in the base class and then propagate throughout your hierarchy:

```csharp
#region IDataErrorInfo Members  
string IDataErrorInfo.Error  
{  
   get { return ValidationManager.Validate(null, this); }  
}  
  
string IDataErrorInfo.this[string columnName]  
{  
   get { return ValidationManager.Validate(columnName, this); }  
}  
#endregion  
```  

That's it!  Now, you have an extensible validation mechanism which is reusable and easy to apply.  Plus, it makes it much more obvious what the validation rules are for the business object - and code clarity is important in large projects with multiple devs.  Feel free to use the above code in your own projects, if you create any really cool validators, please share!
