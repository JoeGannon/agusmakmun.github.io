---
layout: post
title:  "Write, Smell, Refactor"
---

Parallel to the phrase Red, Green, Refactor is what I've coined Write, Smell, Refactor &trade;. It's a general mindset I find myself in when writing code. This is by no means a hard and fast rule, but it's something that works for me. I'll explain what it is and later will go through my thought process of a real world example. 

## Write

Just as in Red, Green, Refactor, it's important to get the _simplest_ thing working first. Writing code can be hard enough. So, before you're code is ultra-pluggable, ultra-flexible, and ultra-sentient, it needs to work first and foremost. Are you doing something with generics? Well, hard code everything and go from there! With the simple things in place you can use it as a jumping off point to improve further. It may also force you to think differently about the problem. 

## Smell

This one is tricky because it can be hard to agree on what smells or what constitutes "bad code". It always depends and there's many interpretations. A good first start is to look at the [solid principles](https://en.wikipedia.org/wiki/SOLID), [anti patterns](https://sourcemaking.com/antipatterns), and [code smells](https://sourcemaking.com/refactoring/smells). I recommend following up with Jeremy Miller's [putting solid into perspective](https://jeremydmiller.com/2022/08/10/putting-solid-into-perspective/). 

Basically, every now and then take a step back and look at the bigger picture. Ask yourself does what I'm doing make sense or do I feel like I'm coding myself into a corner? As you grow as a developer your nose and intuition will naturally get better. You'll also understand when it's _okay_ to write "bad code".

## Refactor

There's a ton to say here and I can't even begin to scratch the surface. I think it's important to gain a cursory knowledge of the [gang of four patterns](https://sourcemaking.com/design_patterns) and [refactoring techniques](https://sourcemaking.com/refactoring/refactorings). My personal list of the most important patterns are the Builder, Factory, Singleton, Adapter, Decorator, Chain of Responsibility, Command, and Strategy (my personal favorite and in my experience most applicable).

You don't have to go down the rabbit hole completely with the gang of four patterns and learn everything inside and out. The important part is simply knowing they exist. Seriously, that's it. Just know they exist and the problems they solve. When a use case comes along you'll be able to recognize a pattern may fit. It's common to study these and think you'll never know when to apply them, but trust me when I say you just will. 

Another common mistake is attempting to apply the patterns/principles EVERYWHERE. If you're like me, you'll start to think every switch statement and if/else clause is a violation of the open close principle. Or any class with an "AddOrUpdate" method violates Single Responsibility. But those are traps to be avoided, these are principles not rules. It's important to always be pragmatic.
 

## Putting it Together

In our MVC application we had a lot of html for rendering a grid that was being copied and pasted in multiple places. Inspired by [Telerik's grid](https://demos.telerik.com/aspnet-mvc/grid/local-data-binding) helpers I set out to make our own small html helper classes. The result of the below code will be an `html table`. We'll be looking at the code that reads and formats the model properties.

```
@(Html.Grid<WebPageViewModel>()    
    .Columns(columns => {    	
        columns.Bind(x => x.Title);
        columns.Bind(x => x.PageName);
        columns.Bind(x => x.Url);
    }))
```

Behind the scenes each call to `columns.Bind` builds a class of `TableColumn<TModel, TProperty>`. The `GetValue` function is particularly noteworthy. Right now it only returns our property as a string (the simplest thing), later we want to format hyperlinks, textboxes, etc.

```c#
public class TableColumn<TModel, TProperty>
{ 
    public Func<TModel, TProperty> GetProperty { get; set; }

    public TableColumn(Func<TModel, TProperty> getProperty)
    {
        GetProperty = getProperty;
    }

    public string GetValue(TModel model)
    {
        var value = GetProperty(model).ToString();

        return value;
    }
}
```

Expanding the code to allow our properties to be formatted as a textbox or hyperlink results in something like this.

```
@(Html.Grid<WebPageViewModel>()    
    .Columns(columns => {    	
        columns.Bind(x => x.Title);
        columns.Bind(x => x.PageName).TextBox();
        columns.Bind(x => x.Url).HyperLink("display text");
    }))
```

```c#
public class TableColumn<TModel, TProperty>
{ 
    public bool TextBox = false;
    public bool HyperLink = false;
    public string LinkText = "";

    public Func<TModel, TProperty> GetProperty { get; set; }

    public TableColumn(Func<TModel, TProperty> getProperty)
    {
        GetProperty = getProperty;
    }

    public string GetValue(TModel model)
    {
        var value = GetProperty(model).ToString();

        if(TextBox)
        {
            value = $"<input type=\"text\" value=\"{value}\">"
        }
        else if(HyperLink)
        {
            value = $"<a href=\"{value}\">{LinkText}</a>";
        }

        return value;
    }
}
```

How does it hold up to the smell test?

It's not exactly the worst code in the world, but there's room for improvement. The main issue I see (besides poor encapsulation) is that this class will grow linearly with every new feature. Each new format will require new properties and an additional `else if` clause inside the `GetValue` function. `GetValue` also violates the open close principle and with enough of these features this class can quickly become a [large class](https://sourcemaking.com/refactoring/smells/large-class). 

We'll also have to consider how client code will interact with this class. A class like this can be difficult to work with if it requires specific permutations of properties to function properly. 

## Refactoring

At the end of the day all the code is doing is preparing some text/html to be rendered inside a `td` tag and based on some conditions we want that rendered output to be different. 

_Hmmm, that actually sounds a lot like a design pattern I'm familiar with. Oh, that's right it's the strategy pattern! And yes, now I remember the strategy pattern is one way of solving open close violations!_

After refactoring towards the strategy pattern we've now got something that leads to less class bloat and allows client code to have an easier time using our object.

```c#
public class TableColumn<TModel, TProperty>
{ 
    private RenderStrategy _renderStrategy;

    public Func<TModel, TProperty> GetProperty { get; set; }

    public TableColumn(Func<TModel, TProperty> getProperty)
    {
        GetProperty = getProperty;
        _renderStrategy = new DefaultRenderStrategy();
    }

    public string GetValue(TModel model)
    {
        var value = GetProperty(model).ToString();

        value = _renderStrategy.Render(value);

        return value;
    }

    public void SetRenderStrategy(RenderStrategy renderStrategy)
    {
        _renderStrategy = renderStrategy;
    }
}

public interface RenderStrategy 
{
    string Render(string value);
}

public class DefaultRenderStrategy : RenderStrategy 
{
    public string Render(string value) => value;
}

public class TextBoxRenderStrategy : RenderStrategy 
{
    public string Render(string value) => $"<input type=\"text\" value=\"{value}\">";
}

public class HyperLinkRenderStrategy : RenderStrategy 
{
    private string _linkText;

    public HyperLinkRenderStrategy(string linkText)
    {
        _linkText = linkText;
    }

    public string Render(string value) => $"<a href=\"{value}\">{_linkText}</a>";
}

```