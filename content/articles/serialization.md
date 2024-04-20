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
The target for this is, of course, my game engine: [Murder Engine](https://github.com/isadorasophia/murder). This is an extremely long-term project of my own and I always have the best technical interests in mind when making decisions around it. My motivation was to:

- Trim as much third-party dependencies as I can.
- Target native AOT when compiling games for web and consoles.
- Always stay on top of the latest .NET releases.
- Performance...? Question mark?

Note that I did not make this decision out of performance. This was not an issue at the time and I could not know how would System.Text.Json performed against Newtonsoft ahead of time. If performance was my focus, I would have just opened [PerfView](https://github.com/microsoft/perfview) and trimmed all the hot paths until Newtonsoft was the absolute bottleneck. This was not the time for that, so I made this migration being okay even with a performance regression. I would soon find out, anyway.

The part that was bothering me the most was that Newtonsoft was not being kept up with the latest C# features (`init` properties or native AOT) or any updates, for that matter. While this is understandable, it just meant that it was time for me to move on.

## Recipe for my perfect serializer
I am very picky regarding serialization in my engine. I want my workflow to be _fast_ and I want to think of serialization as little as possible when making content. I also want to keep implementation details hidden _as often as I can_. This means readonly and private fields everywhere. In summary:

- Private fields
- Properties with private setters
- Ignore getter only properties
- A lot of immutable arrays and dictionaries
- Dictionaries with keys other than strings or enums
- **Polymorphism**: Interface collections have to be okay for implementations in **different assemblies**

It was very easy to implement all of these in Newtonsoft with custom converters, which only rely on reflection. It was almost impossible to do this in .NET 7 with System.Text.Json and source generation. However, as of .NET 8, things started to look better...

<center> 
    <img width="180" src="/images/serialization/sisyphus.png" /> <br>
    <i>How it felt dealing with each of the serialization problems.</i> <br><br>
</center>

At the end of all of it, I was able to successfully:

- Convert all my data structures from Newtonsoft to System.Text.Json without any manual work and guaranteeing no loss of data.
- Successfully publish with native AOT and trimming without special casing any types other than my game assemblies.
- Other than replacing my `[JsonProperty]` attributes and getting rid of `using Newtonsoft.Json`, **I didn't need to change any of my existing code** to accomodate the change.

Finally, I would say this article is probably better intended for any of the following people:

- Wants to move to System.Text.Json but needs the extensibility of Newtonsoft
- Is looking to workaround a very specific weird problem in System.Text.Json
- Likes to use private fields and interfaces as much as I do
- Wants to go down a rabbit hole
- <s>Myself one year from now</s>

I will now break down into sections for each of the problems I had to address. Hopefully this will make it easier to read if you are only interested in some of them and don't care about the other ones:

1. [Polymorphism from an interface declared in a separate assembly](#1)
2. [Serialize readonly fields and ignore readonly properties](#2)
3. [Private fields](#3)
4. [Making it not explode with hot reload 🔥](#4)
5. [Serializing weird dictionaries](#5)
6. [Keeping yourself sane by source generating the serializable context](#6)
<!-- 7. [Use metadata from a parent assembly on the source generator](#7) -->

### 1. Polymorphism from an interface declared in a separate assembly {#1}
While System.Text.Json on .NET 8 does support polymorphism, all the docs suggested that I would do something like this:

```csharp
[JsonDerivedType(typeof(PrefabAsset))]
public abstract class GameAsset 
{ ...
```

Which is just not reasonable to me. `GameAsset` is declared on `Murder.dll`, the engine assembly. My game, `Game.dll`, imports the engine and implements as many assets as it wants. This means that the engine has no visibility over my game types, by design. 

However! There was another option. I could **tweak the contract model itself** so it supports my derived types. They cover this a [tiny bit](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/polymorphism?pivots=dotnet-8-0#configure-polymorphism-with-the-contract-model) in the official documentation. Mine ended up slightly different:

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

The implementation of `WithModifiers` looks like this:

```csharp
public static IJsonTypeInfoResolver WithModifiers(
    this IJsonTypeInfoResolver resolver, 
    params Action<JsonTypeInfo>[] modifiers)
    => new ModifierResolver(resolver, modifiers);

private sealed class ModifierResolver : IJsonTypeInfoResolver
{
    private readonly IJsonTypeInfoResolver _source;
    private readonly Action<JsonTypeInfo>[] _modifiers;

    public ModifierResolver(IJsonTypeInfoResolver source, 
        Action<JsonTypeInfo>[] modifiers)
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

> I **highly recommend** trying all of this in a separate console project. Do not just start doing it in your project, or this will likely get overwheming and mess up your assets very quickly. You want a controlled environment.

This effectively allowed me to do something like:

```csharp
PrefabAsset prefab = new();

string json = JsonSerializer.Serialize<GameAsset>(asset, _options);
PrefabAsset deserializedPrefab = 
    JsonSerializer.Deserialize<GameAsset>(json, _options);
```

This also worked when deserializing interfaces from fields:

```csharp
public class PrefabAsset : GameAsset
{
    public readonly ImmutableArray<IComponents> Components;
}
```

As long as I added all the components that derive from `IComponent` in the code above, this is precisely what I needed for my engine. 

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

Except that it **doesn't really work**. By design, `IgnoreReadOnlyProperties` will skip properties without a setter [except if those return a collection type](https://github.com/dotnet/runtime/issues/37599#issuecomment-669742740). The guidance is to manually add a `[JsonIgnore]` when choosing to not serialize these properties...

Well, this doesn't really work for me. All my getter properties are only there so it's easier to read. If I wanted to serialize it, I would've used `init` for that. So I decided to bruteforce my way with ~*reflection~*.

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

I wasn't happy that I needed to call `t.GetProperty` for every property in the resolver, but that didn't seem to have a perf impact. It was mostly a trade-off between doing that or adding a `[JsonIgnore]` for every getter property in my project. While I like perfomance, between that and a fast inner dev loop, I would rather go with a faster dev loop (until a certain point).

### 3. Private fields {#3}
Reading what was new in [System.Text.Json in .NET 8](https://devblogs.microsoft.com/dotnet/system-text-json-in-dotnet-8/), I was very happy to see that they were now supporting [non-public members](https://devblogs.microsoft.com/dotnet/system-text-json-in-dotnet-8/#extend-jsonincludeattribute-and-jsonconstructorattribute-support-to-non-public-members)... as long as they were not private. Which did not solve my problem.

Not only that, but I noticed that all my readonly fields from [step #2](#2) were only deserialized if **they had a matching constructor parameter**. While this seemed okay, I can't risk having my asset incorrectly serializing data since I have _way_ too many `public readonly fields` around. It just doesn't scale well.

> You may have noticed that I could've used source generation to solve this. This is perfectly reasonable if this is your performance bottleneck. However, this was not really my performance bottleneck and I chose to prioritize readability and simplicity of the code. There are no right answers here, it's up to you.

I also used the same `MyOwnPropertyModifier` from [step #2](#2) to address this:

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

By manually scanning all the private fields, property setters, and public readonly fields that do not have a matching constructor parameter, I can finally serialize and deserialize them as a json property.

You can also have a lot of flexibility on things like choosing the name of the fields on the json. If I wanted to make my deserialization more performant, I can just implement matching constructors for that type, if used often enough.

I also defined a custom `SerializeAttribute` instead of the default `JsonIncludeAttribute`, because the analyzer yells at me whenever I added that in private fields. Which is perfectly reasonable, probably.

My own attribute looked like this:

```csharp
[AttributeUsage(AttributeTargets.Property | AttributeTargets.Field,
 AllowMultiple = false)]
public sealed class SerializeAttribute : Attribute
{
    public SerializeAttribute() { }
}
```

And we are not done yet! While that implementation worked, Another Fun Thing I learned was that when adding a `JsonSerializable` for a type, for example, `MoveAroundInteraction`:

```csharp
public readonly struct MoveAroundInteraction : IInteraction
{
    public readonly ImmutableArray<string> Positions = [];

    [Serialize]
    private readonly FloatRange _interval = new();
```

With the following `JsonSerializableAttribute`:

```csharp
[JsonSerializable(typeof(MoveAroundInteraction))]
public partial class MurderSourceGenerationContext : JsonSerializerContext { }
```

System.Text.Json will be smart enough to generate a `JsonTypeInfo` for both serializing and deserializing `MoveAroundInteraction` and any of its serializable fields, such as `ImmutableArray<string>`. However, the source generation **won't pick up any of my private fields**, such as `FloatRange`, because I never told it that it was being a part of the serialization in compilation time, it's all through reflection! Or runtime.

So that was fun. You will need to manually (or use another source generator, who knows) tell the generator to keep a `JsonTypeInfo` for you for all the private members in order to successfully deserialize `MoveAroundInteraction` with its private fields.

```csharp
[JsonSerializable(typeof(MoveAroundInteraction))]
[JsonSerializable(typeof(FloatRange))] // private member is manually added
public partial class MurderSourceGenerationContext : JsonSerializerContext { }
```


### 4. Making it not explode with hot reload 🔥 {#4}
This is a fun one that I decided to add for a treat! At my resolver implementation, I decided to cache the list of `JsonPropertyInfo` types so I could avoid doing that while going up the inheritance chain of types:

```csharp
internal static readonly ConcurrentDictionary<Type, List<JsonPropertyInfo>?> 
    _types = [];
```

Until I started noticing that some of my deserialization was blowing up when playing the game from the editor. I didn't know why. Pedro told me that he could only reproduce after applying a change with hot reload... which I got suspicious.

My theory is that I _cannot_ cache the field getter and setter of the `JsonPropertyInfo` between hot reload operations. Probably something around "hot reload retriggers analyzers" to "there are metadata changes everywhere". I was able to solve this with:

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

### 5. Serializing weird dictionaries {#5}
Of course I also had the weirdest dictionaries. Stuff like this:

```csharp
public class BlackboardTracker
{        
    [Serialize]
    private readonly Dictionary
       <(Guid Character, int SituationId, int DialogId), int> _dialogCounter = [];
}
```

By default, System.Text.Json only support string keys for dictionaries, so unless I implement a custom converter for this specific dictionary, I cannot serialize this. I also did not want to implement a converter for every each dictionary with a non-string key.

To be fair, this was also a problem in Newtonsoft, which I workarounded by:

```csharp
[JsonArray]
public class ComplexDictionary<TKey, TValue> : Dictionary<TKey, TValue> 
    where TKey : notnull { }
```

This effectively allows Newtonsoft to deserialize dictionaries as if they were an array of KeyAndValue types. I figured the same solution could also exist in System.Text.Json...

This is the closest I got:

```csharp
private static readonly JsonSerializerOptions _options = new()
{        
    Converters =
    {
        new ComplexDictionaryConverter<
            (System.Guid Character, System.Int32 SituationId, 
                System.Int32 DialogId), System.Int32>(),
        new ComplexDictionaryConverter<
            Murder.Core.Dialogs.DialogItemId, Bang.Components.IComponent>(),
        // Add converter for all ComplexDictionary that will serialized...
    }
};
```

I was able to reuse `ComplexDictionary` and share the converter implementation for all these dictionaries. This was my implementation (probably a lot of improvements can be made here):

```csharp
public sealed class ComplexDictionaryConverter<T, V> : 
    JsonConverter<ComplexDictionary<T, V>> where T : notnull
{
    private bool _initialized = false;

    private JsonConverter<T>? _keyConverter = null;
    private JsonConverter<V>? _valueResolver = null;

    // Write the json.
    public override void Write(Utf8JsonWriter writer, 
        ComplexDictionary<T, V> dictionary, JsonSerializerOptions options)
    {
        InitializeArrayResolver(options);

        if (_keyConverter is not JsonConverter<T> converterKey)
        {
            throw new InvalidOperationException($"Unable to serialize 
                {typeof(T).Name}. Could not find a valid JsonConverter.");
        }

        if (_valueResolver is not JsonConverter<V> converterValue)
        {
            throw new InvalidOperationException($"Unable to serialize 
                {typeof(V).Name}. Could not find a valid JsonConverter.");
        }

        writer.WriteStartObject();

        // For every key value pair, effectively serialize it as an array.
        // Make sure the json is still happy with how it's formatted.
        foreach ((T key, V value) in dictionary)
        {
            writer.WritePropertyName("key");
            converterKey.Write(writer, key, options);

            writer.WritePropertyName("value");
            converterValue.Write(writer, value, options);
        }

        writer.WriteEndObject();
    }

    // Read a serialized json.
    public override ComplexDictionary<T, V>? Read(ref Utf8JsonReader reader, 
        Type typeToConvert, JsonSerializerOptions options)
    {
        InitializeArrayResolver(options);

        if (_keyConverter is not JsonConverter<T> converterKey)
        {
            throw new InvalidOperationException($"Unable to serialize 
                {typeof(T).Name}. Could not find a valid JsonConverter.");
        }

        if (_valueResolver is not JsonConverter<V> converterValue)
        {
            throw new InvalidOperationException($"Unable to serialize 
                {typeof(V).Name}. Could not find a valid JsonConverter.");
        }

        ComplexDictionary<T, V> result = [];
        while (reader.Read()) // {
        {
            if (reader.TokenType == JsonTokenType.EndObject)
            {
                break;
            }

            reader.Read(); // "key:"
            T? key = converterKey.Read(ref reader, typeof(T), options);

            reader.Read(); // "}"
            reader.Read(); // "value:"
            V? value = converterValue.Read(ref reader, typeof(V), options);

            if (key is null || value is null)
            {
                throw new JsonException(
                    $"Unable to read key or value for {typeToConvert.Name}");
            }

            result[key] = value;
        }

        return result;
    }

    // Cache the converters for these value and keys.
    private void InitializeArrayResolver(JsonSerializerOptions options)
    {
        if (!_initialized)
        {
            options.TryGetTypeInfo(typeof(T), out JsonTypeInfo? keyInfo);
            options.TryGetTypeInfo(typeof(V), out JsonTypeInfo? valueInfo);

            _keyConverter = keyInfo?.Converter as JsonConverter<T>;
            _valueResolver = valueInfo?.Converter as JsonConverter<V>;

            _initialized = true;
        }
    }
}

```

> _Why not make generic converters in runtime?_
> Again, I need this to work in native AOT. Apparently, it absolutely does not work well to instantiate generic types on the fly in native AOT because of how generic works in native code. Even if you skip trimming your assembly, a generic type that has not existed in compilation will throw an exception if created in runtime.

This worked well enough for me. However, there was another case I stumbled upon while serializing **dictionary keys of System.Type**. This can be a serious security vulnerability, which I wasn't aware before. I workarounded it with this implementation, use it at your own risk:

```csharp
public class JsonTypeConverter : JsonConverter<Type>
{
    public override Type Read(ref Utf8JsonReader reader, 
        Type typeToConvert, JsonSerializerOptions options)
    {
        string? assemblyQualifiedName = reader.GetString();
        if (assemblyQualifiedName is null || Type.GetType(assemblyQualifiedName) 
            is not Type t)
        {
            throw new NotSupportedException("Invalid type for json.");
        }

        return t;
    }

    public override Type ReadAsPropertyName(
        ref Utf8JsonReader reader,
        Type typeToConvert,
        JsonSerializerOptions options)
    {
        return Read(ref reader, typeToConvert, options);
    }

    public override void Write( 
        Utf8JsonWriter writer,
        Type value,
        JsonSerializerOptions options)
    {
        string? assemblyQualifiedName = value.AssemblyQualifiedName;
        writer.WriteStringValue(assemblyQualifiedName);
    }

    public override void WriteAsPropertyName(
        Utf8JsonWriter writer,
        [DisallowNull] Type value,
        JsonSerializerOptions options)
    {
        string? assemblyQualifiedName = value.AssemblyQualifiedName;
        writer.WritePropertyName(assemblyQualifiedName ?? string.Empty);
    }
}
```

And don't forget to add the converter in your options:
```csharp
private static readonly JsonSerializerOptions _options = new()
{        
    Converters =
    {
        new JsonTypeConverter()
    }
};
```

### 6. Keeping yourself sane by source generating the serializable context {#6}
Now, I was finally at a place that I was happy with the code and could see myself actually porting all of this to my engine. There was just one problem: I need this to scale. I want to write something and forget that serialization can be a problem as much as I can.

Since System.Text.Json source generates everything, I thought I could myself also source generate the code that will be source generated by System.Text.Json.

As I have only dealt with reflection but never with source generation, it was probably time to learn. I made a simple source generator that simply defined a component as serializable:

```csharp
[JsonSerializable(typeof(PositionComponent))]
public partial class MurderSourceGenerationContext : JsonSerializerContext { }
```

I decided to make a quick test and deserialize a `PositionComponent` to see how it worked out and...- nothing! Nothing at all! As I would soon learn, **C# source generators do not generate source from generated source**. You can read it yourself (and thumbs up the issue, who knows?) [here](https://github.com/dotnet/roslyn/issues/57239).

Look, this is fair, I get it, but I still needed it to be done. I was close to hit my rock bottom here - I got everything working and in place, and I found yet another blocker. _"I gotta keep going"_, I thought to myself. _"Maybe I can copy the generated code from the binary and ship in source, and then I only need to compile the project twice..."_. It was all unmanageable and risky, which I would like to avoid.

After reading a lot of posts of people saying _"This is a bad idea and you should not do that"_ and other half of people saying _"Yeah, BUT"_, I finally found an angel who brought me this idea:

```csharp
public void Initialize(IncrementalGeneratorInitializationContext context)
{
    var structs = context.PotentialStructs().Collect();
    var classes = context.PotentialClasses().Collect();

    var compilation = structs
        .Combine(classes)
        .Combine(context.CompilationProvider);

    context.RegisterSourceOutput(
        compilation,
        EmitSource
    );
}

private void EmitSource(SourceProductionContext context, 
    (((ImmutableArray<TypeDeclarationSyntax>, 
       ImmutableArray<ClassDeclarationSyntax>), Compilation w)) input)
{
    // actually generate the source here on whatever metadata types you collected.
    
    SourceWriter contextWriter = Templates.GenerateContext(metadata, projectName);
    SourceText contextSourceText = contextWriter.ToSourceText();
    context.AddSource(contextWriter.Filename, contextSourceText);

    SourceWriter optionsWriter = Templates.GenerateOptions(metadata, projectName);
    SourceText optionsSourceText = optionsWriter.ToSourceText();
    context.AddSource(optionsWriter.Filename, optionsSourceText);
    
    RunIllegalSecondSourceGenerator(context, compilation, 
        contextSourceText, optionsSourceText);
}
```

Now, you can say, _"What do you mean with `RunIllegalSecondSourceGenerator`?"_, which I reply with: _"heheheheheh"_.

```csharp
private static void RunIllegalSecondSourceGenerator(
    SourceProductionContext context, Compilation compilation, 
    params SourceText[] sources)
{
    ParseOptions options;
    if (compilation is CSharpCompilation csharpCompilation && 
        csharpCompilation.SyntaxTrees.Length > 0)
    {
        options = csharpCompilation.SyntaxTrees[0].Options;
    }
    else
    {
        options = CSharpParseOptions.Default.WithLanguageVersion(
            LanguageVersion.Latest);
    }

    // Add all sources to our compilation.
    foreach (SourceText source in sources)
    {
        SyntaxTree syntaxTree = SyntaxFactory.ParseSyntaxTree(source, options);
        compilation = compilation.AddSyntaxTrees(syntaxTree);
    }

    Assembly? a = AppDomain.CurrentDomain.GetAssemblies().FirstOrDefault(
        a => a.FullName.Contains("System.Text.Json.SourceGeneration"));
    Type? textJsonForbiddenImporter = a?.GetType(
        "System.Text.Json.SourceGeneration.JsonSourceGenerator");

    if (textJsonForbiddenImporter is null)
    {
        Debug.Fail("Unable to find System.Text.Json generator?");
        return;
    }

    // See declaration of type at
    // https://github.com/dotnet/runtime/blob/c5bead63f8386f716b8ddd909c93086b3546efed/src/libraries/System.Text.Json/gen/JsonSourceGenerator.Roslyn4.0.cs
    ISourceGenerator jsonGenerator =
        ((IIncrementalGenerator)Activator
            .CreateInstance(textJsonForbiddenImporter)).AsSourceGenerator();

    GeneratorDriver driver = CSharpGeneratorDriver.Create(jsonGenerator);
    driver = driver.RunGenerators(compilation);

    GeneratorDriverRunResult driverResult = driver.GetRunResult();
    foreach (GeneratorRunResult result in driverResult.Results)
    {
        foreach (GeneratedSourceResult source in result.GeneratedSources)
        {
            context.AddSource("__Custom" + source.HintName, source.SourceText);
        }
    }
}
```

By instantiating the System.Text.Json gemerator myself, I can basically force it to run with my generated source by manually adding them to the syntax tree! I probably don't have to mention this, but this is relying on Roslyn implementation details and is obviously prone to be broken upon updating the .NET version. BUT it works!


### 7. Use metadata from a parent assembly on the source generator {#7}
Once again, I found myself in trouble. While I got this working and was running it with my engine project (`Murder.dll`), I needed this to also run with `Game.dll`. I would also need to define the json options there, as well.

<center> 
    <img width="250" src="/images/serialization/ride-ends.jpg" /> <br>
    <i>Me, probably, at this point.</i>
</center>