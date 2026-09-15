# GPU

CPU profiling asks "what is the game thread doing". GPU profiling asks "what is the renderer being
asked to draw, and why is it expensive". The tools are different, the failure modes are different,
and, importantly for this book, most of the causes are created by people who do not write C++.
[For Designers and Artists](./for_designers_and_artists.md) is the companion to this page.

## Capture tools

A GPU capture records a **single frame**: every resource and every instruction the GPU needed to
produce it. You can then replay it and inspect it in enormous detail.

| Platform | Tool |
|---|---|
| Windows | PIX for Windows, RenderDoc |
| Xbox | PIX |
| PlayStation | Razor |
| Android | Nvidia Insight |

Two practical notes. Add **`-attachpix`** to your command line or PIX will not attach. And a capture
analysed on a different GPU than it was taken on will warn you. Take that seriously, because the
timings will not be comparable.

Since **5.6** there is also GPU Profiler 2, which unifies GPU statistics across PIX, Razor, Insights
and `stat GPU`, and shows the compute pipe inside Unreal Insights. That is a significant
quality-of-life change: you can see individual draw calls and how long they took in Insights, without
taking a GPU capture at all.

## Three cvars, and what happens without them

```
r.RHISetGPUCaptureOptions 1
r.DynamicRes.TestScreenPercentage 75
r.RDG.AsyncCompute 0
```

**`r.RHISetGPUCaptureOptions 1`** turns on draw events, material draw events and `r.RDG.Events 3`, so
that passes and materials have readable names instead of being anonymous. It is the GPU equivalent of
`-statnamedevents` and carries the same trade: more information, more overhead. On 5.4 and earlier
you also need `r.Nanite.ShowMeshDrawEvents 1` to get names for Nanite shading bins.

**`r.DynamicRes.TestScreenPercentage 75`** is the one that stops a capture from lying to you, and it
takes a paragraph to explain why.

With dynamic resolution enabled, the renderer adjusts internal resolution to hit your frame target.
Which means that when you profile, you will always measure close to your frame target: the cost
does not show up in milliseconds, it shows up in a smaller back buffer. One of the speakers describes
going through an entire round of captures, concluding the game was comfortably at 30 fps everywhere,
and only then remembering dynamic resolution was on. Pin the screen percentage and you measure real
milliseconds again. 75% is a reasonable target to develop against, leaving dynamic resolution as
headroom for when things get busy rather than as a permanent crutch.

**`r.RDG.AsyncCompute 0`** moves async compute work onto the main pipe. Async compute jobs are not
captured in GPU captures, so leaving this on gives you a picture with holes in it. Turn it off while
profiling so nothing can hide.

## A debugging story

This is the most instructive thing in either talk, and it is not a fact about the renderer. It is a
demonstration of method.

A gap appeared in the GPU timeline, between HZB and the base pass. Nothing scheduled, apparently
nothing happening.

- It was not visible in `stat GPU`.
- It was not visible in PIX, because PIX replays the events submitted to the GPU and this was not one.
- It **was** visible in draw thread time.
- It only reproduced on some machines, and more readily on the weaker one.

Scrolling down through the threads in Insights eventually found it, labelled `OcclusionCullPipe`.
At which point the obvious conclusion is "it's an occlusion problem", and it would be very easy to
spend a day there.

It was not an occlusion problem. The full label read *waiting for GPU occlusion queries*: the render
thread was waiting, not doing occlusion work. Disabling occlusion queries confirmed it: draw time
dropped, GPU time rose, and the gap simply moved.

The real cause was two things stacked. First, the RHI thread had a great deal of work translating
Nanite shading bins. Second, and this is the uncomfortable part, **the capture options turned on to
investigate the problem were themselves adding overhead to that thread**. The tools were part of what
was being measured.

Two things to take from this. The label nearest your problem is not the same as its cause. And your
profiler is not a neutral observer, which is why this section keeps telling you what each flag costs.

## Nanite raster and shading bins

Bins are Nanite's version of draw calls. They are not per unique mesh. **Every material instance
present in the scene needs a shading bin**, and each one costs a minimum of around 3 microseconds on
the GPU.

Observed in one capture: about 2,100 raster bins and 4,000 shading bins. A healthy figure is nearer
**50 raster bins and about 1,000 shading bins**. (Rules of thumb from the people who look at these
professionally, not official limits.)

Since the count tracks material instances, the fix is to need fewer of them:

- Keep a **fixed material topology** so instances can actually be reused.
- Vary appearance with **custom primitive data**, **material parameter collections**, and
  **per-instance custom data** instead of creating another instance.
- **Material layers are not Photoshop layers, and not Niagara modules.** Used as though they were,
  stacking a new layer whenever you want a variation, they generate material permutations, and the
  permutations become bins.

In the game where this showed up, the bins came from crowd characters using a large number of
material instance dynamics to drive animation state.

Note that landscape materials are a different case: they compile per component with that
component's layer weights, so you cannot deduplicate them the same way. There, the advice is to set
the landscape up with runtime virtual textures and do as little work as possible after sampling. The
landscape covers a lot of pixels, so per-pixel cost is multiplied by a lot.

## Translucency and depth of field

**Translucent materials default to rendering After DOF.** After-DOF translucency renders at full
resolution, even when dynamic resolution has scaled everything else down.

The example was chimney smoke. One particle system, nearly a millisecond, entirely because it was
rendering more pixels than anything around it for no reason anybody had chosen.

Chimney smoke does not need to be after depth of field. Most translucency does not. Audit your
translucent materials and move what you can to Before DOF; it is a per-material checkbox and one of
the highest ratios of saving to effort on this page.

## Overlapping lights

If you see cost in **virtual shadow map projection mask bits**, that number is not telling you what
its name suggests. It tracks **the number of shadow-casting lights affecting each pixel**, so it is
where overlapping lights show up, under a name that gives you no hint of that.

To find the culprits, PIX will re-run a vertex shader for you: click into the draw call, into the
resources, and run the spotlight's vertex shader to draw its cone. Normal lights look normal;
oversized ones are then obvious.

In the game profiled, these turned out to be fake bounce lights approximating global illumination,
which did not need to cast shadows at all. Worth checking:

- The **light complexity view mode**, the cheap first look.
- **Lighting channels**, to constrain what a light affects.
- **Attenuation radii**, kept to something the light actually reaches.

## Shadows and Nanite

Non-Nanite shadow cost concentrated in small meshes with very large bounds, for example a single
instanced static mesh component holding two instances placed far apart, giving it bounds spanning
everything in between.

The broader guidance: if it can be Nanite, it should be Nanite. Nanite has a fixed cost, and the
more of your scene uses it, the more that fixed cost is amortised, and it behaves better with Lumen
and virtual shadow maps.

### Nanite foliage

Foliage is the exception that needs care, and the mechanism is worth understanding because it
explains an otherwise baffling cost.

Opaque meshes eventually fall back to **fixed-function rasterisation**, which is very fast. Masked
materials **never can**, because masking is itself programmable rasterisation. So a masked leaf
material stays on the expensive path at every distance, forever.

The approach that has worked for studios who have solved this:

1. **Crop the masked material tightly** around the actual leaf shape, so less is being masked away.
2. Use **Pixel Programmable Distance** to switch to an **opaque** version of the leaf beyond a
   distance where nobody can tell the difference. At range, a roughly leaf-shaped opaque polygon and
   a perfectly masked leaf look identical.
3. Set **WPO Disable Distance**, and keep world position offset cheap where it is active: a panning
   texture with a vertex alpha, rather than per-vertex multi-joint transforms.

## Numbers where there shouldn't be any

The third reading pass from [the method](./profiling.md): work happening that should not be happening
at all.

In the timeline, most of the frame was normal draw dispatches, and then a long grey region made of
many tiny draws, filed under **visibility commands**. Which sounds like occlusion, and is not:
Niagara GPU compute dispatches are part of that pass.

What was actually happening: the water system had around 96 Niagara systems reading water height back
from the GPU to the CPU to drive other effects. Two milliseconds, every frame, for **the ocean around
an island that was not on screen**.

Fixes, in increasing order of ambition:

- The **significance manager**, so distant systems do less or nothing.
- **Niagara data channels**, so one system serves many effects rather than one system per effect.
- **World Partition**, to unload cells nowhere near the camera. And a useful point from the talk: the
  team had dismissed World Partition because only *one* level had grown large enough to need it, and
  converting the whole game seemed disproportionate. It can be enabled per level.

A second example of the same shape: Nanite programmable rasterisation running in the **custom depth**
pass, for an outline shader that did not actually need anything rendered into custom depth. Turning
it off cost nothing and saved real time.

## Back to the start

Every item on this page was found the same way: look at the big number, look at the many small
numbers, look at the numbers that should not exist. None of it required knowing in advance what was
wrong. [The method](./profiling.md) does not ask you to be right, only to keep looking.
