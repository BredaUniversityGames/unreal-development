# For Designers and Artists

Every other chapter in this book is written for the person *building* the structure: the modules,
the dependencies, the build rules. This one is for everybody who then has to work inside it.

You will probably never open a `.Build.cs` file. But you cross the boundaries those files define
several times a day, usually by doing something entirely reasonable: dragging one asset into another,
picking a mesh from a dropdown, referencing a material from a Blueprint. Each of those creates a
dependency, and dependencies have costs that are invisible at the moment you create them and very
visible three months later when the main menu takes twelve seconds to open.

The good news is that Unreal ships excellent tools for *seeing* this, and almost nobody opens them.
So we will start with the tools, because those are what you can use on Monday, and get to the
concepts afterwards, each one attached to the tool that makes it visible.

## Reference Viewer

**Right-click any asset in the Content Browser > Reference Viewer.**

You get a graph. Your asset sits in the middle; everything that *references it* is on one side,
everything *it depends on* is on the other. Between them those two sides answer the questions that
come up most often: "is anything still using this?" and "what am I dragging along when I use
this?"

The controls that make it actually usable live in the search panel in the upper left:

- **Search References Depth** and **Search Dependencies Depth** control how many hops out to follow
  in each direction. Start shallow. A depth of 1 tells you the immediate story; going deep on a busy
  asset produces a hairball nobody can read.
- **Search Breadth Limit** is the maximum number of references shown per column. When it kicks in
  you get an **overflow node**; double-click it to expand.
- **Collection Filter** restricts the graph to a single Collection. This is the main reason to keep
  Collections around: scope a reference graph to "the assets in this level" and it becomes legible.
- **Filters** work by asset type, the same way the Content Browser filters do.

Nodes are colour-coded by asset type and carry a name, type and thumbnail. Assets filtered out go
semi-transparent rather than disappearing, so the chain stays readable.

Keyboard shortcuts toggle which *kinds* of reference are drawn. Learn these, because the difference
between a hard and a soft reference is the difference between an asset loading and not loading:

| Key | Reference type |
|---|---|
| **S** | Soft references |
| **H** | Hard references |
| **E** | Editor-only references |
| **M** | Management references (Primary Asset IDs) |
| **N** | Name references (searchable names) |
| **C** | C++ package references |
| **V** | Toggle compact / full node view |

## Size Map

**Right-click an asset > Size Map.**

A treemap of that asset *and everything it drags in with it*, sized by how much space each thing
takes. The Reference Viewer tells you the shape of the problem; the Size Map tells you how much it
costs, and it does it in a single glance that you do not need any training to interpret. If a widget
you thought was small opens a Size Map showing half your character art, you have found something.

`Ctrl+E` opens whatever you are hovering over.

One honest caveat: in the editor, things are loaded that a packaged game would not load, so the
editor's Size Map can paint a gloomier picture than shipping reality. Use it to compare assets and to
spot the outrageous cases, which is what it is good at, rather than as an exact memory figure.

## Asset Audit

**Tools > Audit > Asset Disk Size.**

Where the previous two tools look at one asset, this one looks at many, in a sortable table. Two
columns carry the entire lesson of this chapter:

- **Exclusive Disk Kb** is what this asset costs by itself.
- **Disk Kb** is what this asset costs *including everything it manages*.

When those two numbers are close, the asset is self-contained. When Disk Kb is enormously larger than
Exclusive Disk Kb, that asset is a doorway into the rest of the project. Sort by the gap between them
and you have a prioritised list of what to look at, produced in about four seconds.

The **Chunk** column shows where each asset will land when the project is cooked, which is how you
check that your packaging split is doing what you intended. There is more on what sits behind this in
Epic's [Asset Management documentation](https://dev.epicgames.com/documentation/en-us/unreal-engine/asset-management-in-unreal-engine).

## Plugin Reference Viewer

The same idea, one level up: which plugins depend on which. It needs enabling in **Edit > Plugins**
first.

This stops being academic the moment a project uses Game Feature plugins, where content is *supposed*
to stay inside its own plugin boundary and the whole design falls apart quietly if it does not. On a
large project the dependency web is too big to keep track of from memory, and seeing it drawn is
usually the point at which someone says "wait, why does that depend on that?"

Worth knowing alongside it: Unreal's asset validation tooling can **refuse** a drag-and-drop that
would create a reference across a plugin boundary the target does not depend on, and tell you what to
do instead. That turns the rule below from advice into a guardrail, which is the only form of this
rule that survives contact with a deadline.

## Now the concepts

### Content, modules, plugins

The chapters before this one built a structure. In terms you can see from the Content Browser:

- A **module** is a unit of *code*. It compiles to one DLL. Your project has at least one.
- A **plugin** is a self-contained bundle that can hold its own modules *and its own content*. Think
  of it as a sub-project: its own `Source` folder, its own `Content` folder, its own boundary.
- **Content** is everything in the Content Browser: assets, not code.

The boundaries between these are real. Code in one module can only use another module if the
programmer declared that dependency in a `.Build.cs` file. Content, by contrast, will happily
reference across boundaries the moment you drag something, with nothing asking whether that
dependency was intended. That asymmetry is the whole reason this chapter exists: the code side has an
enforcement mechanism and the content side mostly does not.

### Hard and soft references

This is the concept to take away if you take away one.

A **hard reference** is resolved at load time. When Unreal loads an asset, it follows every hard
reference it holds and loads those too, and then follows *their* hard references, recursively,
before anything runs.

Follow that through. Your main menu widget hard-references the player character, so it can show a
preview. The player character hard-references its weapon. The weapon hard-references its impact
effect. The impact effect hard-references a set of destruction meshes. Nobody did anything
unreasonable at any single step. But opening the main menu now loads the destruction meshes.

This pattern has a name in profiling circles, *reference hell*, and it is routinely the largest
single cause of slow startup in real projects. In one of the games profiled in the
[Profiling](./profiling_uobjects.md) chapter, showing the main menu pulled in 8,300 packages and
around 2.5 GB, because a globals data asset sat in the chain.

A **soft reference** stores the *path* to an asset instead of the asset. Nothing loads until
something asks for it. In C++ these are `TSoftObjectPtr` and `TSoftClassPtr`; in Blueprint you hold a
Soft Object Reference and use **Async Load Asset** when you need the thing.

The trade is real and worth stating plainly: soft references mean the asset is not there the instant
you ask for it, so you need to handle the loading: a frame of delay, a placeholder, a loading state.
That is why the answer is not "make everything soft". The answer is that things which are *always*
needed can be hard, and things which are *sometimes* needed should be soft.

Size Map is how you see which one you have. That is the pairing to remember: the concept is
invisible, and the tool makes it visible.

### Which way dependencies should point

The rule programmers apply to modules applies just as well to content:

> The general thing must not reference the specific thing.

Your generic "door" Blueprint should not reference the boss room's special door. Your shared UI
widget should not reference a particular level's HUD. A shared material should not reference one
character's texture.

The reason is not tidiness. It is that a reference is a loading instruction. When the general thing
references the specific thing, everyone who uses the general thing pays for the specific thing, and
because the general thing is *general*, that means everyone.

## When you find one

You opened a Size Map, found something enormous behind something small. Now what?

1. **Make the reference soft**, if the thing is only sometimes needed. Cheapest fix, and usually the
   right one for optional or situational content.
2. **Move the asset**, if it is on the wrong side of a boundary, such as a shared folder holding
   something that only one feature uses.
3. **Split the asset**, if a Blueprint or data asset has become a junction box that everything routes
   through. The globals-data-asset pattern above is exactly this: it is convenient, and it welds
   everything to everything.
4. **Hand it to a programmer**, if the reference is created in C++ or if breaking it means changing
   how something is loaded. Bring the Size Map screenshot; "this widget loads 800 MB and here is the
   chain" is a bug report someone can act on immediately.

## Where to go next

- [Profiling: GPU](./profiling_gpu.md) covers what the things you make cost at render time: Nanite
  shading bins and material instance reuse, overlapping shadow-casting lights, translucency settings,
  Nanite foliage. Written with the same "here is the mechanism" approach as this chapter.
- [Profiling: Physics](./profiling_physics.md) matters because collision defaults are an
  art-pipeline decision more than a programming one, and the default is expensive. This is probably
  the highest-value twenty minutes in the book for an environment artist.
