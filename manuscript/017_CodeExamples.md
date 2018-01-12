# About code examples

## Notes for Java users

The language of choice for code examples is C#. That said, I wanted the book to be as technology-agnostic as possible, to allow especially Java programmers to benefit from it. I tried using a minimum number of C#-specific features and in several places even made remarks targeted at Java users to make it easier for them. Still, there are some things I could not avoid. That's why I wrote up a list containing several of the differences between Java and C# that Java users could benefit from knowing when reading the book.

### Naming conventions

Most languages have their default naming conventions. For example, in Java, a class name is written with pascal case (e.g. `UserAccount`), methods and fields are written with camel case, e.g. `payTaxes` and constants/readonly fields are typically written in underscored upper-case (e.g. `CONNECTED_NODES`).

C# uses pascal case for both classes and methods (e.g. `UserAccount`, `PayTaxes`, `ConnectedNodes`). For fields, there are several naming conventions. I chose the one starting with underscore (e.g. `_myDependency`). There are other minor differences, but these are the ones you are going to encounter quite often.

### `var` keyword

For example brevity, I chose to use the `var` keyword in the examples. This keyword serves as automatic type inference, e.g.

```csharp
var x = 123; //x inferred as integer
```

Of course, this is no dynamic typing by any means - everything is resolved at compile time.

One more thing - `var` keyword can only be used when the type can be inferred, so ocassionally, you will see me declaring types explicitly as in:

```csharp
List<string> list = null; //list cannot be inferred
```

### `string` as keyword

C# has a `String` type, similar to Java. It allows, however, to write this type name as keyword, e.g. `string` instead of `String`. This is only syntactic sugar which is used by default by the C# community.

### Attributes instead of annotations

In C#, attributes are used for the same purpose as annotations in Java. So, whenever you see:

```csharp
[Whatever]
public void doSomething()
```

think:

```java
@Whatever
public void doSomething()
```

### `readonly` and `const` instead of `final`

Where Java uses final for constants and readonly fields, C# uses two keywords: `const` and `readonly`. Without going into details, whenever you see something like:

```csharp
public class User
{
    private const int DefaultAge = 15;
    private readonly List<int> Marks = new List<int>();
}
```

think:

```java
public class User {
    private final int DEFAULT_AGE = 15;
    private final List<Integer> MARKS = new ArrayList<Integer>();
}
```

### a List<T>

If you are a Java user, note that in C#, `List<T>` is not an abstract class, but a concrete one. it is typically used where you would use an `ArrayList`.

### Generics

One of the biggest difference between Java and C# is how they treat generics. First of all, C# allows using primitive types in generic declarations, so you can write `List<int>` in C# where in Java you have to write `List<Integer>`. 

The other difference is that in C# there is no type erasure as there is in Java. C# code retains all the generic information at runtime. This impacts how most generic APIs are declared and used in C#.

A generic class definition and creation in Java and C# is roughly the same. There is, however, difference on the method level. A generic method in Java is typically written as:

```java
public <T> List<T> createArrayOf(Class<T> type) {
    ...
}
```

and called like this:

```java
List<Integer> ints = createArrayOf(Integer.class);
```

while in C# it is defined like this:

```java
public List<T> CreateArrayOf<T>()
{
    ...
}
```

and is called like this:

```csharp
List<int> ints = CreateArrayOf<int>();
```

These differences are visible in the design of the library for generating test data that I use throughout this book. While in the C# version, one generates test data by writing:

```csharp
var data = Any.Instance<MyData>();
```

the Java version of the library is used like this:

```java
MyData data = Any.instanceOf(MyData.class);
```