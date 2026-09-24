# Programming in Unreal

The first half of this book is about how an Unreal project is put together. This chapter is about
the code that goes inside it. It covers the things that trip up programmers in their first weeks
with the engine: the vocabulary, the conventions, and the parts of C++ that Unreal has quietly
replaced with its own.

The structure of this chapter, and much of its framing, follows Gerke Max Preussner's talk
*Programming in UE4*, which Epic gave at a number of conferences. It has aged well. The details
below have been updated for 5.8.

## Programming is awesome, and it sucks

It's worth saying out loud once. Programming is awesome because you make something out of nothing
and it moves. It also sucks: the tools are imperfect, every codebase of any size is partly bad,
there's never enough time to do it right, and you are always behind. Peter Welch's essay
[*Programming Sucks*](https://www.stilldrinking.org/programming-sucks) is the most accurate
description of working in a programming team that exists, and Epic's team is no exception.

Unreal is several million lines of that. You won't understand it all, and nobody on the team does
either. What helps is knowing your tools, reading other people's code, following the coding
guidelines, and asking the people around you. The rest of this chapter is meant to get you past
the first wall.

## Common blockers

### Visual Studio does not compile your code

The solution file is a convenience. Compilation is driven by **UnrealBuildTool (UBT)**, which reads
the `.Target.cs` and `.Build.cs` files you wrote in
[Creating an Unreal Project](./creating_unreal_project_from_scratch.md), runs **UnrealHeaderTool
(UHT)** over your headers to generate reflection code, and then calls the platform compiler itself.
Visual Studio and Xcode project files are generated *from* that setup so that an IDE has something
to open. If you have followed this book from the start, you have already seen this for yourself:
you have been building without a solution file the whole time.

### Acronym soup

You will meet these constantly, often without explanation:

| Acronym | Meaning |
|---|---|
| UBT | UnrealBuildTool: builds C++ targets |
| UHT | UnrealHeaderTool: parses `UCLASS`/`UPROPERTY` markup and generates the glue code |
| UAT | UnrealAutomationTool: scripts larger jobs such as build, cook, stage and package (`RunUAT.bat`) |
| DDC | Derived Data Cache: cached, platform-ready versions of your assets |
| CDO | Class Default Object: the one instance per `UCLASS` that holds its default values |
| BP | Blueprint |
| PIE | Play In Editor |
| INI | The text configuration files in `Config/` |

Then there are the code names. Tools tend to keep whatever name the programmer who started them
chose, and those names end up in class names, module names and console variables. **Niagara** is
the particle system, replacing **Cascade**. **Persona** is the animation editor. **PhAT** is the
Physics Asset Tool. **Kismet** was Unreal 3's visual scripting, and it survives in names like
`UKismetMathLibrary`, the class behind most Blueprint math nodes. **Lumen** is runtime global
illumination; **Lightmass** is the older baked version. **Sequencer** is the cinematics tool.
**Cooking** turns editor assets into the platform-specific form a packaged game loads
([Running a Game](./running_a_game.md#cooking-content) covers it).

### Where is `main()`?

It is in `Engine/Source/Runtime/Launch/Private/Launch.cpp`, which calls into
`LaunchEngineLoop.cpp` next to it. If you have written your own engine, it's tempting to start
there to find out "how it all works". Don't start there. There are a couple of million lines between that loop and
the code you will write day to day, and none of it will make sense until you know what the layers
above it are for. Learn the [Gameplay Framework](./gameplay_framework.md) first, then come back
here when you have a specific question.

## Unrealisms

### Type prefixes

Every type name in Unreal carries a prefix, and UHT enforces some of them:

| Prefix | Used for | Examples |
|---|---|---|
| `U` | Classes deriving from `UObject` | `UTexture2D`, `UActorComponent` |
| `A` | Classes deriving from `AActor` | `APawn`, `AGameModeBase` |
| `F` | Every other class or struct | `FVector`, `FName`, `FString` |
| `T` | Templates | `TArray`, `TMap`, `TObjectPtr` |
| `I` | Interfaces | `IConsoleManager` |
| `E` | Enums | `ECollisionChannel` |
| `b` | Boolean variables | `bHidden`, `bCanEverTick` |

The story goes that Tim Sweeney used `F` for floating-point vector types and `U` for Unreal classes
early on, and everyone who joined later assumed it was a rule. Now it is one. Everything is
PascalCase too: functions, parameters, locals, loop counters. It will bother you for a week. Go
along with it anyway, because consistency with the code around you is worth more than your
personal preference.

### UObjects: what C++ does not give you

C++ has no runtime reflection, no garbage collection and no automatic serialization. Unreal wants
all three, so it builds them on top of C++. You mark up your types with macros:

- `UCLASS()` on classes
- `USTRUCT()` on structs
- `UENUM()` on enums
- `UPROPERTY()` on member variables
- `UFUNCTION()` on member functions

UHT reads the markup and generates the code that registers each type with the reflection system:

```cpp
USTRUCT(BlueprintType)
struct FWeaponStats
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    float Damage = 10.0f;

    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    int32 MagazineSize = 30;
};
```

That markup gets you a lot. The Details panel can show and edit these fields. Blueprints can read
them. They are saved to disk and restored when loaded, and they can be replicated over the network.
And a `UPROPERTY` that points to another `UObject` is a reference the garbage collector can see.
That last one matters most. A `UObject*` member *without* `UPROPERTY` is invisible to the collector,
and one of the most common beginner crashes is an object being garbage collected while a raw
pointer to it is still in use. In UE5, prefer `TObjectPtr<T>` for `UObject` member pointers.
[UObjects and Garbage Collection](./profiling_uobjects.md) shows what the collector is doing in a
running game.

### Basic types and containers

Unreal does not use `int`, `long` or `char` directly. `Engine/Source/Runtime/Core/Public/GenericPlatform/GenericPlatform.h`
defines fixed-size types (`int32`, `uint8`, `int64`, `TCHAR`, `ANSICHAR`) so that sizes stay the
same on every platform. Their limits are in `Math/NumericLimits.h`. Since 5.0, `FVector`,
`FVector2D` and friends store **doubles** for large world coordinates (look up the `using` aliases
in `Math/MathFwd.h`), so don't assume a vector is three floats.

The Core module has its own containers, and engine code uses them instead of the standard library:

| Unreal | Closest `std::` equivalent |
|---|---|
| `TArray` | `std::vector` |
| `TMap` | `std::unordered_map` |
| `TSet` | `std::unordered_set` |
| `TSparseArray` | none; an array with stable indices across removals |
| `TQueue` | a lock-free FIFO queue |
| `TLinkedList`, `TDoubleLinkedList` | `std::list` |

It also has delegates (unicast, multicast and dynamic, the last of which Blueprints can bind to),
along with thread-safe variants.

### Smart pointers

For ordinary C++ objects (not `UObject`s), Core provides `TSharedPtr`, `TSharedRef`, `TWeakPtr` and
`TUniquePtr`. They are close to their `std::` equivalents, and you can choose thread-safe reference
counting. For `UObject`s you use `TObjectPtr` (a strong reference the GC tracks, as a
`UPROPERTY`), `TWeakObjectPtr` (doesn't keep the object alive, and becomes null when it's collected)
and `TSoftObjectPtr` (a path to an asset that may not be loaded yet, covered in
[Asset Manager](./asset_manager.md)). Never put a `UObject` in a `TSharedPtr`. The garbage
collector owns its lifetime.

Older material mentions `TAutoPtr` and `TScopedPointer`. Both have since been removed; use
`TUniquePtr`.

### Three string types

| Type | What it is | Use it for |
|---|---|---|
| `FString` | A mutable, heap-allocated string | Building and manipulating text, logging |
| `FName` | An index into a global name table; cheap to copy and compare | Identifiers: asset names, bone names, tags, anything compared often |
| `FText` | Localizable text | Anything a player reads |

Literals need a macro. `TEXT("Hello")` makes a `TCHAR` literal. For localized text, define
`LOCTEXT_NAMESPACE` at the top of the file and use `LOCTEXT("Key", "Hello")`, or give the namespace
inline with `NSLOCTEXT("Namespace", "Key", "Hello")`. The definitions are in
`Internationalization/Internationalization.h`.

**`FName` is case-insensitive** (see the comment above `class FName` in `UObject/NameTypes.h`).
It keeps the casing it was first created with, but `Blast` and `bLast` are the same name. Since
reflected properties are identified by `FName`, you cannot have both as `UPROPERTY`s on one class.

### Macros for everything else

- **Logging:** `UE_LOG(LogCategory, Verbosity, TEXT("..."), ...)`. You declared a category in
  [Setup an Unreal Project](./setup_unreal_project_from_scratch.md).
- **Assertions:** `check(expr)` halts in Debug and Development builds and compiles out of Test and
  Shipping, and `checkSlow(expr)` only runs in Debug. `ensure(expr)` logs a callstack the first time
  it fails and then carries on, so use it for "this should not happen, but the game can survive it".
  The macros are in `Misc/AssertionMacros.h`, and `Misc/Build.h` controls which configuration
  enables which.
- **Localization:** `LOCTEXT_NAMESPACE`, `LOCTEXT` and `NSLOCTEXT`, as above.
- **Slate** (the editor's UI framework): `SLATE_BEGIN_ARGS`, `SLATE_ATTRIBUTE` and friends. You
  will read these long before you ever write one.

## Best practices

**Follow the [Epic C++ Coding Standard](https://dev.epicgames.com/documentation/en-us/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine).**
Code written by thousands of people over decades isn't perfectly consistent. When the standard and
the file in front of you disagree, match the file.

**Names are the hard part.** Pick names that are descriptive but as short as possible, and that
goes for local and loop variables too. Don't invent acronyms of your own; the engine has enough.

**The general principles still apply.** Keep it simple (KISS) and don't build what you don't need
yet (YAGNI). Prefer composition to inheritance, which Unreal's component model already pushes you
towards. Keep modules loosely coupled, and prefer many small, trivial pieces over a few clever
ones. From SOLID, single responsibility, open/closed, Liskov substitution and interface segregation
all carry over well. Dependency injection is clumsy in C++, and Unreal mostly doesn't use it.
Instead it uses the *Hollywood principle* ("don't call us, we'll call you"): the engine calls your
`BeginPlay`, `Tick` and `StartupModule`, not the other way round. The Gang of Four's *Design
Patterns* is worth reading if you haven't yet. Unreal also has an automation and unit-test
framework if you want to work test-first.

## Where to learn more

- **[Epic Developer Community](https://dev.epicgames.com/community/unreal-engine/learning)**: the
  official documentation, tutorials and courses.
- **[Unreal Engine forums](https://forums.unrealengine.com)** and the Unreal Source Discord
  community.
- **Alex Forsythe** and **Mathew Wadstein** on YouTube. Forsythe's videos on the gameplay framework
  and the build process are some of the best explanations anywhere. Wadstein has a short video on
  almost every Blueprint node.
- **The engine source on your disk**, which is always correct for your version. See
  [Additional Workflow Items](./additional_workflow_items.md).
