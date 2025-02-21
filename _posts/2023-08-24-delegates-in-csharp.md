---
layout: post
title: "Usage of Delegates in C#"
date: 2024-08-24
author: "Dimitri Pursanov"
tags: [csharp, delegates, dotnet]
---

# Usage of Delegates in C#

Delegates are powerful tools that enable developers to encapsulate methods as objects, allowing for flexible and dynamic method invocation. While their primary purpose is to represent references to methods with a specific signature, delegates offer several interesting and advanced usages in modern C# programming.

## Event Handling

One of the most common uses of delegates is in event handling. In C#, events are based on delegates, allowing methods to be called when an event occurs. The `EventHandler` delegate is often used in this context.

```csharp
using System;

public class Button
{
    public event EventHandler Clicked;

    public void Click()
    {
        Clicked?.Invoke(this, EventArgs.Empty);
    }
}

public class Program
{
    static void Main()
    {
        Button button = new Button();
        button.Clicked += OnButtonClicked;
        button.Click();
    }

    private static void OnButtonClicked(object sender, EventArgs e)
    {
        Console.WriteLine("Button was clicked!");
    }
}
```

In this example, the `Button` class has a `Clicked` event, and the `Program` class subscribes to this event using a delegate method `OnButtonClicked`. When the button is clicked, the event is raised, and the delegate method is invoked.

## Callbacks

```csharp
using System;

public class Calculator
{
    public void Calculate(int a, int b, Func<int, int, int> operation)
    {
        int result = operation(a, b);
        Console.WriteLine($"Result: {result}");
    }
}

public class Program
{
    static void Main()
    {
        Calculator calculator = new Calculator();
        // Different operations
        calculator.Calculate(10, 5, (a, b) => a + b); // Addition
        calculator.Calculate(10, 5, (a, b) => a - b); // Subtraction
    }
}
```

In this example, the `Calculator` class's `Calculate` method takes two integers and a delegate of type `Func<int, int, int>`, representing the operation to be performed. This allows different operations (for example addition, subtraction, etc.) to be passed as arguments, making the `Calculate` method highly flexible.

## Custom Sorting

Delegates can be used to define custom sorting logic in scenarios where the default sorting behavior is insufficient. By passing a delegate that encapsulates the comparison logic, you can sort collections based on custom criteria.

```csharp
using System;
using System.Collections.Generic;

public class Program
{
    public static void Main()
    {
        List<string> names = new List<string> { "John", "Anna", "Peter", "Maria" };

        names.Sort((x, y) => x.Length.CompareTo(y.Length));

        foreach (var name in names)
        {
            Console.WriteLine(name);
        }
    }
}
```

Here, the `Sort` method is provided with a delegate that compares the lengths of the strings in the list. As a result, the list is sorted by the length of the names rather than alphabetically.

## Chain of Responsibility Pattern

The Chain of Responsibility pattern is a behavioral design pattern that allows an event or a request to be passed along a chain of handlers. Each handler in the chain decides whether to process the request or pass it to the next handler in the chain. This pattern promotes the decoupling of request senders and receivers, allowing different parts of a program to handle specific types of requests without needing to know about each other.

```csharp
using System;

public class Program
{
    public delegate void ProcessRequest(string request);

    public static void Main()
    {
        ProcessRequest chain = null;

        chain += HandleLowPriorityRequest;
        chain += HandleHighPriorityRequest;
        chain += HandleDefaultRequest;

        chain("Low");
        chain("High");
        chain("Unknown");
    }

    private static void HandleLowPriorityRequest(string request)
    {
        if (request == "Low")
        {
            Console.WriteLine("Handled by Low Priority Handler");
        }
    }

    private static void HandleHighPriorityRequest(string request)
    {
        if (request == "High")
        {
            Console.WriteLine("Handled by High Priority Handler");
        }
    }

    private static void HandleDefaultRequest(string request)
    {
        Console.WriteLine("Handled by Default Handler");
    }
}
```

In this example, a delegate `ProcessRequest` (represents ANY method that takes a `string` as a parameter and returns `void`) is used to form a chain of responsibility. We start with an empty delegate (`chain = null;`) and then add three methods to it using the `+=` operator. This creates a chain where each method is part of the `chain` delegate. The order of addition matters because it defines the sequence in which the methods are called. If we invoke the `chain` delegate with different request strings: `"Low"`, `"High"`, and `"Unknown"` — each time the delegate is invoked, it triggers the methods in the chain sequentially.

## Functional Programming with Delegates

With the rise of functional programming paradigms in C#, delegates have become essential tools for writing concise and expressive code. Delegates like `Func<>`, `Action<>`, and `Predicate<>` are commonly used to implement higher-order functions and LINQ queries.

```csharp
using System;
using System.Linq;

public class Program
{
    public static void Main()
    {
        int[] numbers = { 1, 2, 3, 4, 5, 6 };

        var evenNumbers = numbers.Where(IsEven).ToList();

        evenNumbers.ForEach(n => Console.WriteLine(n));
    }

    private static bool IsEven(int number)
    {
        return number % 2 == 0;
    }
}
```

In this example, the `IsEven` method is passed as a delegate to the `Where` method in a LINQ query. This approach exemplifies how delegates enable functional programming techniques in C#, making the code more declarative and easier to understand.

---

Delegates in C# provide a versatile way to encapsulate and pass methods around, making them a powerful feature for both simple and advanced programming scenarios. Whether you’re handling events, implementing callback mechanisms, defining custom sorting, or applying design patterns like the Chain of Responsibility, delegates offer a robust solution. Understanding and leveraging these techniques will enhance your ability to write flexible, maintainable, and efficient C# code.