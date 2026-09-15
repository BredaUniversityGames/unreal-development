# Physics

This page exists because the same finding turned up in **both** of the games profiled in these
talks: two unrelated projects, different studios, different genres, neither of which is a physics
game. When something appears independently like that, it is not a mistake those teams made. It is a
default doing what defaults do.

## The default

**Every static mesh in Unreal gets both simple and complex collision.** If you never explicitly set
up complex collision, it is generated from the top of the mesh's LOD chain, or, if there is no LOD
chain, from the mesh itself.

That default exists for a good reason: it means things collide sensibly for somebody learning the
engine, without them having to know what collision complexity is. The cost is that everybody else
inherits it silently, forever, across every asset anyone ever imports.

Here is what matters conceptually, because it is the bit that makes the fix obvious:

- **Simple collision** is what is used when objects move and collide against each other. Boxes,
  spheres, capsules, convex hulls.
- **Complex collision** is used for **scene queries**: ray casts, sweeps, traces. Per-triangle
  accuracy.

If you are not tracing against a mesh at triangle precision, you are paying for complex collision and
getting nothing. And "paying" is not theoretical: it costs memory to store, it costs load time to
build, and it makes every query against it slower.

### The fix

**Project Settings > Physics > Default Shape Complexity** → *Use Simple Collision As Complex*.

One setting. In one of the profiled games this was worth roughly 10 seconds of load time and removed
6,382 physics bodies being simulated every frame in a cove-management game with no physics gameplay
at all.

Then, with the pathological case gone, you can actually see and optimise your simple collision, which
is the thing that was being drowned out.

### The Nanite caveat

Complex collision is built from the LOD chain. With Nanite, it is easy to end up without a
conventional LOD chain, and then complex collision is generated from the **full Nanite mesh**, which
is catastrophic for both memory and query cost. If you use Nanite, still set a LOD count and let the
fallback meshes generate.

## Chaos Visual Debugger

**Tools > Debug > Chaos Visual Debugger.**

This shows you the complete physics scene state and every physics query, per frame, for a recording.
It is the X-ray view of the world your game is actually simulating, as opposed to the world you can
see. In the talk it was put to a room of about 300 developers, of whom roughly seven had ever
opened it.

It works on packaged development builds too, via console commands that write a file you load
afterwards; you do not need the editor.

One setting to change immediately: there are show flags for **simple** and **complex** geometry and
both are on by default, which makes your first look confusing. Turn off whichever one you are not
currently investigating.

### What "wrong" looks like

These are all real, from the two captures, and they are worth listing because they are so much more
absurd than anything you would think to look for:

- Every individual **leaf** on a tree with its own collision.
- The **wooden dowels** modelled into a wall, one collision shape per decorative nail.
- **Moss**, with per-triangle complex collision.
- A **crate of fruit** where every apple, every stem, every onion and every cabbage had complex
  collision. Nested, several layers deep.

That last one is common enough that it has become a running joke among the people who do this for a
living. They call it the fruit basket, and they see it in nearly every project. The replacement is
one box. The players cannot tell.

Meanwhile, in the same scene, the trees' *trunks* were capsules, which is exactly right. Nobody was
being careless. The difference is that somebody made a decision about the trunk and nobody made a
decision about the leaves, so the leaves got the default.

## Overlaps on non-static geometry

A subtler one. When non-static geometry streams in, it checks for overlaps by default.

The engine does not need this. It does it so that **you** can listen for overlap events and react to
them. If you are not listening, it is free work being done on your behalf that you never asked for.

Two fixes, in order of preference: mark the object **static** if it never moves, which skips the check
entirely, or turn overlap generation off for the non-static ones.

## Tick groups

Unreal's tick runs in groups: `PrePhysics`, `DuringPhysics`, `PostPhysics`, `PostUpdateWork`. Physics
is kicked off on another thread after `PrePhysics`, and `EndPhysics` is where the game thread waits
for it to finish.

So if you see the game thread idling at `EndPhysics`, you can move some of your `PrePhysics` work to
`DuringPhysics`. It then overlaps with the simulation instead of delaying its start, and you spend
less time waiting.

Inside `DuringPhysics` you **can** read transforms; they give you the values from before this frame's
simulation, which is usually fine. You cannot write them.

But be clear about what this is. To quote the talk directly, it is *"applying a bandage to an
infection"*. You have not made physics cheaper, you have hidden some of the wait behind other work.
The real question is why physics costs that much in the first place, which brings you back to the
top of this page.

## For artists

If you make meshes, the first half of this page is more in your hands than in any programmer's.
Collision complexity is set at import and authoring time, and no amount of engineering downstream
fixes a project where every prop ships with per-triangle collision.

[For Designers and Artists](./for_designers_and_artists.md) covers the content-side tooling from the
same angle.

## Next

[GPU](./profiling_gpu.md), the other half of the frame.
