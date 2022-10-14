---
layout: post
title:  "Hacking Fluent Validation: Configuring via Attributes"
date:   2019-08-09 14:39:34 +0700
---
 
About a year ago I was brought on a new project to help build out the API's. As I usually do with my API projects I added [FluentValidation](https://github.com/JeremySkinner/FluentValidation). It seemed like everything was going smooth until a couple developers shared they weren't a fan of it. They didn't like creating separate classes and wanted to use attributes instead, sigh fine. I was never a fan of attributes because I feel it makes everything too cluttered. This post is about the journey of destroying everything that's great about `FluentValidation` by using attributes.

I wasn't going to be able to use the built in 
[DataAttributes](https://docs.microsoft.com/en-us/dotnet/api/system.componentmodel.dataannotations?view=netframework-4.8)
because the classes that needed them lived in our contracts project. We wanted that dll to be dependency free so we could share it easily. Since I didn't want to get in the business of creating a new validation framework I set out to create a set of custom attributes that could be leveraged by FluentValidation. Unknowingly, I did exactly as described in this [issue](https://github.com/JeremySkinner/FluentValidation/issues/393). There will be a fair amount of code in this post, you can view the the final result in this [gist](https://gist.github.com/JoeGannon/91e76605a2a7a7944a1a6ccde3abb3c3).

## Fluent Validation

If you aren't familiar with `FluentValidation`, it's a validation framework. If you hate attributes as much as I do 
it's a great alternative. Validation is built from validators that you can add to the `asp.net` pipeline. A simple example is

```c#
public class PersonValidator : AbstractValidator<Person>
{
    public PersonValidator()
    {
        RuleFor(x => x.Name).NotEmpty();
    }
}
```

## Warning

Before doing something like this, it's important to note that it's potentially very dangerous. You can get into trouble by leveraging the _internals_ of the library and using it in ways the authors never intended. You can say goodbye to [semantic versioning](https://semver.org/) since that only covers the public API. Which means even a bump in a minor or patch version (which are backwards compatible by definition) could break your custom code. If you can't incorporate the new changes for whatever reason you won't be able to upgrade. You may also burn hours with other oddities that appear more silently. This all has to be taken into account before you go 1337 hacker. With that in mind, considering the internal surface area that I'll be touching is relatively small, along with the fact FV is a mature library with less risk of major breaking changes, I was confident going forward with this (famous last words).

## The plan

The ultimate goal is to configure FluentValidation with attributes and not use the `AbstractValidator<T>` classes. 

* don't reinvent the wheel, leverage fluent validation (version 8.1.3), look at internal pieces if needed
* map our custom attributes to FV parlance (RequiredAttribute => NotEmpty)
* inject instances of `IValidator<T>` to the IoC container, FV by [design](https://fluentvalidation.net/aspnet#asp-net-core)   requires these implementations
* build these instances dynamically, the types will be unknown at compile time


Using the example from earlier, my first thought was I could call the `RuleFor(x => x.Name).NotEmpty()` methods using reflection. At the time it sounded like a good idea and the most simple way, though I knew it would also be the ugliest (spoiler: awful idea). Well, I was having trouble making it work even with hardcoded types, had doubts I'd even be able to make it work without the hardcodes, the code was super ugly, and not to mention I'm calling 3rd party library code through reflection...not a good idea. While I was spinning my tires on this piece, I realized I didn't have to actually call these methods at all! I only have to mimic their behavior and I can do that by whatever means. 

## Opening up FluentValidation
I opened up fluent validation and started debugging through a simple test from [this](https://github.com/JeremySkinner/FluentValidation/blob/64b78d6bdc9595d221b4d56ce70a00e6de08aa4e/src/FluentValidation.Tests/NotEmptyTester.cs) class. 
I tried not to fall into the common tendency of trying to learn EVERYTHING, that's just impossible in any reasonable amount of time and unnecessary. After stepping through the code for a bit I found FV to be a great codebase, everything was very intuitive and it was easy to see what was going on.

Eventually I found a [key piece](https://github.com/JeremySkinner/FluentValidation/blob/64b78d6bdc9595d221b4d56ce70a00e6de08aa4e/src/FluentValidation/Internal/RuleBuilder.cs#L53-L57) in `Rule.AddValidator(validator)`. I saw I just need to add a validator to a rule. Next I had to find out what a [PropertyRule](https://github.com/JeremySkinner/FluentValidation/blob/64b78d6bdc9595d221b4d56ce70a00e6de08aa4e/src/FluentValidation/Internal/PropertyRule.cs) is and how they are used. 

I learned that the PropertyRule is the [essential](https://github.com/JeremySkinner/FluentValidation/blob/64b78d6bdc9595d221b4d56ce70a00e6de08aa4e/src/FluentValidation/AbstractValidator.cs#L111) piece in validating objects. If my assumption is correct then according to that line of a code all I have to do is create a `PropertyRule` and add `IPropertyValidator's` for the different validations (required, max length, etc). It's turning out to be a lot simpler than I first thought. To prove that assumption I wrote a test to see if the PropertyRule produced the behavior I expected...a validated object with a friendly error message. 

```c#
public void should_validate_type()
{
   // Create the property rule and add the validator as FluentValidation does internally
   var rule = PropertyRule.Create<Person, string>(x => x.Name);
   rule.AddValidator(new NotEmptyValidator(defaultValueForType: null));
   
   var instanceToValidate = new Person { Name = null };

   var results = rule.Validate(new ValidationContext(instanceToValidate));

   results.Select(x => x.ErrorMessage)
          .SingleOrDefault()
          .ShouldBe("'Name' must not be empty.");
}
```

Perfect! The test passed and I know I'm on the right track. I tried to take a very methodical approach, all hard codes to start. If you begin trying to do things dynamically first it makes the task that much harder. If you can't get something to work
with hardcodes how are you going to be able to make it work dynamically? I wanted to [make it work](http://wiki.c2.com/?MakeItWorkMakeItRightMakeItFast) first and prove the concept. Now that I know this works, I have a clear path forward to make it dynamic. Dynamic meaning I won't be writing code like `PropertyRule.Create<Person, string>(x => x.Name)`. At compile time the types will be unknown, so I won't be able to strongly type the generic parameters and lambda expression. 

## Dynamic Property Rule
The next goal is to create a `PropertyRule` dynamically. I'll save "dynamic validators" for later. I have to figure out how to call the factory method `PropertyRule.Create` and pass a lambda expression to it. I could call into `PropertyRule.Create` but the implementation is simple enough I can add my own factory method named `CreateRule`.

The one I added has the same exact signature `PropertyRule CreateRule<TInstance, TProperty>(Expression<Func<TInstance, TProperty>> expression)`, though the implementation is slightly different. Calling `PropertyRule.CreateRule<TInstance, TProperty>()`  is easy enough with the `MakeGenericMethod` api but the hard part is going to be the lambda expression `x => x.Name`. I'll have to hand roll expression trees. 

To do that you can use the [expression api](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/expression-trees/). If you're unfamiliar with expressions and compiled expressions it's a complex but powerful language feature. Essentially, you can create code dynamically and execute it as if it were handwritten. There's a slight performance hit but it can be much faster than reflection. Many libraries like [NServiceBus](https://github.com/Particular/NServiceBus) and [AutoMapper](https://github.com/AutoMapper/AutoMapper) use compiled expression trees.

After some googling and trial and error I successfully created an instance of `PropertyRule` along with the expression parameter dynamically. This was single handedly the hardest part. The test is now

```c#
public void should_validate_type()
{
   var instanceType = typeof(Person);
   var propertyType = typeof(string);
   var propertyFunc = typeof(Func<,>).MakeGenericType(instanceType, propertyType);

   var instance = Expression.Parameter(instanceType, "x");
   var property = Expression.Property(instance, propertyName: "Name");
   var expression = Expression.Lambda(propertyFunc, property, instance);

   // The dynamic version of PropertyRule.Create<Person, string>(x => x.Name)
   var rule = (PropertyRule)typeof(FluentValidationTests)
                            .GetMethod(nameof(CreateRule))
                            .MakeGenericMethod(instanceType, propertyType)
                            .Invoke(null, new object[] { expression });

   rule.AddValidator(new NotEmptyValidator(defaultValueForType: null));
   
   var instanceToValidate = new Person();
   
   var results = rule.Validate(new ValidationContext(instanceToValidate));

   results.Select(x => x.ErrorMessage)
          .SingleOrDefault()
          .ShouldBe("'Name' must not be empty.");
}
```

## Custom Attributes

With the rule being created dynamically I now needed a way of mapping my custom attributes to an `IPropertyValidator` from `FluentValidation`. What I came up with are these couple classes. 

```c#
[AttributeUsage(AttributeTargets.Property)]
public abstract class ValidationAttribute : Attribute 
{

}

public class Required : ValidationAttribute 
{

}


public interface AttributeValidator
{
   bool Matches(ValidationAttribute attribute);
   IPropertyValidator GetValidator(PropertyInfo property, ValidationAttribute attribute);
}

public abstract class AttributeValidator<T> : AttributeValidator where T : ValidationAttribute
{
   public bool Matches(ValidationAttribute attr) => typeof(T) == attr.GetType();

   public IPropertyValidator GetValidator(PropertyInfo property, ValidationAttribute attribute)
   {
       return GetValidator(property, (T)attribute);
   }

   // This will do the mapping from ValidationAttribute to IPropertyValidator
   protected abstract IPropertyValidator GetValidator(PropertyInfo property, T attribute);
}
```

This `Matches => DoSomething` concept is commonly seen in the [Strategy](https://sourcemaking.com/design_patterns/strategy) pattern. I first came across this way of implementing the pattern 
after reading how [FubuMVC](https://github.com/DarthFubuMVC/fubucore/blob/baf77456ee61ff674ebf4fdd8128d4b7a210a72e/src/FubuCore/Conversion/StatelessConverter.cs#L9) uses [strongly typed config classes](https://jeremydmiller.com/2014/11/07/strong_typed_configuration/) (now comes ootb in asp.net core). With these pieces `ValidationAttributes` can automatically be added without having to change existing code. Once you add a new  `ValidationAttribute` and `AttributeValidator<T>` everything will just work.

If you're interested about the strategy pattern, other implementations can be found in [HtmlTags](https://github.com/HtmlTags/htmltags/blob/5ecbedec1beeeecc7d97af8467af6b017cfd5dc3/src/HtmlTags/Conventions/ITagBuilder.cs#L7), [AutoMapper](https://github.com/AutoMapper/AutoMapper/blob/3d485332c6d32314dc1d6d47b44ed1bf26a855e3/src/AutoMapper/IObjectMapper.cs#L21]),  and my not abandoned library [Construktion](https://github.com/Construktion/Construktion/blob/af1fecd1908d8c83b272ba89b1c8918acb11ad2d/Construktion/Blueprint.cs#L10) (as a [chain of responsibility](https://sourcemaking.com/design_patterns/chain_of_responsibility)).

Implementations of `AttributeValidator<T>` look like

```c#
public class RequiredValidator : AttributeValidator<Required>
{
    protected override IPropertyValidator GetValidator(PropertyInfo property, Required attribute)
    {
        var defaultValue = property.PropertyType.IsValueType
                       ? Activator.CreateInstance(property.PropertyType)
                       : null;

        return new NotEmptyValidator(defaultValue);
    }
}

public class MaxLengthValidator : AttributeValidator<MaxLength>
{
    protected override IPropertyValidator GetValidator(PropertyInfo property, MaxLength attribute)
    {
        return new MaximumLengthValidator(attribute.MaxLength);
    }
}
```

Next I need some orchestration class (which is a glorified static class in this case) to handle the strategies.
This class handles the mapping of an `AttributeValidator` to a FluentValidation `IPropertyValidator` and adds it to the `PropertyRule`. The interface `AttributeValidator`
is used as a crutch to make referencing the open the generic easier. 

```c#
public static class ValidatorExtensions
{
  private static readonly List<AttributeValidator> _attributeValidators = typeof(AttributeValidator)
      .Assembly
      .GetTypes()
      .Where(x => typeof(AttributeValidator).IsAssignableFrom(x) && !x.IsAbstract)
      .Select(Activator.CreateInstance)
      .Cast<AttributeValidator>()
      .ToList();

 public static void AddValidator(this PropertyRule rule, PropertyInfo property, ValidationAttribute attribute)
 {
     var attributeValidator = _attributeValidators.SingleOrDefault(x => x.Matches(attribute)) ??
                    throw new Exception(
                       $"Could not find a validator for attribute {attribute.GetType()}. " +
                       $"Please add an implementation of AttributeValidator<{attribute.GetType()}>");
            
     var propertyValidator = attributeValidator.GetValidator(property, attribute);

     rule.AddValidator(propertyValidator);
 }
}
```
Now I just need to add a `RequiredAttribute` to the Name property on my class. The unit test has to grab all
the ValidationAttributes on a property and add their corresponding validator. 

```c#
public void should_validate_type()
{
   var rule = // Created Dynamically
   
   var propertyInfo = typeof(Person).GetProperty("Name");

   var validationAttributes = propertyInfo
                              .GetCustomAttributes(typeof(ValidationAttribute), true)
                              .Cast<ValidationAttribute>()
                              .ToList();

   // Leverage the "orchestrator"
   foreach(var attribute in validationAttributes)
       rule.AddValidator(propertyInfo, attribute)
   
   var instanceToValidate = new Person();

   var results = rule.Validate(new ValidationContext(instanceToValidate));

   results.Select(x => x.ErrorMessage)
          .SingleOrDefault()
          .ShouldBe("'Name' must not be empty.");
}
```

The last thing left to do is create dynamic instances of `IValidator<T>` to inject into the container.
I'll need a marker class to build instances from and my initial thought was `public class CustomValidator<T> : AbstractValidator<T> { }` But how am I going to add the PropertyRules? The base `AbstractValidator<T>` expects the rules to be added through the DSL, not manually like this. It seemed like I was stuck and I went off the rails a little. I chose to implement the interface `IValidator<T>` instead of leveraging `AbstractValidator<T>`. I basically copy and pasted the whole `AbstractValidator<T>` into my custom implementation. It worked but it was pretty annoying that I had to do that, it wasn't until a 2nd pass (which was the writing of this post) that I realized `AbstractValidator<T>` has an `AddRule` method, doh! 

After that epiphany the custom implementation was removed and the CustomValidator class became.

```c#
public class CustomValidator<T> : AbstractValidator<T>
{
     public CustomValidator(IEnumerable<PropertyRule> rules)
     {
         foreach (var rule in rules)
             AddRule(rule);
     }
}
```

Now the final unit test is

```c#
public void should_validate_type()
{
   var rule = // Created Dynamically
   
   // Add validators to rule

   // Create the validator dynamically
   var customValidator = typeof(CustomValidator<>).MakeGenericType(instanceType);
   var validator = (IValidator)Activator.CreateInstance(customValidator, new List<PropertyRule> { rule });
   var instanceToValidate = new Person();

   var result = validator.Validate(instanceToValidate);

   result.Errors
          .Select(x => x.ErrorMessage)
          .SingleOrDefault()
          .ShouldBe("'Name' must not be empty.");
} 
```

This proves the whole concept now! I was able to successfully create an `IValidator<T>` with the only input being attributes on a property. All that's left is to add the `CustomValidator<T>` to the IoC container, [structuremap](http://structuremap.github.io/registration/auto-registration-and-conventions/#sec5) in this case.

As I said in the beginning it's generally a bad idea to bypass the public api of 3rd party libraries. Only hack libraries like this after you've explored all other options and considered the potential consequences. This was a fun project and a lesson learned why as a library author you should think about making more of your types public instead of internal. If these types were internal I'd never be able to extend it in this way. 

Complete [gist](https://gist.github.com/JoeGannon/91e76605a2a7a7944a1a6ccde3abb3c3)