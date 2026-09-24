# The Asset Manager

Every asset in a game raises three questions:

1. **What is it?** How does code refer to it without hard-coding a file path?
2. **When is it in memory?** Who loads it, when, and what keeps it alive?
3. **Does it ship, and where?** Is it cooked into the build, and in which package file?

For a small game the answers take care of themselves. Everything references everything, it all
loads at startup, and it all ships. As a game grows, that stops working: load times climb, memory
runs out, and the prototype level that "nobody uses" turns out to be in the shipped build. The
**Asset Manager** is Unreal's answer to all three questions at once, and this chapter builds up to
it one piece at a time.

The main source for this chapter is Ben Zeigler's *Inside Unreal: Asset Manager Explained*
livestream. Zeigler wrote the system. Epic's
[Asset Management](https://dev.epicgames.com/documentation/en-us/unreal-engine/asset-management-in-unreal-engine)
documentation and Tom Looman's
[Asset Manager for Data Assets & Async Loading](https://tomlooman.com/unreal-engine-asset-manager-async-loading/)
are good follow-ups. One piece of advice from the livestream is worth repeating first: don't expect
to get your asset setup right the first time. Everyone restructures it at least once.

## The problem: hard references

A **hard reference** is a `UPROPERTY` that points straight at another asset, such as a
`TObjectPtr<UStaticMesh>` or a Blueprint variable of an object type. When the asset holding it
loads, everything it hard-references loads too, and everything *those* reference, and so on.
Referencing one weapon Blueprint from the player can pull in its mesh, its materials, their
textures, its sounds, its impact effects, and every other weapon those happen to reference.
[For Designers and Artists](./for_designers_and_artists.md#hard-and-soft-references) shows how to
spot this in the Reference Viewer and Size Map.

A **soft reference** (`TSoftObjectPtr<T>`, `TSoftClassPtr<T>`, or `FSoftObjectPath`
underneath) stores only the *path*. Nothing loads until you ask for it. That fixes the cascade,
and it gives you a new job: now *you* have to load the asset at the right moment, asynchronously
so the game doesn't hitch, and keep it alive while it's in use. Making every reference soft doesn't
solve your memory problem. It hands you the loading problem instead.

In Blueprint the difference is the variable type. An object reference is hard, and a *soft* object
reference is soft.

### Loading a soft reference yourself

There are two ways to turn the path back into an object:

```cpp
// Blocking: the game thread stalls until the asset is read from disk
USkeletalMesh* LoadedMesh = Mesh.LoadSynchronous();

// Asynchronous: ask, keep running, get a callback
FStreamableManager& Streamer = UAssetManager::GetStreamableManager();
MeshHandle = Streamer.RequestAsyncLoad(Mesh.ToSoftObjectPath(),
    FStreamableDelegate::CreateUObject(this, &AHero::OnMeshLoaded));
```

`LoadSynchronous` is fine in editor tools and at boot, and it causes a visible hitch during
gameplay. The asynchronous version streams the asset in the background. Keep the returned
`TSharedPtr<FStreamableHandle>`: the handle is what keeps the loaded asset alive, and if you drop
it and nothing else references the asset, it can be garbage collected straight after loading. The
Blueprint equivalents are the **Async Load Asset** and **Async Load Class Asset** nodes.

Doing this by hand for five hundred items, with menus, levels and a server that all need different
subsets, gets messy fast. The Asset Manager exists to do that bookkeeping for you.

## The Asset Manager

`UAssetManager` is a single global `UObject`. The engine creates it at startup, in
`UEngine::InitializeObjectReferences`, from the class named in `AssetManagerClassName` in
`DefaultEngine.ini`. It lives until shutdown, so it survives every level and game mode change.
That's how it can hold on to loaded assets between levels. It isn't technically a subsystem, but
it behaves like an engine-lifetime one. It grew out of an older class, `UObjectLibrary`, which could
only find assets of one type.

You never create it. You ask for it:

```cpp
// Anywhere in game code
UAssetManager& Manager = UAssetManager::Get();

// Code that can run before the engine is up: module startup, CDOs, commandlets
if (UAssetManager* EarlyManager = UAssetManager::GetIfInitialized())
{
    // safe to use here
}
```

`Get()` returns a reference and stops with a fatal error if there is no Asset Manager, because
that means the project is misconfigured. `GetIfInitialized()` returns `nullptr` instead. It
replaces `GetIfValid()`, which has been deprecated since 5.3. Blueprint has no node that returns
the Asset Manager itself. You use the nodes that call it instead: **Async Load Primary Asset**,
**Get Primary Asset Id List**, **Get Object from Primary Asset Id** and **Unload Primary Asset**.

## Packages

Before the Asset Manager can find anything, two building blocks need explaining: what an asset is
on disk, and how Unreal knows about assets it hasn't loaded.

The storage unit is the **package**. One asset has three names, and they get mixed up constantly:

| Name | Example | Where you see it |
|---|---|---|
| Package name | `/Game/Items/HealthPotion` | The Content Browser. `/Game` is the `Content/` folder |
| Object path | `/Game/Items/HealthPotion.HealthPotion` | What a soft reference stores (package, then object) |
| File on disk | `Content/Items/HealthPotion.uasset` | Explorer; levels are `.umap` |

In the editor a package is one file. It usually holds one main asset plus the sub-objects that
belong to it. A Blueprint package, for example, holds the generated class and its default object.
Material instances and textures aren't sub-objects: they are assets of their own, in packages of
their own. (One package *can* hold several assets, in legacy content and from some importers, but
one asset per package is the rule in UE5.)

Packages reference other packages. The potion uses a mesh, the mesh uses a material, and the
material uses a texture. That dependency graph is what the Reference Viewer draws and what the
cooker walks. When you cook, the editor-only data is stripped out and each package is split into
`.uasset` (the header), `.uexp` (the object data) and `.ubulk` (large data such as texture mips).
You'll never see those last two in your `Content/` folder. They end up inside the container files
described under [Cooking and chunking](#cooking-and-chunking).

## The Asset Registry: knowing without loading

The **Asset Registry** is an index of every package: its name, its asset name, its class, a set of
tags, and its dependencies. One entry is an `FAssetData`. The registry is built by reading package
*headers*, never by loading the objects. That's why the Content Browser can list fifty thousand
assets instantly, and it's the "discovering assets" progress bar you see the first time you open a
large project.

The tags are how you search it. The fixed fields are the package name, asset name and class path,
and on top of those every property you mark `UPROPERTY(AssetRegistrySearchable)`, or that a class
adds by overriding `GetAssetRegistryTags`, is stored as a name/string pair. Tags are written when
the asset is **saved**, so after adding the specifier, re-save the existing assets. From then on you
can ask "every item where `Rarity` is `Rare`" without loading a single item:

```cpp
#include "AssetRegistry/AssetRegistryModule.h"

IAssetRegistry& Registry = IAssetRegistry::GetChecked();

TArray<FAssetData> Items;
Registry.GetAssetsByClass(UItemData::StaticClass()->GetClassPathName(), Items);

for (const FAssetData& Item : Items)
{
    FString Rarity;
    Item.GetTagValue(TEXT("Rarity"), Rarity);   // read from the index, nothing loaded
    UE_LOG(LogTemp, Log, TEXT("%s is %s"), *Item.AssetName.ToString(), *Rarity);
}
```

`GetAssetsByPath` searches a folder, and `GetAssets` takes an `FARFilter` that combines class, path
and tag values. What comes back is only the index entry. `Item.GetAsset()` gets you the real object
but loads it *synchronously*. It's usually better to take `Item.GetSoftObjectPath()` and load that
asynchronously, or to let the Asset Manager do it. Blueprint has the same queries: **Get Asset
Registry**, then **Get Assets By Class** or **Get Assets By Path**.

Some notes for UE5:

- **Use `AssetClassPath` and class path names.** The old `AssetClass` field was deprecated in 5.1.
- **In the editor, the registry fills in asynchronously.** Query it too early and you get a partial
  list. Check `IsLoadingAssets()` and wait for `OnFilesLoaded()`.
- **A cooked copy, `AssetRegistry.bin`, ships with the game.** Runtime queries work, and in a
  packaged game the registry is loaded up front.
- **Projects can strip tags from cooked builds** in the `[AssetRegistry]` section of the engine
  config. If a query works in the editor and returns nothing in a build, check that first.

The API is in `Engine/Source/Runtime/AssetRegistry/Public/AssetRegistry/IAssetRegistry.h`.

The split to remember: **the registry is the catalogue, and the Asset Manager is the librarian.**
The registry knows every asset that exists, loads nothing, and has no opinion about what matters.
It's used engine-wide by the editor, the cooker and your code. The Asset Manager is specific to
your game. You tell it which assets matter, it finds them in the registry, then loads them, unloads
them and applies rules to them.

## Primary and secondary assets

The Asset Manager divides assets into two kinds:

- **Primary assets** are the ones you ask for by name: an item definition, a hero, a map, a quest.
  The Asset Manager tracks them, loads them and unloads them.
- **Secondary assets** are everything else: meshes, textures, sounds, materials. They're never
  loaded directly. They load because a primary asset references them.

That split is what makes the system manageable. You deal with a few hundred primary assets, and the
thousands of secondary assets follow from their references. It's also why rules you set on a
primary asset, such as cook and chunk rules, spread to the secondary assets it pulls in.

Maps are primary assets out of the box, with the type `Map`. Anything else you opt into. Technically
any asset type can be scanned as primary, but if you want a texture to be independently loadable,
it's cleaner to wrap it in a data asset.

An asset is primary when **both** of these are true:

1. Its class returns a valid ID from `GetPrimaryAssetId()`. `UPrimaryDataAsset` does this for
   you. A plain `UDataAsset` does **not**, and that's the most common reason an asset "isn't found".
2. Its type is listed in **Primary Asset Types to Scan** in the Asset Manager settings.

### Primary Asset IDs

A **Primary Asset ID** (`FPrimaryAssetId`) is a type plus a name, written `Type:Name`, for example
`Item:HealthPotion` or `Map:Arena01`. Code, save games and data tables can refer to an asset by ID
instead of by path. IDs are small, they serialize, and they survive the asset being **moved** to
another folder. They don't survive it being **renamed**. For that you add a Primary Asset ID
redirect in the settings.

Because the name part is the asset's *short* name and not its path, two items can't both be called
`HealthPotion`, even in different folders. The Asset Manager warns about duplicates at startup. A
naming convention (`DA_Item_HealthPotion`, or a prefix per type) avoids the problem.

`UPrimaryDataAsset` works out its type from the class hierarchy. It's the first native class going
up from the asset, or the top-most Blueprint class (the rules are spelled out above the class in
`Engine/Source/Runtime/Engine/Classes/Engine/DataAsset.h`). It's clearer to state it outright:

```cpp
UCLASS()
class UItemData : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    virtual FPrimaryAssetId GetPrimaryAssetId() const override
    {
        return FPrimaryAssetId(TEXT("Item"), GetFName());
    }

    UPROPERTY(EditDefaultsOnly, AssetRegistrySearchable)
    FName Rarity;

    UPROPERTY(EditDefaultsOnly, meta = (AssetBundles = "UI"))
    TSoftObjectPtr<UTexture2D> Icon;

    UPROPERTY(EditDefaultsOnly, meta = (AssetBundles = "Gameplay"))
    TSoftObjectPtr<UStaticMesh> WorldMesh;
};
```

### Telling the Asset Manager what to scan

**Project Settings → Game → Asset Manager** lists the primary asset types to scan. Each entry gives
a type name that matches your `GetPrimaryAssetId()`, a base class, the directories to search, and
whether the type is Blueprint *classes* (**Has Blueprint Classes** on) or data asset *instances*
(off). **Is Editor Only** is for types that only tools use: they're never loaded at runtime or
cooked. Keep the directory list tight, because every scanned folder costs startup time, and
scanning all of `/Game` is expensive. The settings are saved to `DefaultGame.ini`, one line per
entry. Editing that file directly works, but needs an editor restart:

```ini
[/Script/Engine.AssetManagerSettings]
+PrimaryAssetTypesToScan=(PrimaryAssetType="Item",AssetBaseClass=/Script/MyProjectCore.ItemData,bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=((Path="/Game/Data/Items")),Rules=(CookRule=AlwaysCook))
```

The engine's own entries (maps and labels) are in `{UE-Root}/Engine/Config/BaseGame.ini`. They're
worth reading as examples. If an asset isn't showing up, check these in order: the class returns
a valid ID, the type name matches exactly, the directory is being scanned, **Has Blueprint
Classes** is set correctly. Then run `AssetManager.DumpTypeSummary` in the console.

## Loading by ID

A load by ID takes five steps. You ask with an ID, optionally naming bundles. The Asset Manager
looks the ID up in the registry data to get the path and bundle lists. It starts an asynchronous
load through its streamable manager while the game keeps running. Your delegate fires when
everything is in memory. Then you fetch the object:

```cpp
#include "Engine/AssetManager.h"

UAssetManager& Manager = UAssetManager::Get();

// One asset, with only its "Gameplay" bundle
const FPrimaryAssetId Id(TEXT("Item"), TEXT("HealthPotion"));
Manager.LoadPrimaryAsset(Id, { TEXT("Gameplay") },
    FStreamableDelegate::CreateUObject(this, &AMyActor::OnItemLoaded, Id));

// Every asset of a type
TArray<FPrimaryAssetId> AllItems;
Manager.GetPrimaryAssetIdList(TEXT("Item"), AllItems);
Manager.LoadPrimaryAssets(AllItems, { TEXT("UI") });
```

Loading is asynchronous. The data streams in in the background, and your delegate runs on the game
thread once it's ready, so don't touch the object before then. If the asset is already loaded, the
delegate still fires, straight away. That way you write the same code for both cases. In the
callback, `Manager.GetPrimaryAssetObject<UItemData>(Id)` gives you the object. It returns `nullptr`
while the asset isn't loaded. `LoadPrimaryAsset` also returns a `TSharedPtr<FStreamableHandle>`. You
don't need to keep it, because the Asset Manager holds its own, but it's useful for progress
(`GetProgress`) or for cancelling.

Getting the list of IDs is free, because it comes from registry data. Loading is what costs, so
filter first where you can, for example on a registry tag via `GetPrimaryAssetData`.
`LoadPrimaryAssetsWithType` loads every asset of a type in one call. Think twice before using it on
a type with five hundred entries. In Blueprint, use **Get Primary Asset Id List** followed by **Async
Load Primary Asset List**.

The Asset Manager keeps what it loaded alive until you call `UnloadPrimaryAsset(Id)`. That doesn't
free anything immediately. It releases the Asset Manager's hold, and the object goes at the next
garbage collection, as long as nothing else references it. If you load soft references yourself
instead, through `FStreamableManager`, keep the returned `FStreamableHandle`. Otherwise the asset
can be collected right after it loads. The Asset Manager doesn't replace garbage collection. It's
one more thing keeping objects alive, on top of ordinary references, which is convenient when you
want items to survive a level change and a leak when you forget to unload.
[UObjects and Garbage Collection](./profiling_uobjects.md) covers the collector itself.

The load functions are declared in `Engine/Source/Runtime/Engine/Classes/Engine/AssetManager.h`.

### Asset bundles

An **asset bundle** is a named group of soft references *inside one primary asset*. In
`UItemData` above, the icon is in the `UI` bundle and the world mesh in the `Gameplay` bundle. The
inventory screen loads items with `UI`, and a pickup in the world loads with `Gameplay`. A hero
might add a `Cinematics` bundle with high-resolution textures that only load for cutscenes.

What ends up in memory is the primary asset itself, everything it hard-references, and the soft
references in whichever bundles are active. Bundles combine, and you can switch them on assets that
are already loaded:

```cpp
// Character select: the hero data asset plus only its UI bundle
Manager.LoadPrimaryAsset(HeroId, { FName("UI") });

// Start of the match: add Gameplay, drop UI
Manager.ChangeBundleStateForPrimaryAssets({ HeroId }, { FName("Gameplay") }, { FName("UI") });
```

Ben Zeigler recommends switching bundles during loading screens, where a hitch doesn't matter.
Bundles are also a common way to keep client-only content such as UI and cosmetics off a dedicated
server.

Two things catch everyone:

- **Bundles only filter soft references.** A hard `TObjectPtr` loads with the asset no matter
  which bundles you ask for. If "everything still loads", check for hard references first.
- **Bundle data is collected when the asset is saved.** After adding `AssetBundles` meta to a
  class, re-save its data assets.

(These are nothing like Unity's AssetBundles, which are downloadable archives. The closest Unreal
equivalent to those is chunks, below.)

## Rules, labels and collections

### Primary asset rules

**Rules** (`FPrimaryAssetRules`) are set per type (the `Rules` of a Primary Asset Types to Scan
entry), per individual asset (the Primary Asset Rules list in the Asset Manager settings), or
through a label (below). With `bApplyRecursively`, which is on by default, they also apply to the
secondary assets a primary asset pulls in. There are three knobs:

- **CookRule**: whether the asset ships. `AlwaysCook`, `NeverCook`, `ProductionNeverCook` (cooks
  in development builds if something references it, never in production), and a few variations.
  `Unknown`, the default, means "cook it if something references it". The full list, with
  comments, is `EPrimaryAssetCookRule` in `Engine/Source/Runtime/Engine/Classes/Engine/AssetManagerTypes.h`.
  `DevelopmentCook` from older tutorials is now a legacy alias for `ProductionNeverCook`.
- **ChunkId**: which chunk the asset, and the secondary assets it pulls in, go into.
- **Priority**: **not** load order. When one secondary asset is referenced by several primary
  assets that have different rules, the higher priority's chunk and cook rule win. Epic's own
  example gives the main menu's label a very high priority, so that everything the menu uses ends
  up in chunk 0.

The Asset Audit window shows the rule that actually applies to each asset.

### Primary asset labels

A **Primary Asset Label** (`UPrimaryAssetLabel`) is a primary asset with no gameplay data of its
own. It exists to apply rules to a group of other assets. You create one in the Content Browser
under **Miscellaneous → Data Asset → PrimaryAssetLabel**. It collects assets in three ways, which
you can combine:

- **Label Assets In My Directory**: everything in the label's folder and its subfolders.
- **Explicit Assets** and **Explicit Blueprints**: a list you pick by hand.
- **Asset Collection**: a Content Browser collection.

Its rules then apply to all of them, and to everything *they* reference. For example, label
`/Game/Cinematics` with `ChunkId=2` and all cinematic content ships in its own chunk. Label
`/Game/Prototype` with `NeverCook`, and it is guaranteed never to ship. Labels are how most projects
assign chunks. The engine scans for labels everywhere under `/Game` by default, so you don't need
to register them. **Is Runtime Label** lets the label itself be loaded at runtime to load everything
it covers. It's off by default, because labels are normally a cook-time tool.

If a label points at a collection, make it a **Shared** collection (checked into source control),
so that the build machine sees the same collection you do. Local and private collections only exist
on your own machine.

### Which grouping tool to use

| | Asset Bundle | Primary Asset Label | Collection |
|---|---|---|---|
| Scope | Inside one primary asset | Many assets, by folder or list | Hand-picked set of assets |
| Used for | Loading *part* of an asset at runtime | Cook and chunk rules in bulk | Organising in the Content Browser |
| Exists at runtime? | Yes | Only in its effect on the build | No, editor only (but a label can reference one) |

## Cooking and chunking

[Running a Game](./running_a_game.md#cooking-content) covers the mechanics of cooking. From the
Asset Manager's side, the cooker starts from a set of **seeds**:

- the maps in **Project Settings → Packaging → List of maps to include in a packaged build**, or
  every map if that list is empty;
- the primary assets the Asset Manager adds, which are only those with `AlwaysCook` (plus the
  `DevelopmentAlways...` rules in a development cook);
- **Additional Asset Directories to Cook** in the same Packaging settings;
- anything still in memory when the Asset Manager finishes its initial loading.

It then follows every reference, **soft ones too**, and cooks whatever it reaches, subject to the
cook rules. A primary asset left on the default `Unknown` rule is therefore only cooked if something
references it. You can read the seeding in `UAssetManager::ModifyCook` in `AssetManager.cpp`.

Following soft references is why a prototype asset turns up in a shipped build: something
references it, even softly. Use the Reference Viewer to find what, then fix the reference or give
the asset a `NeverCook` or `ProductionNeverCook` rule. **Project Settings → Game → Asset Manager →
Only Cook Production Assets** turns any development-only asset that would be cooked into an error.

Cooked packages are bundled into container files for distribution. In UE5 that means **IoStore**
by default (`bUseIoStore=True` in `{UE-Root}/Engine/Config/BaseGame.ini`). Most of the data goes
into a `.ucas` file, with a `.utoc` table of contents, and a small `.pak` sits next to them for
loose files.

**Chunking** splits those containers into several numbered sets. By default everything goes into
chunk 0. Chunk IDs from rules and labels only take effect when **Project Settings → Packaging →
Generate Chunks** is on (`bGenerateChunks`, `False` in `BaseGame.ini`). Each chunk then becomes its
own set of files: `pakchunk0-Windows.pak`/`.utoc`/`.ucas`, `pakchunk1-Windows...`, and so on. Chunk
0 acts as the parent of the others, so anything already in chunk 0 isn't duplicated into another
chunk. More complex parent/child setups use `ChunkDependencyInfo` in the config. The
**ChunkDownloader** plugin (`{UE-Root}/Engine/Plugins/Runtime/ChunkDownloader`) builds optional
downloads on top of chunks.

**Chunks are about files, not memory.** Putting an asset in chunk 3 changes which download it
ships in. It doesn't change when it loads or how much memory it uses. You need chunks for mobile
download size limits, console streaming installs ("start playing at 30%"), DLC and language packs.
A student project almost certainly doesn't need them. What matters is knowing they exist.

### Advanced: overriding the Asset Manager

You can subclass `UAssetManager` and set it under **Project Settings → Engine → General Settings →
Asset Manager Class Name** (written to `AssetManagerClassName` in `DefaultEngine.ini`). Give your
subclass its own static `Get()` that casts `GEngine->AssetManager`, the way Lyra does.

- `StartInitialLoading` and `FinishInitialLoading` let you load game-wide data at startup. Keep in
  mind that anything in memory when initial loading finishes gets cooked.
- `ModifyCook` and `ModifyCookReferences` (editor only) let you add packages to or remove them from
  a cook.
- `GetPackageCookRule` (editor only) decides the cook rule per package, for example `NeverCook` for
  everything under `/Game/Prototype/`.

These signatures have changed between engine versions, so copy them from `AssetManager.h` rather
than from a tutorial. Lyra's `ULyraAssetManager` is a good real-world subclass to read.

## Tools

"Don't expect anyone to do this correctly the first time," as Ben Zeigler puts it, so check the
result with the tools:

- **Reference Viewer** (right-click an asset): who references what. You can toggle hard, soft and
  *management* references. Management references show which primary asset or label owns a
  secondary asset for chunking.
- **Size Map** (right-click an asset): the total size of an asset including everything it pulls
  in. Look for the one outlier that's ten times the rest.
- **Asset Audit** (**Tools → Audit → Asset Audit**): a sortable list with chunk IDs, cook rules and
  primary asset IDs. Switch it to a cooked platform to see cooked sizes, the best quick estimate of
  memory cost. It also exports to CSV.

All three are covered in more depth in [For Designers and Artists](./for_designers_and_artists.md).
- **Console commands:**
  - `AssetManager.DumpTypeSummary`: which types exist, and how many assets of each.
  - `AssetManager.DumpLoadedAssets`: what the Asset Manager currently has loaded.
  - `AssetManager.DumpBundlesForAsset <Type:Name>`: which bundles an asset has, and what's in them.
  - `AssetManager.DumpReferencersForPackage <path>`: why a package is being pulled in.

## Summary

| Term | In one line |
|---|---|
| Package | One asset on disk: a `.uasset` or `.umap` |
| Hard reference | Loads the target together with the referencer |
| Soft reference | A path; nothing loads until someone asks |
| Asset Registry | The index of every asset; never loads anything |
| Asset Manager | Finds, loads and applies rules to primary assets |
| Primary asset | An asset with an ID that you ask for by name |
| Primary Asset ID | `Type:Name`, e.g. `Item:HealthPotion` |
| Asset bundle | A named group of soft references inside one primary asset |
| Primary asset rules | CookRule, ChunkId, and Priority (resolves conflicts, not load order) |
| Primary asset label | A data-less primary asset that applies rules to a group |
| Chunk | Which shipped file set an asset goes in; not memory |

Good asset management isn't about loading less. It's about loading the right thing at the right
time.
