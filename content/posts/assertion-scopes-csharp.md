---
title: "Using Assertion Scopes to execute multiple Assertions in C#"
date: 2023-06-27
draft: false
tags: ["CSharp","Visual Studio","Unit Testing","Dotnet"]
ShowToc: true
TocOpen: true
cover:
    image: https://dev-to-uploads.s3.amazonaws.com/uploads/articles/s99t9hkx3ozf28lvp0lm.png
    alt: "Using Assertion Scopes to execute multiple Assertions in C#"
    caption: "Assertion Scopes make our lives easier when using multiple assertions within our unit tests by saving us time and effort when finding out why our tests are failing."
---

[Fluent Assertions](https://fluentassertions.com/basicassertions/) is a .NET library that provides use with a bunch of useful extension methods that allow us to test our C# code in a more natural way.

Let's say for example we are testing the output of a string. Without Fluent Assertions, we might write something like this:

```csharp
string testString = "hello";
string expectedOutput = testString.ToUpper();
Assert.Equal(expectedOutput, "HELLO");
```

If we were to write this test using Fluent Assertions, we could do so like this:

```csharp
string testString = "hello";
string expectedOutput = testString.ToUpper();
expectedOutput.Should().Be("HELLO");
```

See? Much more natural 😊

## Introducing Assertion Scopes

Lets use a more extensive example. Say if I have a class that generates a shopping list like so:

```csharp
public class ShoppingListGenerator
    {
        public static List<Item> GenerateItems()
        {
            return new List<Item>
            {
                new Item
                {
                    Name = "Apple",
                    Quantity = 5
                },
                new Item
                {
                    Name = "Banana",
                    Quantity = 1
                },
                new Item
                {
                    Name = "Orange",
                    Quantity = 3
                }
            };
        }
    }
```

For more complex unit tests, we may want to assert on multiple properties like so:

```csharp
public class ShoppingListGeneratorShould
    {
        [Fact]
        public void MultipleAssertions()
        {
            var testShoppingList = ShoppingListGenerator.GenerateItems();

            testShoppingList.Should().NotBeNullOrEmpty();
            testShoppingList.Should().Contain(new Item { Name = "Cheese", Quantity = 2 });
            testShoppingList.Should().HaveCount(10);
            testShoppingList.Should().OnlyHaveUniqueItems();           
        }
    }
```

This approach is fine, but looking at our code, we can see that this test would fail on the assertion that our list will fail on the .Contain() method, since we don't have an item in our list that contains Cheese. This test would also fail on our .HaveCount() method, since we have only 3 items in our list, not 10.

Let's confirm our thinking by running our test.

![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o7kh0ee41xkcg0gxgic0.png)

We were correct! But in this example, not only has our test failed against our .Contains() method, but it has stopped running our test!

Imagine that we fix this error so that the assertion passes, but then fails on the next assertion. And the next one, then the next one and so forth.

Testing our code like this would be rather tedious. Thankfully we can fix this using a Assertion Scope!

Using Assertion Scopes, we can batch multiple assertions into a AssertionScope so that FluentAssertions will only throw the one exception at the end of the scope with all of our failures.

Let's change our test to use an Assertion Scope.

```csharp
using AssertionScopes;
using FluentAssertions;
using FluentAssertions.Execution;
using System;
using Xunit;

namespace Tests
{
    public class ShoppingListGeneratorShould
    {
        [Fact]
        public void MultipleAssertions()
        {
            var testShoppingList = ShoppingListGenerator.GenerateItems();

            using (new AssertionScope())
            {
                testShoppingList.Should().NotBeNullOrEmpty();
                testShoppingList.Should().Contain(new Item { Name = "Cheese", Quantity = 2 });
                testShoppingList.Should().HaveCount(10);
                testShoppingList.Should().OnlyHaveUniqueItems();
            }                     
        }
    }
}
```

We are now wrapping our Assertions in a using statement and our exception will only be thrown at the point of disposing our AssertionScope.

Let's run the test and see what happens!

![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/i5pph0xph3hv636pwmni.png)

Our test has failed, but here we now get both exceptions thrown back to us!

```csharp
Message: 
    Expected testShoppingList {AssertionScopes.Item
       {
          Name = "Apple"
          Quantity = 5
       }, AssertionScopes.Item
       {
          Name = "Banana"
          Quantity = 1
       }, AssertionScopes.Item
       {
          Name = "Orange"
          Quantity = 3
       }} to contain 
    
    AssertionScopes.Item
    {
       Name = "Cheese"
       Quantity = 2
    }.
    Expected testShoppingList to contain 10 item(s), but found 3.
``` 

Now instead of having to run and re-run our tests each time an assertion fails in our test, we can see all the assertions that are failing in a test at once!

## Conclusion

Fluent Assertions is a great library that allows us to write our C# tests in a more natural and expressive way. Assertion Scopes make our lives easier when using multiple assertions within our unit tests by saving us time and effort when finding out why our tests are failing.

If you want to learn more about Fluent Assertions, check out the docs here: https://fluentassertions.com/about/

I hope you learned something from this post! If you have any questions, comment below or reach out to me on Twitter: https://twitter.com/willvelida

Happy Coding! ☕💻