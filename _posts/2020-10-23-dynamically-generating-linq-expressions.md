---
layout: post
title:  "Dynamically Generating LINQ Expressions"
date:   2020-10-23 11:31:00 +0200
categories: programming
---

## Introduction

[LINQ] (a.k.a Language Integrated Query) is one of the nicer, and more
importantly, ubiquitously used features in C#. It is often used to
operate on collections, but also to generate SQL queries when working
with [Entity Framework][efcore]. The way this wondrous feat is achieved 
is a miracle of its own: Instead of directly emitting code for your LINQ
queries, the compiler generates expression trees (really, a relatively
high-level [AST]) that can be manipulated at runtime before it is being
converted to executable code and executed. This gives Entity Framework and
similar libraries the opportunity to take the LINQ expression and transform
it into an SQL query.

This makes things like the following possible:

```cs
class static MyClass
{
  public static async Task<IList<SomeClass>> SomeFunction(DbSet<SomeClass> dbSet)
  {
    var items = await dbSet
      .Where(e => e.SomeProperty == "Some Random String")
      .ToListAsync();
    return items;
  }
}
```

That statement with the `Where()` call gets translated to something like this:

```sql
SELECT aproperty, someproperty
FROM someclass
WHERE someproperty = 'Some Random String'
```

That is very neat. The whole thing is extremely powerful. Not only do you get
to filter the data, you can also perform joins. And the best part about all
of this: This API is available to everybody!

![So Exciting!][exciting]

So, let's take this goodness for a test drive.

## A Simple Example

Before there was the `nameof()` operator in C# passing the name of
a property to (e.g. for the purpose of binding it to a UI component)
was very error prone. The compiler did not notice any typos and renaming
the property also immediately broke things because the hard-coded string
literal with the old name didn't get updated (either manually or the
automatic refactoring).

However, creative minds came up with a solution: instead of passing in the
name of the property as a string, you would pass in an expression that could
be inspected at runtime and the name of the property could be extracted via
reflection. Of course, the `nameof()` operator is vastly superior to this
work around and you should avoid (ab)using this powerful tool for such a
simple purpose. But for a gentle introduction to expressions it will do.

Below is a small function that takes an expression that accesses a property
on an object and returns the property name. A run-time version of `nameof()` 
if you will (try it out on [.NET Fiddle][fiddle for propname]):

```cs
using System;
using System.Linq.Expressions;
using FluentAssertions;

public static class MyExpressions
{
  public static string NameOfProperty<T>(Expression<Func<T>> expression)
  {
    // Validate argument.
    _ = expression ?? throw new ArgumentNullException(nameof(expression));
    if (expression.Body.NodeType != ExpressionType.MemberAccess)
    {
      throw new ArgumentException("The expression must be a member access expression", nameof(expression));
    }

    // Now that we know we have a member expression in the body
    // we can cast it, get a reference to the MemberInfo
    // and return the property name.
    var memberExpression = (MemberExpression)expression.Body;
    return memberExpression.Member.Name;
  }
}

public class MyClass
{
  public string SomeProperty { get; set; } = "Some Value";
  
  public static void Main()
  {
    var obj = new MyClass();
    
    var propertyName = MyExpressions.NameOfProperty( () => obj.SomeProperty );
    
    propertyName.Should().Be(nameof(MyClass.SomeProperty));
    Console.WriteLine("The name of the property is: {0}", propertyName);
  }
}
```

What does `MyExpressions.NameOfProperty()` do? First, note, that the parameter
type is `Expression<Func<T>>`, but that when the method gets called we pass
a nullary lambda function. The compiler helps us here by **not** compiling
the lambda and passing the AST expression as the argument instead. In the
method the expression is first checked to be not `null` and that the lambda
body has the correct AST node type. This method only determines the name of a
property, trying to get the name of a complex expression (such as arithmetic
expressions, deeply nested properties, method calls, lambda bodies with many
statements, etc.) would be completely nonsensical.

Once we are sure that the expression has the right form we cast it to the
specific type and extract the `MemberInfo` object from which we finally get
the property name to be returned. 

Simple enough, so far. But we haven't created _new_ expressions yet. That is
where the real power comes in.

## The Problem

I recently came across the problem that I had to filter a database table
that used a so-called *composite* key. I.e. row identity was not given by a
single column, but by the combined values of multiple columns, in this case
two columns. I needed to fetch a whole list of items, so I didn't want to
query for each item individually. When using LINQ on collections this is
a fairly straight forward task:

```cs
var listOfItems = new List<(string Key1, int Key2, string Value)>
    {
        ("foo", 1, "A"),
        ("foo", 2, "B"),
        ("bar", 1, "C"),
        ("bar", 2, "D"),
    };

var keysToFind = new[]{ ("foo", 1), ("bar", 2) };
var myItems = listOfItems.Where(i => keysToFind.Contains((i.Key1, i.Key2))).ToList();
```

Piece of cake. However, there's a catch: how would this translate to SQL? With
a single key our SQL might look like this:

```sql
SELECT * FROM mytable WHERE mykey IN (value1, value2)
```

And indeed, Entity Framework does this for single valued keys.

The `IN` clause, however, cannot be used for composite expressions and we would
resort to something like what is shown below:

```sql
SELECT *
FROM mytable
WHERE
     (key1 = v11 AND key2 = v12)
  OR (key1 = v21 AND key2 = v22)
  OR (key1 = v31 AND key2 = v32)
  -- and so on...
```

This can also be translated in a straight-forward manner to LINQ as long as
the number of composite values is known and fixed at compile time:

```cs
var listOfItems = new List<(string Key1, int Key2, string Value)>
    {
        ("foo", 1, "A"),
        ("foo", 2, "B"),
        ("bar", 1, "C"),
        ("bar", 2, "D"),
    };

var keysToFind = new[]{ ("foo", 1), ("bar", 2) };
var myItems = listOfItems.Where(i =>
        (i.Key1 == keysToFind[0].Item1 && i.Key2 == keysToFind[0].Item2)
     || (i.Key1 == keysToFind[1].Item1 && i.Key2 == keysToFind[1].Item2)
     ).ToList();
```

But what to do if the length of `keysToFind` is not known? E.g. if it is a
parameter to your method? One hackish work-around would be chaining
`UNION ALL` clauses between queries that select each item individually,
something that Entity Framework can do too with the `Union()` method:

```sql
SELECT *
FROM mytable
WHERE (key1 = v11 AND key2 = v12)
UNION ALL
    SELECT *
    FROM mytable
    WHERE (key1 = v21 AND key2 = v22)
    UNION ALL
        SELECT *
        FROM mytable
        WHERE (key1 = v31 AND key2 = v32)
            -- and so on...
```

**_Eww, gross._** Never ever do this.

![Eww, gross][eww]

What we need is a way of dynamically chaining `OR` clauses together as we did
with a fixed number of composite key values to search. Something that performs
the _"and so on"_ part in below code:

```cs
var listOfItems = new List<(string Key1, int Key2, string Value)>
    {
        ("foo", 1, "A"),
        ("foo", 2, "B"),
        ("bar", 1, "C"),
        ("bar", 2, "D"),
        // ...
    };

var keysToFind = new[]{ ("foo", 1), ("bar", 2), ("foo", 2), /* ... */ };
var myItems = listOfItems.Where(i =>
       (i.Key1 == keysToFind[0].Item1 && i.Key2 == keysToFind[0].Item2)
    || (i.Key1 == keysToFind[1].Item1 && i.Key2 == keysToFind[1].Item2)
    || (i.Key1 == keysToFind[2].Item1 && i.Key2 == keysToFind[2].Item2)
    // and so on...
    ).ToList();
```

Enter expression building.

## Building Dynamic Expressions

Browsing through the [API documentation] we find that the [Expression class]
looks particularly promising, because it contains many factory methods for
the construction of new expressions. Especially the static methods
[`Expression.Equal()`], [`Expression.AndAlso()`] and [`Expression.OrElse()`]
look exactly like they'd do what we need.

> ### Note
> Do not fall into the trap of using [`Expression.And()`] and [`Expression.Or()`].
> Those correspond to the bitwise operators `&` and `|`; definitely not what we
> went shopping for.

Another piece of the puzzle is missing. One that allows us to capture the key
constants we are searching for. Unsurprisingly, we find it in the in
[`Expression.Constant()`] method.

Having these basic tools we can start sketching out an implementation:

```cs
public static class MyExpressions
{
    public static IQueryable<TEntity> WhereCompositeKeyContainedIn<TEntity, TKey1, TKey2>(
        this IQueryable<TEntity> query,
        Expression<Func<TEntity, TKey1>> key1expression,
        Expression<Func<TEntity, TKey2>> key2expression,
        IEnumerable<(TKey1 Key1, TKey2 Key2)> keysToFind)
    {
        Expression predicate = null;

        foreach (var key in keysToFind)
        {
            // create the predicate for the current key
            // === DOES NOT YET WORK ===
            var keyPredicate = Expression.AndAlso(
                Expression.Equal(key1Expression.Body, Expression.Constant(key.Key1)),
                Expression.Equal(key2Expression.Body, Expression.Constant(key.Key2)));

            // OR it with the previous key predicates
            if (predicate == null)
            {
                predicate = keyPredicate;
            }
            else
            {
                predicate = Expression.OrElse(predicate, keyPredicate);
            }
        }

        // == NEED TO APPLY THE PREDICATE TO QUERY ==

        return query;
    }
}
```

There's quite a bit going on already. First, notice that we defined an
[extension method] called `WhereCompositeKeyContainedIn()` for the interface
`IQueryable<T>`. The method is generic, with type parameters for the entity
type (`TEntity`) and the types of the composite key items (`TKey1` and
`TKey2`). The first parameter of the method is the `IQueryable<TEntity>`
object to which the method should be applied. The second and third arguments
require a bit more explanation. Just as in the first example, they represent
expressions that access the entity properties; in this case they should
return the first and second item of the composite key, respectively. The last
argument is represents contains a collection of composite key values that the
method should find.

The method then iterates over each of those key values and creates a predicate
expression for this individual item, representing one `AND` part of our query.
We use the `Expression.Equal()` to create the equality expression and
`Expression.Constant` to capture the key values.

Next we accumulate all these individual key-predicates into the final predicate
expression using `Expression.OrElse()`.

But as can be seen, there's the part missing where the predicate should actually
be applied to the `query` object. The `predicate` as it is now is only a raw
expression tree, it is not yet callable. For this we need to first turn it into
a lambda expression using [`Expression.Lambda()`]. The lambda expression,
however, needs a parameter expression that can be used to pass an argument
(the entity) into the predicate. We create it with the help of
[`Expression.Parameter()`]:

```cs
public static class MyExpressions
{
    public static IQueryable<TEntity> WhereCompositeKeyContainedIn<TEntity, TKey1, TKey2>(
        this IQueryable<TEntity> query,
        Expression<Func<TEntity, TKey1>> key1Expression,
        Expression<Func<TEntity, TKey2>> key2Expression,
        IEnumerable<(TKey1 Key1, TKey2 Key2)> keysToFind)
    {
        var parameter = Expression.Parameter(typeof(TEntity), "i");
        Expression predicate = null;

        foreach (var key in keysToFind)
        {
            // == SNIP ==
        }

        // create the lambda
        var lambda = Expression.Lambda<Func<TEntity, bool>>(predicate, parameter);

        // apply the lambda
        query = query.Where(lambda);

        return query;
    }
}
```

**We did it!** Let's [try this hot stuff][fiddle fail].

Or rather, no, we didn't.

There's one subtle but fatal flaw: Our new parameter expression is never used
inside the predicate expression. I.e. the lambda will be invoked, the parameter
will be substituted with the entity object, and then ... nothing. It won't
get substituted into `key1Expression` and `key2Expression`, because those
have parameter expressions of their own!

![Failure]

There's two possible solutions to this:

1. Explicitly call the `key1Expression` and `key2Expression` with our
   own `parameter` expression.
2. Somehow replace the parameter expression inside the bodies of
   `key1Expression` and `key2Expression` with our newly created `parameter`
   expression.

Let's try both.

## Invoking the Key Expressions

For this option to work, we need an expression that represents an invocation
of our key expressions. True to its name, [`Expression.Invoke()`] will do
the job.

> ### Note
> No, lambdas and delegates are not _called_, they are _invoked_, and hence
> [`Expression.Call()`] would not work, as it only deals with calling static
> and instance methods.

Let's do this:

```cs
public static class MyExpressions
{
    public static IQueryable<TEntity> WhereCompositeKeyContainedIn<TEntity, TKey1, TKey2>(
        this IQueryable<TEntity> query,
        Expression<Func<TEntity, TKey1>> key1Expression,
        Expression<Func<TEntity, TKey2>> key2Expression,
        IEnumerable<(TKey1 Key1, TKey2 Key2)> keysToFind)
    {
        var parameter = Expression.Parameter(typeof(TEntity), "i");
        Expression predicate = null;

        foreach (var key in keysToFind)
        {
            // create the predicate for the current key
            var keyPredicate = Expression.AndAlso(
                Expression.Equal(
                    Expression.Invoke(key1Expression, parameter),
                    Expression.Constant(key.Key1)),
                Expression.Equal(
                    Expression.Invoke(key2Expression, parameter),
                    Expression.Constant(key.Key2)));

            // OR it with the previous key predicates
            if (predicate == null)
            {
                predicate = keyPredicate;
            }
            else
            {
                predicate = Expression.OrElse(predicate, keyPredicate);
            }
        }

        // create the lambda
        var lambda = Expression.Lambda<Func<TEntity, bool>>(predicate, parameter);

        // apply the lambda
        query = query.Where(lambda);

        return query;
    }
}
```

Now, this works, but it is quite a lot of code and especially the `foreach` and
`if` condition seem a bit bloated.

![Bloated]

Using LINQ we can shrink this considerably:

```cs
public static class MyExpressions
{
    public static IQueryable<TEntity> WhereCompositeKeyContainedIn<TEntity, TKey1, TKey2>(
        this IQueryable<TEntity> query,
        Expression<Func<TEntity, TKey1>> key1Expression,
        Expression<Func<TEntity, TKey2>> key2Expression,
        IEnumerable<(TKey1 Key1, TKey2 Key2)> keysToFind)
    {
        var parameter = Expression.Parameter(typeof(TEntity), "i");

        var predicate = keysToFind
            // create the predicates for the keys
            .Select(key =>
                Expression.AndAlso(
                    Expression.Equal(
                        Expression.Invoke(key1Expression, parameter),
                        Expression.Constant(key.Key1)),
                    Expression.Equal(
                        Expression.Invoke(key2Expression, parameter),
                        Expression.Constant(key.Key2))))
            // aggregate them together with OR
            .Aggregate(Expression.OrElse);

        // create the lambda
        var lambda = Expression.Lambda<Func<TEntity, bool>>(predicate, parameter);

        // apply the lambda
        query = query.Where(lambda);

        return query;
    }
}
```

Let's take this for a [test ride][fiddle with invocations]:

```cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Linq.Expressions;
using FluentAssertions;

class Program
{
    static void Main()
    {
        var listOfItems = new List<(string Key1, int Key2, string Value)>
            {
                ("foo", 1, "A"),
                ("foo", 2, "B"),
                ("bar", 1, "C"),
                ("bar", 2, "D"),
            };

        var keysToFind = new[]{ ("foo", 1), ("bar", 2) };

        var foundItems = listOfItems.AsQueryable()
            .WhereCompositeKeyContainedIn(
                i => i.Key1,
                i => i.Key2,
                keysToFind)
            .ToList();

        foundItems.Should().BeEquivalentTo(("foo", 1, "A"), ("bar", 2, "D"));
    }
}

public class MyExpressions { /* SNIP */ }
```

**It works!**

![celebrate]

Only fly in the ointment: that's potentially a lot of `Expression.Invoke()`'s.
Let's see how we can get rid of them by implementing the second option listed
above.

## Replacing the Parameters in the Key Expressions

Substituting the parameters in the key expressions takes a bit more work.
Somehow we need a way of traversing the expression trees, replacing the
parameters as we encounter them. The `System.Linq.Expressions` library
of course has an API for exactly for this modelled after the [visitor pattern].

If you are not familiar with this pattern, or patterns in general, stop right
here, get your hands on a copy of the [GoF book] and read it. No excuses. It
is _mandatory_ curriculum for every programmer.

Back to the problem at hand. The [`ExpressionVisitor`] class is the base
class that we need to derive from in order to implement our parameter
substitution:

```cs
sealed class ParamReplacer : ExpressionVisitor
{
    private readonly ParameterExpression _param;

    internal ParamReplacer(ParameterExpression param)
    {
        _param = param;
    }

    protected override Expression VisitMember(MemberExpression node)
    {
        return node.Expression.NodeType == ExpressionType.Parameter
            ? Expression.MakeMemberAccess(_param, node.Member)
            : base.VisitMember(node);
    }
}
```

This is straight forward enough. The constructor takes the parameter
expression we want to use in our substitutions. The override of the
`VisitMember(MemberExpression)` method allows us to take some action
whenever the visitor encounters a member expression, and that's the
ones we're after. We then check whether this member expression has
a subexpression that is a parameter expression and if so, we return
a new member access expression that uses the original member but
substitutes the parameter. Otherwise we just call the base class
implementation.

Using this small nugget we can simplify our extension method from above:

```cs
public static class MyExpressions
{
    public static IQueryable<TEntity> WhereCompositeKeyContainedIn<TEntity, TKey1, TKey2>(
        this IQueryable<TEntity> query,
        Expression<Func<TEntity, TKey1>> key1Expression,
        Expression<Func<TEntity, TKey2>> key2Expression,
        IEnumerable<(TKey1 Key1, TKey2 Key2)> keysToFind)
    {
        var parameter = Expression.Parameter(typeof(TEntity), "i");
        var replacer = new ParamReplacer(parameter);
        var key1 = replacer.Visit(key1Expression.Body);
        var key2 = replacer.Visit(key2Expression.Body);

        var predicate = keysToFind
            // create the predicates for the keys
            .Select(key =>
                Expression.AndAlso(
                    Expression.Equal(key1, Expression.Constant(key.Key1)),
                    Expression.Equal(key2, Expression.Constant(key.Key2))))
            // aggregate them together with OR
            .Aggregate(Expression.OrElse);

        // create the lambda
        var lambda = Expression.Lambda<Func<TEntity, bool>>(predicate, parameter);

        // apply the lambda
        query = query.Where(lambda);

        return query;
    }
}
```

[Trying it out][fiddle with substitutions] we see that this works, too!

![dance]

## Conclusion

In this article we saw how we can use the extremely powerful expressions
API to inspect, transform and dynamically generate LINQ code, putting it
to good use on a real world problem.

However, a word of caution is also due: Don't be the proverbial boy with a
hammer for whom suddenly everything looks like a nail. Before you take out
the big guns rethink your problem. And then again. Most times when you have
to resort to this type of black sorcery it is a sure sign of a
[code][code smell] or [design smell]. When writing an application (as opposed
to creating a new library or framework), using this API is almost certainly
wrong and only justifiable in cases where the alternatives would be even
worse or there are external constraints (such as an external database with
compound keys).

[linq]: https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/
[efcore]: https://docs.microsoft.com/en-us/ef/core/
[ast]: https://en.wikipedia.org/wiki/Abstract_syntax_tree
[exciting]: https://media.giphy.com/media/YnBntKOgnUSBkV7bQH/giphy.gif
[fiddle for propname]: https://dotnetfiddle.net/vls4qK
[eww]: https://media1.tenor.com/images/61c46f4d2d4902fc2fa1bb333edab447/tenor.gif?itemid=14149250
[API documentation]: https://docs.microsoft.com/en-us/dotnet/api/system.linq.expressions
[Expression class]: https://docs.microsoft.com/en-us/dotnet/api/system.linq.expressions.expression
[`Expression.Equal()`]: https://docs.microsoft.com/en-us/dotnet/api/system.linq.expressions.expression.equal
[`Expression.AndAlso()`]: https://docs.microsoft.com/en-us/dotnet/api/system.linq.expressions.expression.andalso
[`Expression.OrElse()`]: https://docs.microsoft.com/en-us/dotnet/api/system.linq.expressions.expression.orelse
[`Expression.And()`]: https://docs.microsoft.com/en-us/dotnet/api/system.linq.expressions.expression.and
[`Expression.Or()`]: https://docs.microsoft.com/en-us/dotnet/api/system.linq.expressions.expression.or
[`Expression.Parameter()`]: https://docs.microsoft.com/en-us/dotnet/api/system.linq.expressions.expression.parameter
[`Expression.Constant()`]: https://docs.microsoft.com/en-us/dotnet/api/system.linq.expressions.expression.parameter
[extension method]: https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods
[`Expression.Lambda()`]: https://docs.microsoft.com/en-us/dotnet/api/system.linq.expressions.expression.lambda
[`Expression.Parameter()`]: https://docs.microsoft.com/en-us/dotnet/api/system.linq.expressions.expression.parameter
[fiddle fail]: https://dotnetfiddle.net/Wvm2Vw
[failure]: https://media.giphy.com/media/A1SxC5HRrD3MY/giphy.gif
[`Expression.Call()`]: https://docs.microsoft.com/en-us/dotnet/api/system.linq.expressions.expression.call
[bloated]: https://media.giphy.com/media/gBItxZWZwH3Vu/giphy.gif
[celebrate]: https://media.giphy.com/media/Is1O1TWV0LEJi/giphy.gif
[fiddle with invocations]: https://dotnetfiddle.net/cHUcHX
[visitor pattern]: https://en.wikipedia.org/wiki/Visitor_pattern
[GoF book]: https://en.wikipedia.org/wiki/Design_Patterns
[`ExpressionVisitor`]: https://docs.microsoft.com/en-us/dotnet/api/system.linq.expressions.expressionvisitor
[fiddle with substitutions]: https://dotnetfiddle.net/Vygp8O
[dance]: https://media.giphy.com/media/1TJB4TPjtaEJq/giphy.gif
[code smell]: https://en.wikipedia.org/wiki/Code_smell
[design smell]: https://en.wikipedia.org/wiki/Design_smell