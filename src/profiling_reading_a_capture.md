# Reading a Capture

The first time most people open an Unreal Insights capture, they close it again. It is a wall of
coloured bars with no obvious entry point. The speakers whose talks this section is drawn from call
it "rainbow soup", which is fair.

It is not actually complicated. There are three panels that matter and one interaction that does most
of the work.

## The frames panel

Across the top is a row of vertical bars. Zoom in with the scroll wheel and they resolve into
individual columns.

**Each bar is one frame**, not a fixed slice of time. This is the thing to internalise, because it
is unlike every other timeline you have used. A bar can represent 16 milliseconds or several seconds;
the *height* is the duration. A frame that took a minute produces a bar that shoots off the top.

There are horizontal reference lines:

- **16.66 ms** is the budget for 60 frames per second.
- **33.3 ms** is the budget for 30 frames per second.

Anything above your line is a dropped frame. Hold shift and scroll to zoom out and see the shape of a
whole session; the spikes are where you start.

### Game frames and rendering frames

Insights tracks both, and clicking a bar can select the *rendering* frame when you meant the game
frame, which is why your selection sometimes looks slightly offset from what you expected. Press
**R** to toggle, or right-click and turn rendering frames off entirely while you are working on CPU
problems.

## The timing view

Below is the timing view. Time runs left to right; depth is call nesting. `FEngineLoop::Tick` is at
the bottom and everything else is inside it.

Selecting a frame and walking down gets you from "this frame was slow" to "this frame was slow
*here*" in a few clicks:

```
FEngineLoop::Tick
  └─ WorldTick
       ├─ TickTime          (PrePhysics and DuringPhysics work)
       ├─ BlueprintLatentActions
       └─ GameThreadTickableTime
```

When you find `EndPhysics` mostly consisting of "wait for task", that is the game thread sitting idle
waiting for physics running on another thread. That is a signal to go look at
[Physics](./profiling_physics.md) rather than at anything in this list.

## The one interaction to learn

Select a time range, then **single-click** a function. Insights reports how many times that function
was called within the selection, along with inclusive and exclusive time.

This is the interaction that turns impressions into statements. "The physics looks busy" is not
something a programmer can act on. "You are doing 500 ray casts into the physics scene in a single
frame" is, because somebody can go and find out why, and there is usually an obvious answer.

Double-clicking a function highlights every other instance of it across the timeline, which is how
you find out whether something is one expensive call or a hundred cheap ones spread out.

## What real findings look like

Worked examples from the two games profiled in these talks. They are here for their *shape*, not
their specifics. You will not have these exact bugs, but you will have bugs that look like them.

**Work for things nobody can see.** Character movement components were ticking for a large number of
characters in a scene containing two. The rest were enemies from encounters later in the level, being
simulated in advance. The fix was to not simulate them until they matter, and it was available
because there was already a natural checkpoint where the tutorial ended.

**Doing it for everyone instead of for the ones on screen.** A `SetGender` call was running for every
character in the entire level rather than the two visible. This is the single most common shape of
performance bug in gameplay code, and it is invisible in the source: the function does not look
expensive, it is just being handed a much larger set than anyone intended.

**Doing it twice.** See [UObjects and Garbage Collection](./profiling_uobjects.md) for the best
example of this: nine thousand packages loaded, garbage collected, and loaded again.

**Blowing a time budget you did not know existed.** `UpdateLevelStreaming` adds components to the
world incrementally, with a time budget of a few milliseconds per frame, so that streaming does not
hitch. One frame took 28 ms anyway, because the budget is only checked every few
components, so a *single* component that takes 26 milliseconds sails straight past it. In this case a
single instanced static mesh component was creating physics state for an enormous number of
instances.

**Forcing async work to be synchronous.** Skeletal mesh animation is normally evaluated on a worker
thread. Certain operations, such as spawning an actor or changing animation, force it to be evaluated
immediately instead, so that the result is visible in the same frame you asked for it. You usually do
not need that immediacy, particularly during a fade or a transition, and it is a setting rather than
a rewrite. If you hit this, search the engine source for where animation evaluation decides to run
immediately; the flag is near it and the name changes between versions.

**Using the wrong LOD because nobody said not to.** MetaHuman LOD0 is built for virtual production
and film work. For games, Epic's own recommendation is LOD1 at most, dropping aggressively from
there, giving you fewer facial bones and much less evaluation. Characters occupying a
two-hundred-and-fifty-sixth of the screen do not need the Hollywood head.

**UI that is more expensive than it looks.** Widget components redraw much like drawing a material to
a render target, and 2,800 of them were ticking. There is an *Automatic Tick Mode* setting if you
need many of them. Separately, the habit carried over from UE4 of putting a canvas panel at
the root of every user widget costs layout work on every one of them, for no benefit in the common
case.

## Next

[UObjects and Garbage Collection](./profiling_uobjects.md), the counting problem that sits behind a
surprising number of hitches.
