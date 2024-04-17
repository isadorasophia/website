+++
author = "Isadora"
title = "Adventures serializing absolutely everything in C#"
date = "2024-04-16"
description = "Sure it can't be hard to serialize all your data structures..."
tags = [
    "csharp",
    "serialization",
]
subtitle = "Also known as porting Newtonsoft to System.Text.Json."
+++

Welcome to my fantastic journey porting [Newtonsoft.Json](https://github.com/JamesNK/Newtonsoft.Json) (a very cool and dead library) to System.Text.Json (C# now built-in serialization).

<!--more-->

Note that this whole article is written as of **April, 2024 (.NET 8)**. In other words, the chance that all of this will be legacy code in a year or so is very high, because C# keeps doubling its speed every year and the best practices of today are the legacy practices of tomorrow...

<center> 
    <img width="180" src="/images/serialization/club-penguin.png" /> <br>
    <i>How this article will age a year from now.</i>
</center>

## Why would you do all this work?
The target for this is, of course, my game engine: [Murder Engine](https://github.com/isadorasophia/murder). This is an extremely long-term project of my own and I always have the best technical interests in mind when making decisions around it. These were my reasons:

- Trim as much dependencies as I can.
- Target native AOT when compiling games for web and consoles.
- Always stay on top of the latest .NET releases.
- Performance...? Question mark?

Please note that I did not make this decision out of performance. This was not an issue at the time and I could not know how would System.Text.Json performed against Newtonsoft ahead of time. If performance was my focus, I would have just opened Perfview and trimmed all the hot paths until Newtonsoft was the absolute bottleneck. This was not the time for that, so I made this migration being okay even with a performance regression. I would soon find out, anyway.

The part that was bothering me the most was that Newtonsoft was not being updated or kept up with the latest C# features (`init` properties or native AOT). While this is understandable, it just meant that it was time for me to move on.

## Ingredients for a perfect serializer
I am very picky regarding serialization in my engine. I want my workflow to be _fast_ and I want to think of serialization as less as I can when focused on making content. I also want to keep implementation details hidden _as often as I can_. This means readonly and private fields everywhere. In summary:

- Private fields
- Properties with private setters
- Ignore getter only properties
- Immutable arrays and dictionaries
- Dictionaries with keys other than strings or enums
- **Polymorphism**: Interface collections have to be okay for implementations in **different assemblies**

It was very easy to do all of them in Newtonsoft with custom converters (which basically only uses reflection). It was almost impossible to do this in .NET 7 with System.Text.Json and source generation. However, as of .NET 8, things started to look better...

<center> 
    <img width="180" src="/images/serialization/sisyphus.png" /> <br>
    <i>How it felt dealing with each serialization problem.</i> <br><br>
</center>

Each of the items above were an adventure of its own. I will now break down into sections for each of the problems I solved. Hopefully this will make it easier to read if you are only interested in some of the solutions and don't care about the other ones:

1. [Polymorphism from an interface declared in a separate assembly](#1)

### 1. Polymorphism from an interface declared in a separate assembly {#1}
While .NET 8 does support polymorphism, all the docs suggest that I do something like this:

```csharp
[JsonDerivedType(typeof(PrefabAsset))]
public abstract class GameAsset 
{ ...
```

Which is simply **not practical** to me. `GameAsset` is declared on `Murder.dll`, the engine assembly. My game, `Game.dll`, will import the engine and implement as many assets as it wants. This means that the engine has no visibility over my game types, by design. However! There was another option. I could **tweak the contract model itself so it supports my types**. They cover this a [tiny bit](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/polymorphism?pivots=dotnet-8-0#configure-polymorphism-with-the-contract-model) in the official documentation. Mine is slightly different:

```csharp
[JsonSerializable(typeof(Murder.Assets.GameAsset))]
[JsonSerializable(typeof(Murder.Assets.PrefabAsset))]
public partial class MurderSourceGenerationContext : JsonSerializerContext { }

private static readonly JsonSerializerOptions _options = new()
{
    TypeInfoResolver = MurderSourceGenerationContext.Default
    .WithModifiers(
        static typeInfo =>
        {
            if (typeInfo.Type == typeof(Murder.Assets.GameAsset))
            {
                typeInfo.PolymorphismOptions = new()
                {
                    DerivedTypes =
                    {
                        new JsonDerivedType(typeof(Murder.Assets.PrefabAsset), 
                            typeDiscriminator: "Murder.Assets.PrefabAsset"))
                    }
                }
            }
        })
};
```

`WithModifiers` implementation looks like this:

```csharp
public static IJsonTypeInfoResolver WithModifiers(this IJsonTypeInfoResolver resolver, params Action<JsonTypeInfo>[] modifiers)
    => new ModifierResolver(resolver, modifiers);

private sealed class ModifierResolver : IJsonTypeInfoResolver
{
    private readonly IJsonTypeInfoResolver _source;
    private readonly Action<JsonTypeInfo>[] _modifiers;

    public ModifierResolver(IJsonTypeInfoResolver source, Action<JsonTypeInfo>[] modifiers)
    {
        _source = source;
        _modifiers = modifiers;
    }

    public JsonTypeInfo? GetTypeInfo(Type type, JsonSerializerOptions options)
    {
        JsonTypeInfo? typeInfo = _source.GetTypeInfo(type, options);
        if (typeInfo != null)
        {
            foreach (Action<JsonTypeInfo> modifier in _modifiers)
            {
                modifier(typeInfo);
            }
        }

        return typeInfo;
    }
}
```

> I **highly recommend** trying all this in a separate console project. Do not just start doing it in your project. This will likely get overwheming and mess up your assets very quickly. You want a controlled environment.

This effectively allows me to do something like:

```csharp
PrefabAsset prefab = new();

string json = JsonSerializer.Serialize<GameAsset>(asset, _options);
PrefabAsset deserializedPrefab = 
    JsonSerializer.Deserialize<GameAsset>(json, _options);
```

This also worked for fields like:

```csharp
public class PrefabAsset : GameAsset
{
    public readonly ImmutableArray<IComponents> Components;
}
```

As long as I added all the components that derived from `IComponent` using the code above, this is precisely what I needed in my engine. 

### 2. Serialize readonly fields and ignore readonly properties {#2}
The serializer actually does support readonly fields by default!

```csharp
private static readonly JsonSerializerOptions _options = new()
{
    IncludeFields = true,
    IgnoreReadOnlyFields = false,
    IgnoreReadOnlyProperties = true
};
```

You may notice that I am ignoring readonly **properties**, because I want to avoid serializing properties such as:

```csharp
public class PrefabAsset : GameAsset
{
    [Serialize]
    private readonly ImmutableArray<IComponents> _cache;

    public readonly ImmutableArray<IComponents> Components => _cache;
}
```

Except that it **doesn't really work**. By design, `IgnoreReadOnlyProperties` will skip properties without a setter [except if those own a collection type](https://github.com/dotnet/runtime/issues/37599#issuecomment-669742740). The guidance is to manually add a `[JsonIgnore]` when choosing to not serialize these properties.

Well, this doesn't really work for me. All my getter properties are only there to make it easier to read. If I want to serialize it, I would use `init` for that. I decided to bruteforce with my dear friend, ~*reflection~*.

```csharp
private static readonly JsonSerializerOptions _options = new()
{
    TypeInfoResolver = MurderSourceGenerationContext.Default
        .WithModifiers(MyOwnPropertyModifier),
    IncludeFields = true,
    IgnoreReadOnlyFields = false,
    IgnoreReadOnlyProperties = true
};

private static void MyOwnPropertyModifier(JsonTypeInfo jsonTypeInfo)
{
    if (jsonTypeInfo.Kind != JsonTypeInfoKind.Object)
    {
        return;
    }

    Type? t = jsonTypeInfo.Type;
    if (t.Assembly.FullName is null || t.Assembly.FullName.StartsWith("System"))
    {
        // Ignore system types.
        return;
    }

    for (int i = 0; i < jsonTypeInfo.Properties.Count; ++i)
    {
        JsonPropertyInfo property = jsonTypeInfo.Properties[i];

        bool shouldRemoveProperty = ShouldRemoveProperty(property, t);
        if (shouldRemoveProperty)
        {
            jsonTypeInfo.Properties.RemoveAt(i);
            --i;
        }
    }
}

private static bool ShouldRemoveProperty(JsonPropertyInfo property, Type t)
{
    if (property.ShouldSerialize is not null)
    {
        // This means that are already rules in place that will likely deal 
        // with this serialization.
        return false;
    }

    if (property.Set is not null)
    {
        // Setter is available! Don't bother.
        return false;
    }

    if (t.GetProperty(property.Name) is not PropertyInfo prop)
    {
        // Fields are okay!
        return false;
    }

    if (prop.SetMethod is not null)
    {
        // Private setters are okay.
        property.Set = prop.SetValue;
        return false;
    }

    // Skip readonly properties.
    return true;
}
```

I wasn't happy that I needed to call `t.GetProperty` for every property in the resolver, but there didn't seem to be a perf impact from that specifically. It was mostly a trade-off between doing that and adding a `[JsonIgnore]` for every getter property in my project. While I like perfomance, between that and fast inner dev loop (until a certain point), I would rather go with a fast inner dev loop.

### 3. Private fields
Reading what was new in [System.Text.Json in .NET 8](https://devblogs.microsoft.com/dotnet/system-text-json-in-dotnet-8/), I was very happy to read that they were now supporting [non-public members](https://devblogs.microsoft.com/dotnet/system-text-json-in-dotnet-8/#extend-jsonincludeattribute-and-jsonconstructorattribute-support-to-non-public-members)... as long as they were not private. Which did not solve my problem.

Not only that, but all my readonly fields from [step #2](#2) were only deserialized if **they had a matching constructor parameter**. While this is okay, I can't risk having my asset incorrectly serializing data _and_ I have _way_ too many `public readonly fields` around.

> You may have noticed that I could also use source generation here. This is perfectly reasonable if this is your performance bottleneck. However, this is not really my performance bottleneck and I would rather prioritize readability and simplicity of the code. It's up to you!

At this point, since I was okay with the trade-off of relying on reflection for cases like this, I also used `MyOwnPropertyModifier` from [step #2](#2).

```csharp
private static void MyOwnPropertyModifier(JsonTypeInfo jsonTypeInfo)
{
    // (code from #2)

    while (t is not null && t.Assembly.FullName is string name && 
        !name.StartsWith("System"))
    {
        if (_types.TryGetValue(t, out var extraFieldsInParentType))
        {
            if (extraFieldsInParentType is not null)
            {
                foreach (JsonPropertyInfo info in extraFieldsInParentType)
                {
                    if (!existingProperties.Contains(info.Name))
                    {
                        JsonPropertyInfo infoForThisType = 
                            jsonTypeInfo.CreateJsonPropertyInfo(
                                info.PropertyType, info.Name);

                        infoForThisType.Get = info.Get;
                        infoForThisType.Set = info.Set;

                        jsonTypeInfo.Properties.Add(infoForThisType);

                        existingProperties.Add(infoForThisType.Name);
                    }
                }
            }
        }
        else
        {
            bool fetchedConstructors = false;
            HashSet<string>? parameters = null;

            // Slightly evil in progress code. If a public field is *not* found as
            // any of the constructor parameters,
            // manually use the setter via reflection. I don't care.
            for (int i = 0; i < jsonTypeInfo.Properties.Count; i++)
            {
                JsonPropertyInfo info = jsonTypeInfo.Properties[i];

                if (info.Set is not null)
                {
                    continue;
                }

                if (!fetchedConstructors)
                {
                    parameters = FetchConstructorParameters(t);
                }

                if (parameters is null || !parameters.Contains(info.Name))
                {
                    FieldInfo? field = t.GetField(info.Name);
                    if (field is not null)
                    {
                        info.Set = field.SetValue;
                    }
                }
            }

            List<JsonPropertyInfo>? extraPrivateProperties = null;

            // Now, this is okay. There is not much to do here. If the field is 
            // private, manually fallback to reflection.
            foreach (FieldInfo field in t.GetFields(
                BindingFlags.Instance | BindingFlags.NonPublic))
            {
                if (!Attribute.IsDefined(field, typeof(SerializeAttribute)) && 
                    !Attribute.IsDefined(field, typeof(ShowInEditorAttribute)))
                {
                    // private field should be ignored.
                    continue;
                }

                string fieldName = field.Name;

                // We may need to manually format names for private fields so it 
                // matches with constructor.
                if (fieldName.StartsWith('_'))
                {
                    fieldName = fieldName[1..];
                }

                if (existingProperties.Contains(fieldName))
                {
                    // already tracked?
                    continue;
                }

                JsonPropertyInfo jsonPropertyInfo = jsonTypeInfo.CreateJsonPropertyInfo(field.FieldType, fieldName);

                jsonPropertyInfo.Get = field.GetValue;
                jsonPropertyInfo.Set = field.SetValue;

                jsonTypeInfo.Properties.Add(jsonPropertyInfo);

                extraPrivateProperties ??= [];
                extraPrivateProperties.Add(jsonPropertyInfo);

                existingProperties.Add(jsonPropertyInfo.Name);
            }

            _types[t] = extraPrivateProperties;
        }

        t = t.BaseType;
    }
}

internal static readonly ConcurrentDictionary<Type, List<JsonPropertyInfo>?> 
    _types = [];
```

All my private fields were working after this! You can have a lot of flexibility on choosing the name of the fields on the json itself, for example. I could also choose to make my deserialization performant by using the constructors, depending on the assets.

I used `SerializeAttribute` instead of `JsonIncludeAttribute` because the analyzer yells at me if I add that in private fields, which is perfectly reasonable, probably.

I workarounded by creating my own attribute:

```csharp
[AttributeUsage(AttributeTargets.Property | AttributeTargets.Field,
 AllowMultiple = false)]
public sealed class SerializeAttribute : Attribute
{
    public SerializeAttribute() { }
}
```

Another Fun Thing I learned was when adding a `JsonSerializable` for a type, for example:

### 4. Making it not explode with hot reload 🔥
This is a fun one that I decided to add for a treat! At my resolver implementation, I decided to cache the list of `JsonPropertyInfo` types so I could avoid doing that while going up the inheritance chain of types:

```csharp
internal static readonly ConcurrentDictionary<Type, List<JsonPropertyInfo>?> 
    _types = [];
```

Until I started noticing that some of my deserialization was blowing up when playing the game from the editor. I didn't know why. Pedro told me that he could only reproduce after applying a change with hot reload... which I got suspicious.

My theory is that I _cannot_ cache the field getter and setter of the `JsonPropertyInfo` between hot reload. Probably something around "hot reload retriggers analyzers" to "the metadata changes everywhere". I was able to solve this with:

```csharp
#if DEBUG
[assembly: System.Reflection.Metadata.MetadataUpdateHandler(
    typeof(Murder.Utilities.Serialization.AfterHotReloadApplied))]
namespace Murder.Utilities.Serialization;

internal static class AfterHotReloadApplied
{
    internal static void ClearCache(Type[]? _) { }

    internal static void UpdateApplication(Type[]? _)
    {
        // This is very interesting, but apparently if we keep the previous cache 
        // prior to hot reload the metadata is no longer valid and we would be 
        // unable to populate the fields with reflection.
        SerializationHelper._types.Clear();
    }
}
#endif
```

And I didn't get these errors again and I could still keep my cache!

<br>
