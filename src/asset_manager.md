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

The Asset Manager is what organises that loading work.

## The Asset Registry: knowing without loading

Before anything can be loaded by name, something needs to know what exists. The **Asset Registry**
is an index of every asset in the project: its package path, its asset name, its class, and a set
of tags. It holds all of that **without loading any of the assets**, and it's what powers the
Content Browser. The editor builds it at startup (the scan you see when opening a large project)
and again when cooking, and a cooked copy ships with the game, so you can query it at runtime too.

The tags are how you search it. By default they cover a few basics. A property you mark
`UPROPERTY(AssetRegistrySearchable)`, or a tag you add by overriding `GetAssetRegistryTags`, gets
recorded too. So you can ask "find every item where `Rarity` is `Legendary`" without loading a
single item:

```cpp
#include "AssetRegistry/AssetRegistryModule.h"

IAssetRegistry& Registry = IAssetRegistry::GetChecked();

TArray<FAssetData> Items;
Registry.GetAssetsByPath(TEXT("/Game/Data/Items"), Items, /*bRecursive*/ true);

for (const FAssetData& Item : Items)
{
    FString Rarity;
    Item.GetTagValue(TEXT("Rarity"), Rarity);
    UE_LOG(LogTemp, Log, TEXT("%s (%s) rarity=%s"),
        *Item.AssetName.ToString(), *Item.AssetClassPath.ToString(), *Rarity);
}
```

Two notes for UE5. Use `AssetClassPath`, because the old `AssetClass` field was deprecated in
5.1. And in the editor the registry fills in asynchronously, so code that runs early should wait
for `OnFilesLoaded()`. The API is in
`Engine/Source/Runtime/AssetRegistry/Public/AssetRegistry/IAssetRegistry.h`.

The split to remember: **the registry is the database, and the Asset Manager is the manager.** The
registry knows what exists. The Asset Manager decides what matters, loads it and applies rules to
it.

## Packages

The unit the registry indexes is the **package**. In the editor, a package is one `.uasset` or
`.umap` file, with a path like `/Game/Items/HealthPotion` that maps to
`Content/Items/HealthPotion.uasset`. A package usually holds one main asset plus the sub-objects
that belong to it. A Blueprint package, for example, holds the generated class and its default
object. Material instances and textures aren't sub-objects, though: they are assets of their own,
in packages of their own.

Packages reference other packages, and those references form the graph the Reference Viewer draws.
When you cook, the editor-only data is stripped out, and the output is split up differently: large
data moves into `.uexp` and `.ubulk` files, and those end up inside the container files
described under [Cooking and chunking](#cooking-and-chunking).

## Primary and secondary assets

The Asset Manager divides assets into two kinds:

- **Primary assets** are the ones you ask for by name: an item definition, a hero, a map, a quest.
  The Asset Manager tracks them, loads them and unloads them.
- **Secondary assets** are everything else: meshes, textures, sounds, materials. They're never
  loaded directly. They load because a primary asset references them.

That split is what makes the system manageable. You deal with a few hundred primary assets, and the
thousands of secondary assets follow from their references.

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
(off). The settings are saved to `DefaultGame.ini`:

```ini
[/Script/Engine.AssetManagerSettings]
+PrimaryAssetTypesToScan=(PrimaryAssetType="Item",AssetBaseClass=/Script/MyProjectCore.ItemData,bHasBlueprintClasses=False,bIsEditorOnly=False,Directories=((Path="/Game/Data/Items")),Rules=(CookRule=AlwaysCook))
```

The engine's own entries (maps and labels) are in `{UE-Root}/Engine/Config/BaseGame.ini`. They're
worth reading as examples. If an asset isn't showing up, check these in order: the class returns
a valid ID, the type name matches exactly, the directory is being scanned, **Has Blueprint
Classes** is set correctly. Then run `AssetManager.DumpTypeSummary` in the console.

## Loading by ID

`UAssetManager` is a single global `UObject`, created by the engine and alive across every map
change. It behaves much like an engine subsystem. Loading goes through it:

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
callback, `Manager.GetPrimaryAssetObject<UItemData>(Id)` gives you the object.

The Asset Manager keeps what it loaded alive until you call `UnloadPrimaryAsset(Id)`. That doesn't
free anything immediately. It releases the Asset Manager's hold, and the object goes at the next
garbage collection, as long as nothing else references it. If you load soft references yourself
instead, through `FStreamableManager`, keep the returned `FStreamableHandle`. Otherwise the asset
can be collected right after it loads. [UObjects and Garbage Collection](./profiling_uobjects.md)
covers the collector itself.

The load functions are declared in `Engine/Source/Runtime/Engine/Classes/Engine/AssetManager.h`.

### Asset bundles

An **asset bundle** is a named group of soft references *inside one primary asset*. In
`UItemData` above, the icon is in the `UI` bundle and the world mesh in the `Gameplay` bundle. The
inventory screen loads items with `UI`, and a pickup in the world loads with `Gameplay`. A hero
might add a `Cinematics` bundle with high-resolution textures that only load for cutscenes.

Two things catch everyone:

- **Bundles only filter soft references.** A hard `TObjectPtr` loads with the asset no matter
  which bundles you ask for. If "everything still loads", check for hard references first.
- **Bundle data is collected when the asset is saved.** After adding `AssetBundles` meta to a
  class, re-save its data assets.

(These are nothing like Unity's AssetBundles, which are downloadable archives. The closest Unreal
equivalent to those is chunks, below.)

## Rules, labels and collections

### Primary asset rules

Every primary asset type has **rules** (`FPrimaryAssetRules`), and a single asset can override
them:

- **CookRule**: whether the asset ships. `AlwaysCook`, `NeverCook`, `ProductionNeverCook` (cooks
  in development builds if something references it, never in production), and a few variations.
  `Unknown`, the default, means "cook it if something references it". The full list, with
  comments, is `EPrimaryAssetCookRule` in `Engine/Source/Runtime/Engine/Classes/Engine/AssetManagerTypes.h`.
  `DevelopmentCook` from older tutorials is now a legacy alias for `ProductionNeverCook`.
- **ChunkId**: which chunk the asset, and the secondary assets it pulls in, go into.
- **Priority**: **not** load order. When one secondary asset is referenced by several primary
  assets that have different rules, the higher priority's chunk and cook rule win.

### Primary asset labels

A **Primary Asset Label** (`UPrimaryAssetLabel`) is a primary asset with no gameplay data of its
own. It exists to apply rules to a group of other assets. Point it at a folder, a list of assets
or a collection, give it rules, and everything it covers (and everything *they* reference) follows
those rules. For example, label `/Game/Cinematics` with `ChunkId=2` and all cinematic content
ships in its own chunk. Label `/Game/Prototype` with `NeverCook`, and it is guaranteed never to
ship. The engine scans for labels everywhere under `/Game` by default. You don't need to register
them.

### Which grouping tool to use

| | Asset Bundle | Primary Asset Label | Collection |
|---|---|---|---|
| Scope | Inside one primary asset | Many assets, by folder or list | Hand-picked set of assets |
| Used for | Loading *part* of an asset at runtime | Cook and chunk rules in bulk | Organising in the Content Browser |
| Exists at runtime? | Yes | Only in its effect on the build | No, editor only (but a label can reference one) |

## Cooking and chunking

[Running a Game](./running_a_game.md#cooking-content) covers the mechanics of cooking. From the
Asset Manager's side, the cooker starts from the primary assets, maps and labels, follows their
references (**soft ones too**), and cooks whatever it reaches, subject to the cook rules. That's
why a prototype asset turns up in a shipped build: something references it, even softly. Use the
Reference Viewer to find what, then fix the reference or give the asset a `NeverCook` or
`ProductionNeverCook` rule. **Project Settings → Game → Asset Manager → Only Cook Production
Assets** turns any development-only asset that would be cooked into an error.

Cooked packages are bundled into container files for distribution. In UE5 that means **IoStore**
by default (`bUseIoStore=True` in `{UE-Root}/Engine/Config/BaseGame.ini`). Most of the data goes
into a `.ucas` file, with a `.utoc` table of contents, and a small `.pak` sits next to them for
loose files.

**Chunking** splits those containers into several numbered sets. By default everything goes into
chunk 0. Chunk IDs from rules and labels only take effect when **Project Settings → Packaging →
Generate Chunks** is on (`bGenerateChunks`, `False` in `BaseGame.ini`). Chunk 0 acts as the parent
of the others, so anything already in chunk 0 isn't duplicated into another chunk.

**Chunks are about files, not memory.** Putting an asset in chunk 3 changes which download it
ships in. It doesn't change when it loads or how much memory it uses. You need chunks for mobile
download size limits, console streaming installs ("start playing at 30%"), DLC and language packs.
A student project almost certainly doesn't need them. What matters is knowing they exist.

### Advanced: overriding the Asset Manager

You can subclass `UAssetManager` and set it under **Project Settings → Engine → General Settings →
Asset Manager Class Name**. `StartInitialLoading` and `FinishInitialLoading` let you load things at
startup. `ModifyCook` and `ModifyCookReferences` let you add or remove packages from a cook. Its
signature has changed between engine versions, so copy it from `AssetManager.h` rather than from
a tutorial. Lyra's `ULyraAssetManager` is a good real-world subclass to read.

## Tools

- **Reference Viewer**, **Size Map** and **Asset Audit** show what references what, how much it all
  weighs, and a sortable list of assets with their chunks and primary asset IDs. Switch Asset
  Audit to a cooked platform to see cooked sizes, which are the best quick estimate of memory. All
  three are covered in [For Designers and Artists](./for_designers_and_artists.md).
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
