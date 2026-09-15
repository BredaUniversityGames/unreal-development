# UObjects and Garbage Collection

Unreal manages the lifetime of anything deriving from `UObject` for you. Periodically it walks
everything reachable from a set of roots, works out what is no longer referenced, and destroys it.

That walk costs time proportional to how many objects exist. So the number of `UObject`s alive in
your game is not an abstract statistic. It is a direct input to how long your garbage collection
pauses are. This page is about finding out what that number is, what it should be, and what to do
when it is wrong.

One detail up front, because it explains a lot of hitches: when Unreal changes map it runs a **full
purge** garbage collection. A full purge is *not* time-sliced. Normal incremental collection spreads
its work across frames; a full purge does all of it at once, which is why the transition between
levels is where GC cost shows up as a single visible stall.

## Finding out what you have

`obj list` is the command, and it is one of the most useful in the engine.

```
obj list
```

By default it lists classes with their object counts and sizes, **sorted by size**. That is the right
default when you are chasing memory. When you are chasing garbage collection time you care about
*count*, not size:

```
obj list -countsort
```

Once a class looks suspicious, list its instances by full object name:

```
obj list class=OverlaySlot
```

And for feeding into anything else:

```
obj list -csv
obj list -all
```

In 5.8 there is also an object trace snapshot, which produces a dedicated Insights tab you can
filter, sort and group rather than parsing console output.

## Why won't this be collected?

The other essential command:

```
obj refs shortest name=/Game/UI/WBP_MainMenu.WBP_MainMenu_C
```

This prints the **reference chains keeping that object alive**, sorted shortest first. It is the
direct answer to "I destroyed this, why is it still here?", and it is the single best tool for
tracking down a leak, because a `UObject` leak in Unreal is almost never a missing `delete`, it is
something still holding a reference that nobody remembers adding.

This is the same question the Reference Viewer answers for content, from the other side. If you are
working with designers or artists, [For Designers and Artists](./for_designers_and_artists.md)
describes the graphical version of this, and it is often easier to hand someone a picture than a
console dump.

## How many is too many?

These are the rules of thumb used by Epic's developer relations team when they look at a capture.
They are experience, not policy, and they depend on your game and your target hardware, but they are
far better than having no reference point at all.

| UObject count | Reaction |
|---|---|
| ~100,000 | Good. Genuinely impressive. |
| ~200,000 | Fine. You know what you are doing. |
| 300,000 to 700,000 | Not great, not terrible. Probably not your biggest problem. |
| ~1,000,000 | Something is structurally wrong. |
| ~2,000,000 | The engine stops. |

That last row is not a soft limit. The `UObject` table is **preallocated**, by default with room for
**2,162,688** entries, and it is not grown at runtime. Run out and you get a fatal error:
`Maximum number of UObjects (2162688) exceeded`.

You *can* raise it, via `gc.MaxObjectsInGame` and `gc.MaxObjectsInEditor` under
`[/Script/Engine.GarbageCollectionSettings]`. Studios do. The advice from the people who get called
in afterwards is blunt and worth repeating: **raising the limit is not a fix, it is switching off the
smoke alarm.** If you are near two million objects, the collection pauses were already hurting long
before the crash. The limit is doing its job by telling you.

## Reading garbage collection in the log

With `-logcmds="log LogGarbage verbose"` you get per-phase timings. The phases, and what a bad one
tells you:

- **Reachability analysis** is the graph walk. It scales with object count *and* with how deep the
  reference chains are. A surprisingly fast one usually means chains are terminating early, which is
  good news.
- **Unhashing unreachable objects** removes objects from the engine's lookup tables. Observed at
  88 ms in one capture. Worth understanding rather than resenting: those hash tables are exactly what
  makes things like `GetAllActorsOfClass` fast. You pay on the way out for speed on the way in.
- **Purging** is the actual destruction. Observed at 212 ms for 81,000 objects.

In a late-game save from one of the profiled games, collection cost roughly 30 ms of reachability
analysis plus about 2.1 seconds of destruction spread across many frames. Spread out, yes, but still
around 2 ms taken out of every frame's budget, continuously, for work that produced nothing.

The log also tells you *how many* objects were collected. In that same game about 40,000 objects were
being created and thrown away every minute, roughly 11 per second churning through allocation and
destruction. Once you know that, `obj list` tells you what class they are, and the question becomes
whether they need to be `UObject`s at all, or could be pooled and reused.

## Worked example: reference hell

The chain, from an actual capture:

```
MainMenuContainer  →  GameMode_MainMenu  →  GameState_MainMenu  →  DA_UIGlobals  →  (most of the game)
```

A globals data asset, referenced by the game state, holding references to a large amount of content.
Showing the main menu therefore pulled in **8,300 packages, around 2.5 GB, taking five seconds**.

Nobody made a bad decision here. A globals data asset is a convenient, obvious thing to make. But
because it is referenced by something that always loads, everything it touches always loads too.

The fix is not a heroic refactor: load those assets when you enter the game rather than when you show
the menu. Total load time across both is unchanged, but the player reaches the main menu in a
fraction of the time, and players who pick "Load Game" never pay for the rest at all.

## Worked example: loading everything twice

This one is worth walking through slowly, because the cause is counter-intuitive and the
symptom is enormous.

- The main menu loaded **11,000 packages**.
- Loading into the game loaded **16,000 more**.
- Of those 16,000, **9,000 had already been loaded** while showing the main menu.

What happened: changing map runs a full purge. The previous map, the previous game state and
everything only they referenced became unreachable, so garbage collection dutifully destroyed all of
it. Then the new map loaded, needed the same 9,000 packages, and loaded them again.

So the project paid to load 9,000 packages, paid to garbage collect them, and paid to load them a
second time. The purge alone was destroying 81,000 objects.

The fix is a one-line change in either direction: hold a reference so the assets survive the
transition, or better still, stop referencing the globals asset from the menu so they never load
there to begin with. Which of those is right depends on whether you need them in the menu at all.

Note how this connects to the previous section. Both problems have the same root: one convenient
shared asset sitting in a chain that always loads. The hard-reference discussion in
[For Designers and Artists](./for_designers_and_artists.md) is the same lesson arriving from the
content side.

## Worked example: UI as an object factory

From a late-game save with 761,000 `UObject`s, well into "something is wrong" territory, roughly
half were UI objects:

- 132,000 `OverlaySlot`
- 69,000 `Overlay`
- 65,000 `Image`

The cause was structural: every widget structure had every possible child widget and button
pre-allocated, whether or not it was ever shown. One widget type, multiplied by 302 buttons, produced
104,000 overlays by itself, as many objects as a whole healthy game might have.

The fix took a day: construct UI objects when they are needed, pool them, and reuse them. Which is
the encouraging part of this page. Very large `UObject` counts usually come from one or two
mechanical patterns rather than from a thousand small mistakes, and mechanical patterns can be fixed
mechanically.

## Next

[Physics](./profiling_physics.md), a default setting that costs nearly every project real time.
