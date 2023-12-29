---
layout: post
title:  "Model Binding away from Primitive Obsession"
date:   2023-12-25 14:46:08 +0100
categories: csharp aspnet
---

Type systems are great, yet often we don't use them to their full potential. We often use primitive types to represent domain concepts. For example, we may use an `int` to represent a `CustomerId` or a `string` to represent a `Week`. This post will attempt to answer the question: what can I do at the ASP.NET Web API boundary to avoid using primitive types? Sadly, it turns out things aren't so simple, and thus be forewarned: one might choose to not use ASP.NET's model binding capabilities for this purpose.

Let's take a rather simple case: the number `123` representing a `CustomerId`. We can use a `record` to represent this:

{% highlight csharp %}
public readonly record struct CustomerId(int Value) { }
{% endhighlight %}

The [model binding documentation](https://learn.microsoft.com/en-us/aspnet/core/mvc/models/model-binding?view=aspnetcore-8.0) mentions multiple options for converting `123` into a `CustomerId`, amongst them:

1. `IParsable<T>.TryParse`, which converts a `string` into a `T`. 
2. `TryParse`, which is accessed through reflection, and considered a lesser alternative to option 1. Therefore, we'll ignore it.
3. Input formatters, which convert request body content, e.g. `JsonConverter<T>` for JSON.
4. `TypeConverter`, which converts an `object` to another `object` and does so non-generically thus adding boxing overhead.

In the remainder of this post we'll see that the combination of option 1 and 3 works best to cover all regular cases.

## Option 1: IParsable&lt;T&gt;

`IParsable<T>` was introduced with .NET 7 as part of [generic math](https://learn.microsoft.com/en-us/dotnet/standard/generics/math) functionality. Its declaration is quite interesting, as it uses the curiously recurring template pattern: `interface IParsable<TSelf> where TSelf : IParsable<TSelf>?`. More information on this pattern can be found [here](https://zpbappi.com/curiously-recurring-template-pattern-in-csharp/).

`IParsable<T>` only applies when passing data through the request URI (`FromQuery` or `FromRoute`) or as form data (`FromForm`), as in the case of JSON (`FromBody`) we would represent the data as a `number`. A first attempt at using `IParsable<T>` would be:

{% highlight csharp %}
public readonly record struct CustomerId(int Value)
    : IParsable<CustomerId>,
{
    public static CustomerId Parse(string s, IFormatProvider? provider)
    {
        return !TryParse(s, provider, out CustomerId result)
            ? throw new ArgumentException("Could not parse supplied value.", nameof(s))
            : result;
    }

    public static bool TryParse(
        [NotNullWhen(true)] string? s,
        IFormatProvider? provider,
        [MaybeNullWhen(false)] out CustomerId result
    )
    {
        if (int.TryParse(s, provider, out var parsed))
        {
            result = new CustomerId(parsed);
            return true;
        }

        result = default;
        return false;
    }
}
{% endhighlight %}

Holy smokes, that's quite a lot of code, especially when one would consider the need to repeat this for every `Id` type. Can we do better?

### Generic type construction

Functions that convert from a number to a `CustomerId`, `EmployeeId`, or whatever other `Id` are all unary, that is: they take one argument. More specifically, they take one argument `T` where `T : INumber<T>` and output the `TId`. As I'm not familiar with the C# type system defining such a construct itself, I defined it myself:

{% highlight csharp %}
public interface IUnaryConstructor<TInner, TSelf>
{
    static abstract TSelf Create(TInner value);
}
{% endhighlight %}

In this, I left out the `T : INumber<T>` requirement, as that leaves the `interface` as generic as possible.

With this, the `Parse` and `TryParse` methods can be implemented generically as:

{% highlight csharp %}
public static class Parsable
{
    public static T Parse<T>(this string s, IFormatProvider? formatProvider)
        where T : IParsable<T>
    {
        if (!T.TryParse(s, formatProvider, out T? result))
        {
            throw new ArgumentException("Could not parse supplied value.", nameof(s));
        }

        return result;
    }

    public static bool TryParse<TInner, T>(
        [NotNullWhen(true)] this string? s,
        IFormatProvider? provider,
        [MaybeNullWhen(false)] out T result
    )
        where T : IUnaryConstructor<TInner, T>
        where TInner : IParsable<TInner>
    {
        if (TInner.TryParse(s, provider, out TInner? parsed))
        {
            result = T.Create(parsed);
            return true;
        }

        result = default;
        return false;
    }

}
{% endhighlight %}

Then `CustomerId` and all other unary constructor types can be shortened to:

{% highlight csharp %}
public readonly record struct CustomerId(int Value)
    : IParsable<CustomerId>,
        IUnaryConstructor<int, CustomerId>
{
    public static CustomerId Create(int value) => new(value);

    public static CustomerId Parse(string s, IFormatProvider? provider) =>
        s.Parse<CustomerId>(provider);

    public static bool TryParse(
        [NotNullWhen(true)] string? s,
        IFormatProvider? provider,
        [MaybeNullWhen(false)] out CustomerId result
    ) => s.TryParse<int, CustomerId>(provider, out result);
}
{% endhighlight %}

With `IParsable<CustomerId>` set up, we can now use the `CustomerId` in both minimal API as well as regular API controllers:

{% highlight csharp %}
// Minimal API
app.MapPost("/minimal-query", ([FromQuery] CustomerId id) => Results.Ok(id.Value));

app.MapPost("/minimal-form", ([FromForm] CustomerId id) => Results.Ok(id.Value))
    // Disable antiforgery token handling for demo purposes
    .DisableAntiforgery();

// Regular API
[HttpPost("from-query")]
public IActionResult FromQuery([FromQuery] CustomerId id) => Ok(id.Value);

[HttpPost("from-form")]
public IActionResult FromForm([FromForm] CustomerId id) => Ok(id.Value);
{% endhighlight %}

## Option 3: Input formatters

The data can also be sent in through the request body. This may be inferred by ASP.NET or explicitly declared through the `FromBody` attribute. In either case ASP.NET model binding effectively gives way to input formatters, as described [here](https://learn.microsoft.com/en-us/aspnet/core/mvc/models/model-binding?view=aspnetcore-8.0#frombody-attribute):

> Apply the `[FromBody]` attribute to a parameter to populate its properties from the body of an HTTP request. The ASP.NET Core runtime delegates the responsibility of reading the body to an input formatter.

To process a HTTP request body containing JSON, a `JsonConverter<T>` must be created. An initial implementation could be:

{% highlight csharp %}
// Declaring the converter on the type itself.
[JsonConverter(typeof(CustomerIdJsonConverter))]
public readonly record struct CustomerId(int Value)

// The converter implementation.
public class CustomerIdJsonConverter : JsonConverter<CustomerId>
{
    public override CustomerId Read(
        ref Utf8JsonReader reader,
        Type typeToConvert,
        JsonSerializerOptions options
    ) =>
        new(
            JsonSerializer.Deserialize<int?>(ref reader, options)
                ?? throw new JsonException(
                    $"Could not deserialize the given {reader.TokenType.ToString().ToLowerInvariant()} to {nameof(CustomerId)}."
                )
        );

    public override void Write(
        Utf8JsonWriter writer,
        CustomerId value,
        JsonSerializerOptions options
    ) => writer.WriteNumberValue(value.Value);
}
{% endhighlight %}

A `JsonConverter<T>` does not only convert from some value to `T`, but also converts from `T` to some JSON value, thus requiring us to implement both `Read` and `Write`. The implementation suffers from the same problem as the `IParsable<T>` implementation: it's not generic.

### Generic type construction

We previously defined `IUnaryConstructor<TInner, TSelf>`, which we can now use to define a generic `JsonConverter<T>`. As we also need to provide a generic `Write` implementation we're still missing a key element: a way to generically retrieve the value from any `T`, with `T` being a tuple with one element, i.e. a singleton:

{% highlight csharp %}
public interface ISingleton<TInner>
{
    TInner Value { get; }
}
{% endhighlight %}

Using `IUnaryConstructor<TInner, TSelf>` and `ISingleton<TInner>` we can now define a generic `JsonConverter<T>` for any `T` that holds a single numeric value:

{% highlight csharp %}
public class UnaryNumberConstructorJsonConverter<T, TInner> : JsonConverter<T>
    where T : IUnaryConstructor<TInner, T>, ISingleton<TInner>
    where TInner : INumberBase<TInner>
{
    public override T Read(
        ref Utf8JsonReader reader,
        Type typeToConvert,
        JsonSerializerOptions options
    ) =>
        T.Create(
            JsonSerializer.Deserialize<TInner>(ref reader, options)
                ?? throw new JsonException(
                    $"Could not deserialize the given {reader.TokenType.ToString().ToLowerInvariant()} to {typeToConvert.Name}."
                )
        );

    public override void Write(Utf8JsonWriter writer, T value, JsonSerializerOptions options) =>
        writer.WriteNumberValue(decimal.CreateChecked(value.Value));
}
{% endhighlight %}

Thereafter, we modify the `JsonConverterAttribute` usage and implement `ISingleton<int>` on `CustomerId`. The latter being easy as the `Value` property is already present:

{% highlight csharp %}
[JsonConverter(typeof(UnaryNumberConstructorJsonConverter<CustomerId, int>))]
public readonly record struct CustomerId(int Value)
    : IParsable<CustomerId>,
        IUnaryConstructor<int, CustomerId>,
        ISingleton<int>
{
}
{% endhighlight %}


With the `JsonConverter<T>` set up, we can now use the `CustomerId` in both minimal API as well as regular API controllers:

{% highlight csharp %}
// Minimal API
app.MapPost("/minimal-body", ([FromBody] Request request) => Results.Ok(request.Id));

// Regular API
[HttpPost("from-body")]
public IActionResult FromBody([FromBody] Request input) => Ok(input.Id);
{% endhighlight %}

## Option 4: the curious case of TypeConverter

Traditionally, the `TypeConverter` type provides a unified way of converting types, yet it's no longer the default for handling in ASP.NET. One reason for this might be that the non-generic nature of `TypeConverter` adds boxing overhead. One can still use `TypeConverter` if one must, though this does require creating an intermediary `JsonConverter<T>` implementation. One such implementation is provided [here](https://github.com/dotnet/runtime/issues/1761#issuecomment-723647307) by [Constantinos Leftheris](https://github.com/cleftheris):

{% highlight csharp %}
public class TypeConverterJsonAdapter<T> : JsonConverter<T>
{
    public override T? Read(
        ref Utf8JsonReader reader,
        Type typeToConvert,
        JsonSerializerOptions options
    )
    {
        var converter = TypeDescriptor.GetConverter(typeToConvert);
        var text = reader.GetString();
        return (T)converter.ConvertFromString(text);
    }

    public override void Write(
        Utf8JsonWriter writer,
        T objectToWrite,
        JsonSerializerOptions options
    )
    {
        var converter = TypeDescriptor.GetConverter(objectToWrite);
        var text = converter.ConvertToString(objectToWrite);
        writer.WriteStringValue(text);
    }

    public override bool CanConvert(Type typeToConvert)
    {
        var hasConverter = typeToConvert
            .GetCustomAttributes<TypeConverterAttribute>(inherit: true)
            .Any();
        if (!hasConverter)
        {
            return false;
        }
        return true;
    }
}

public class TypeConverterJsonAdapter : TypeConverterJsonAdapter<object> { }
{% endhighlight %}

It can be configured within `Program.cs` as follows:

{% highlight csharp %}
// For regular ApiControllers
builder.Services
    .AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.Converters.Add(new TypeConverterJsonAdapter());
    });

// For minimal API
builder.Services.Configure<Microsoft.AspNetCore.Http.Json.JsonOptions>(options =>
{
    options.SerializerOptions.Converters.Add(new TypeConverterJsonAdapter());
});
{% endhighlight %}

The downside of this implementation is that it only supports `string` input and output. Adapting it to support other types is possible, but not without its own problems, as all kinds of assumptions must be made on the incoming numeric type and its numeric representation within e.g. `CustomerId`. Due to this I would not recommend using `TypeConverter`.

{% highlight csharp %}
public override T? Read(
    ref Utf8JsonReader reader,
    Type typeToConvert,
    JsonSerializerOptions options
)
{
    TypeConverter converter = TypeDescriptor.GetConverter(typeToConvert);
    if (reader.TokenType == JsonTokenType.Null)
    {
        return default;
    }
    return (T?)
        converter.ConvertFrom(
            reader.TokenType switch
            {
                JsonTokenType.String => reader.GetString(),
                // Assuming the number is representable as an int32 is a
                // big assumption.
                // We also don't know at this point if e.g. the CustomerId
                // has an internal int32 representation.
                // We could inspect the typeToConvert, to find e.g. the
                // type of `TInner` in its `ISingleton<TInner>`
                // implementation, but should we?
                JsonTokenType.Number
                    => reader.GetInt32(),
                JsonTokenType.True => reader.GetBoolean(),
                JsonTokenType.False => reader.GetBoolean(),
                _ => throw new NotImplementedException(),
            }
        );
}
{% endhighlight %}


## OpenAPI Swagger

Without any further configuration the OpenAPI specification generated by Swagger will not understand it should represent `CustomerId` as a `number`:

{% highlight json %}
{
  "id": {
    "value": 0
  }
}
{% endhighlight %}

Thus declaring the type as a `number` is required:

{% highlight csharp %}
builder.Services.AddSwaggerGen(options =>
{
    options.MapType<CustomerId>(() => new OpenApiSchema { Type = "number" });
});
{% endhighlight %}

Like before, we run into the problem of this not being a very generic solution to the problem. One could improve the implementation by iterating over all types within e.g. the assembly containing `CustomerId` and then finding all `ISingleton<T>` implementations which have a numeric representation:

{% highlight csharp %}
var numericTypes = new HashSet<Type>
{
    typeof(int), // ...
};

foreach (Type type in typeof(CustomerId).Assembly.GetTypes())
{
    Type? tInner = type.GetInterfaces()
        .Where(i => i.IsGenericType && i.GetGenericTypeDefinition() == typeof(ISingleton<>))
        .Select(i => i.GetGenericArguments().First())
        .FirstOrDefault();

    if (tInner != null && numericTypes.Contains(tInner))
    {
        options.MapType(type, () => new OpenApiSchema { Type = "number" });
    }
}
{% endhighlight %}

The above implementation could be adjusted to support other types such as `string` or `boolean` by modifying the `HashSet<Type>` to a `Dictionary<Type, Func<OpenApiSchema>>`, which is something I'll leave as an exercise to the reader.

## Conclusion

I covered a lot of ground in this post. It might leave you wondering if this is the right path to take. After all, this is a lot of code to do something that is seemingly very trivial. Why not simply use the primitive type in the contract and immediately translate it to the domain type within the controller? Using some form of automatic mapping perhaps? Those are definitely options to consider! I am certainly not advocating for the approach described in this post, but I do hope it has given you some insights that you can use in making the right choice.
