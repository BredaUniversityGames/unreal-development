# Engine Overview

The earlier chapters built a project one file at a time. This chapter steps back and looks at what
surrounds it: where the engine comes from, how its folders are laid out, how configuration is
layered, and which of the tools that ship with it are worth knowing on day one.

Unreal is large: thousands of directories, hundreds of modules, and over twenty-five years of code
from hundreds of people. Nobody knows all of it. The good news is that it is organised the same way
from top to bottom, so once you can read one part of the structure you can read the rest.

Parts of this chapter follow Gerke Max Preussner's talk *A Programmer's Glimpse at UE4*, updated
for 5.8.

## Getting the engine

There are two ways to get the engine, and the book so far has assumed the first.

**The Epic Games Launcher** installs a prebuilt, binary engine. It is ready to use straight away,
and it includes the editor debug symbols if you tick them in the install options. You should,
because without them a callstack into engine code is just addresses. A launcher engine identifies
itself by version number, so the `EngineAssociation` in your `.uproject` reads `"5.8"`
([Creating an Unreal Project](./creating_unreal_project_from_scratch.md) goes through that file).

**The full source from GitHub** comes from Epic's `UnrealEngine` repository, which you can access
once your GitHub account is linked to your Epic account. You build the engine yourself, which takes
a while the first time, and in return you can change it. Two pieces of advice come with that:

- **Change the engine only as a last resort.** Most things you think need an engine change can be
  done in a project module or a plugin. Every engine change is one more thing you have to merge
  again whenever you take a new Unreal release.
- **Make an Installed Build** if a team shares a source engine. An installed build is a prebuilt
  package made from your source tree with UnrealAutomationTool's `BuildGraph`. It behaves like a
  launcher engine, so artists don't need to compile anything. A custom engine registers itself
  under `HKEY_CURRENT_USER\SOFTWARE\Epic Games\Unreal Engine\Builds` with a generated ID, and
  that ID replaces the version number in `EngineAssociation`. If the project sits in a folder
  under the engine's root directory, UnrealVersionSelector finds the engine without needing the
  registry at all.

Either way you get the engine source. Even with a launcher install, `{UE-Root}/Engine/Source` is
right there on your disk, and it's the most accurate documentation you have. The book keeps coming
back to that, and [Additional Workflow Items](./additional_workflow_items.md) explains how to make
it searchable.

## Folder structure

The engine and a project are laid out almost the same way. A project is essentially an extension of
the engine, not a separate thing it opens, so the same folder names mean the same things in both:

| Folder | Holds | Put in version control? |
|---|---|---|
| `Source/` | C++ code, with a `.Target.cs` per target and a `.Build.cs` per module | Yes |
| `Content/` | Assets (`.uasset`, `.umap`) | Yes |
| `Config/` | `.ini` settings | Yes |
| `Plugins/` | Plugins that belong to this project, each laid out like a mini project | Yes |
| `Binaries/` | Compiled DLLs, executables and `.pdb` symbols | Usually no; see below |
| `Intermediate/` | Build products, generated headers, project files | No |
| `Saved/` | Logs, crash dumps, autosaves, local config, cooked output | No |
| `DerivedDataCache/` | Cached, platform-ready asset data | No |
| `{projectname}.uproject` | Defines the project itself | Yes |

You saw most of these appear on disk in [Build an Unreal Project](./building_unreal_project_from_scratch.md)
and [Open an Unreal Project](./opening_unreal_project_from_scratch.md). A few deserve more detail.

**`Content/` is managed from the Content Browser, not from Windows Explorer.** Assets refer to each
other by path. If you move a file in Explorer, every reference to it breaks. If you move it in the
Content Browser, the editor updates the referencers and leaves a redirector behind so nothing
breaks. It also checks out everything it touched in source control. The one time you should go into
`Content/` from the file system is to recover an autosave (below).

**`Saved/` is where you go when something went wrong.** `Saved/Logs/` keeps the logs of previous
sessions, so you can read what happened after the editor has already closed. `Saved/Crashes/` gets a
folder for every crash, containing the log up to that moment and a minidump (`.dmp`). Open the dump
in Visual Studio with the matching symbols and you get the callstack and local variables at the
moment of the crash. `Saved/Autosaves/` mirrors the layout of `Content/`. If the editor crashed and
took your work with it, compare timestamps, and if the autosave is newer, copy it over the
original.

**`Intermediate/` can always be deleted.** Everything in it is regenerated. When a build gets into a
state that makes no sense, deleting `Intermediate/` (and the project's `Binaries/`) is the
standard first step.

**`Binaries/`:** teams where only programmers compile usually keep it out of source control and
share builds another way. Teams without a build machine sometimes commit the editor binaries so
that artists never need a compiler. Both approaches are valid. What matters is that the team
decides deliberately.

**Your own folders.** Nothing stops you from adding folders at the project root. A common one is
`SourceAssets/` for the `.psd`, `.fbx` and `.wav` files that assets are imported *from*. Because
it sits in the project, the import path is the same on every machine, so reimport keeps working
for everyone.

### Plugins

Engine plugins live under `{UE-Root}/Engine/Plugins`, and Fab/Marketplace plugins install there
too. Any project can enable them. Project plugins live in the project's own `Plugins/` folder. Many
engine plugins are enabled by default. If you'd rather opt in to each one, set
`DisableEnginePluginsByDefault` in the `.uproject` (see the table in
[Creating an Unreal Project](./creating_unreal_project_from_scratch.md)).

## Modules

You wrote a module yourself in [Setup an Unreal Project](./setup_unreal_project_from_scratch.md).
The engine is built out of hundreds of them, sorted by where they are allowed to run.
`{UE-Root}/Engine/Source` has one folder per category:

| Folder | Used by |
|---|---|
| `Runtime/` | Games, the editor and programs. Anything that can end up in a shipped game |
| `Editor/` | The editor only |
| `Developer/` | The editor and tools, but never a shipped game |
| `Programs/` | Standalone tools such as UnrealBuildTool and UnrealHeaderTool |
| `ThirdParty/` | External libraries |

One dependency rule follows from that: **a runtime module must not depend on an editor or developer
module**, because that code doesn't exist in a shipped game. Your own modules follow the same rule,
and the `Type` field of each module entry in the `.uproject` (`Runtime`, `Editor`, ...) is how you
declare which kind a module is.

When you're starting out, four modules cover nearly everything. **Core** has the fundamental types
and containers ([Programming in Unreal](./programming_in_unreal.md)). **CoreUObject** is the
`UObject` system: reflection, garbage collection and serialization. **Engine** has actors,
components, worlds and the [Gameplay Framework](./gameplay_framework.md). **Slate** and **UMG**
are the UI. Later you'll find `UnrealEd` (the editor itself), `AssetRegistry`, `Json` and
`JsonUtilities`, `DeveloperSettings` (for adding your own Project Settings page) and
`TargetPlatform`. Editor modules are the best example code for writing your own tools.

Modules still compile to one DLL each for the editor. A packaged game links them all into a single
monolithic executable, which you saw happen in [Running a Game](./running_a_game.md).

On the C++ version: Unreal hasn't been "C with classes" for a long time. **5.8 requires C++20.**
C++17 is marked obsolete and no longer compiles (see `CppStandardVersion` in
`Engine/Source/Programs/UnrealBuildTool/System/CppCompileEnvironment.cs`). You'll still find old
macros such as `OVERRIDE` in parts of the codebase that nobody has cleaned up yet. Write
`override`, `nullptr` and `enum class` in your own code.

## Configuration

Behaviour that isn't in code or content lives in `.ini` files. There isn't just one of each: every
config type (`Engine`, `Game`, `Input`, `Editor`, ...) is a **stack** of files. Each file overrides
the ones before it, and the final result is loaded into class defaults at startup. The order for 5.8
is defined by `GConfigLayers` in `Engine/Source/Runtime/Core/Public/Misc/ConfigHierarchy.h`,
and its core is:

1. `{UE-Root}/Engine/Config/Base{TYPE}.ini`: engine defaults. Read these, never edit them.
2. `{UE-Root}/Engine/Config/{PLATFORM}/Base{PLATFORM}{TYPE}.ini`: engine per-platform defaults.
3. `{project}/Config/Default{TYPE}.ini`: **your project's settings.** This is where Project
   Settings writes.
4. `{UE-Root}/Engine/Config/{PLATFORM}/{PLATFORM}{TYPE}.ini`, then
   `{project}/Config/{PLATFORM}/{PLATFORM}{TYPE}.ini`: per-platform overrides, e.g.
   `Config/Windows/WindowsEngine.ini`.
5. User-level files, and finally `Saved/Config/`: local overrides for this machine only. The editor
   saves per-user state here, and it never goes into source control.

When a setting "doesn't stick", it's almost always because a later layer overrides it. Reading the
`Base` files is also the best way to find out which settings and console variables exist. Browse
them. You'll find things you didn't know you could change.

A section named `[/Script/ModuleName.ClassName]` sets the defaults of that class's `UPROPERTY(Config)`
properties. Any class can read settings this way, not just engine classes:

```cpp
UCLASS(Config = Game)
class UMyTuning : public UObject
{
    GENERATED_BODY()

    UPROPERTY(Config)
    float SprintMultiplier = 1.5f;
};
```

```ini
; Config/DefaultGame.ini
[/Script/MyProjectCore.MyTuning]
SprintMultiplier=1.75
```

Most day-to-day settings are exposed in **Edit → Project Settings** and **Edit → Editor
Preferences**, which write to these same files, so you won't often need to edit an `.ini` by hand.
Knowing where the value actually lives is what lets you diff it, review it and fix it when the UI
won't.

## Tools worth knowing

These ship with the engine and are useful from your first week:

- **Unreal Insights** (Tools → Unreal Insights, or `{UE-Binaries}/UnrealInsights.exe`) is the
  profiler: timing across all threads, memory, networking, asset loading. The
  [Profiling](./profiling.md) section teaches how to read a capture.
- **Session Frontend** (Tools → Session Frontend) connects to running game sessions (on this PC,
  on the network or on a devkit), streams their log and sends them console commands. The **Device
  Manager** next to it handles the target devices themselves.
- **Reference Viewer** and **Size Map** (right-click an asset) show what an asset pulls in and how
  much memory that costs. They also tell hard references, which always load together, apart from
  soft references, which load on demand. [For Designers and Artists](./for_designers_and_artists.md)
  covers both in depth, and [Asset Manager](./asset_manager.md) explains why the difference matters.
