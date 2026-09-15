# Profiling

Up to this point the book has been about making a project build and run. This section is about
finding out why it is slow, which is a different skill, and one that is mostly about refusing to
guess.

The material here comes from two talks by Epic's technical developer relations team, who profile
other studios' games for a living. What makes them worth transcribing into a book is that they were
not demonstrating on sample projects. They were handed two real, in-development games by studios who
said outright that they had not started optimising yet, and worked through them in public. Every
finding below has a cause, a measurement and a fix attached, which is the only form in which this
kind of knowledge is actually useful.

## The method

One sentence carries the whole section:

> **Never guess. Always investigate. What is slow, why is it slow, and how do you fix it.**

The failure mode this guards against is not laziness, it is plausibility. Performance problems come
with a ready-made story ("it's probably the shadows", "it's the physics", "that mesh looks heavy"),
and the story is frequently wrong in a way that costs a week. In one of the captures below a gap in
the GPU timeline sat right next to something called `OcclusionCullPipe`, which made "it's occlusion"
almost irresistible. It was not occlusion. It was the profiling tools' own overhead.

Once you have a capture in front of you, the reading order is:

1. **Find the big number.** Scroll into it. Find out why it is big.
2. **Find lots of little numbers.** Five hundred things costing a microsecond each is also a big
   number, and it hides better.
3. **Find numbers where there shouldn't be any.** Work happening for something off screen, a system
   running in a game that does not use that system. These are the cheapest wins in the whole
   exercise, because the correct cost is zero.

Then it is mechanical: click the tallest frame, find the cause, fix it, click the next one.

## Profile on hardware that means something

A development workstation is not a console and is not a player's PC. This matters more now than it
used to: the render thread was parallelised in **5.4** and RHI translation in **5.5**, so Unreal
scales across cores, and a machine with 64 of them will hide problems that a six-core
machine will not. Half of the CPUs in the Steam hardware survey have six cores or fewer.

Options, roughly in order of preference:

- **Real devkits:** fixed hardware, consistent numbers, comparable between people.
- **PC handhelds used as consoles:** a Steam Deck or ROG Ally is a fixed, known, modest hardware
  target that you can buy without a publishing agreement. Several studios use them as a poor
  developer's devkit, and it works well.
- **Deliberately different machines per developer:** if everyone tests daily on different hardware
  you get broad coverage of GPUs, CPUs and drivers for free, at the cost of never having two
  comparable numbers.
- **`-corelimit=14`** to make a workstation behave more like current-gen hardware. Not a substitute
  for the real thing, but it is one command-line flag.

Whatever you pick, the point is *consistency*. A number you cannot compare to last week's number is
not a measurement.

## Launch parameters

These all go on the command line of the build you are profiling. If the build was distributed through
Steam, you do not need a special build to set them: right-click the game, **Properties > General >
Launch Options**. (Distributing PC dev builds through a private Steam beta branch is a reasonable
trick in general: you have already paid the fee, and you get the content delivery network.)

### The one that matters

```
-trace=default,task
```

`-trace` turns on tracing to Unreal Insights. On its own, with no argument, it uses a default set of
channels and that is enough to start with. Everything else on this page is refinement.

**A note on the default channel set:** it has changed across releases, and Epic's own documentation
and their own conference talks currently disagree about what it contains. So this book is not going
to print a list. Check `TraceAuxiliary.cpp` in the engine source, or simply start a trace and look at
what the Insights front end reports. This is the first of several places in this section where the
source is faster and more trustworthy than documentation about the source.

The Insights front end is `UnrealInsights.exe`, in `{UE-Binaries}`. No shortcut is created for you;
make one. If it is already running when a traced game starts, it connects automatically and the
session shows up in its list.

### Channels worth adding

**`task`** draws arrows showing which task a thread is blocked on, and what that task was waiting
for in turn. Without it, "waiting for task" is where profiling stops and guessing begins; with it you
follow the chain. It carries overhead, which is why it is not on by default, but by the time you are
taking a capture, something is usually already wrong, so pay it.

**`counters`** adds per-frame counters, and a surprising amount of value for one word:

- Platform memory: total and available physical for the system, used physical for the process.
- Primitive component count and draw calls after CPU culling, plus the GPU adapter name.
- Chaos solver body counts: kinematic, dynamic, total. The number of physics shapes actually active
  each frame, which the [Physics](./profiling_physics.md) page leans on heavily.
- Scene lights.
- **UObject count:** added in 5.8, by one of the speakers, in about two lines of code.
  A useful reminder that the profiler is engine code you are allowed to extend when it does not tell
  you what you need.

**`loadtime`** turns on the Loading track in the timing view (press **L**) and adds an **Asset
Loading** tab. It shows CPU-side package processing, not pak or IO Store reads, which is the right
thing to care about because the CPU-side work is where the extra processing happens. The **Requests**
table gives each load request's duration *including everything it dragged in*, and the package count.
That table is how you discover that opening a main menu loaded 8,300 packages.

**`stats`** forwards Unreal's whole `stat` system into Insights. Pure overhead if you do not need
it, but it carries video memory counters and navigation memory. Concretely: it is how one team found
100 MB going to navigation data in a game with no navigation.

### Quality-of-life flags

**`-statnamedevents`** traces additional named events, so Insights shows you far more function
names. It is not free: measured at roughly **20% overhead** on one of the games profiled. At that
cost it is the wrong flag for "are we hitting our frame budget?" and the right flag for
"why aren't we?"

The payoff looks like this. Without it, an animation Blueprint shows up as a 100-microsecond block:
mildly annoying, maybe just what animation Blueprints cost. With it, that block resolves into **16
ray casts into the physics scene, per character, per frame**, all of them tracing straight down at the
ground and most of them redundant. That is not a performance mystery any more, it is a bug with an
obvious fix.

**`-execcmds="stat unitgraph,stat fps"`** runs console commands after engine initialisation.
`stat unitgraph` gives a rolling graph of recent frame times in the corner, so you can *see* a hitch
happen instead of staring at a number; `stat fps` adds the frame rate.

**`-noverifygc`**. Development builds spend extra time verifying the garbage collector's
assumptions. That verification is useful, since it catches real mistakes in C++ code, but
it roughly doubles how long garbage collection takes, and every GC then shows up in your capture as a
hitch. You click it, you investigate it, and it turns out to be the verification. Turn it off when
you are measuring performance rather than correctness. It does not run in shipping builds anyway.

**`-dpcvars="gc.VerifyAssumptionsOnFullPurge=0"`** overrides console variables at launch.

This one deserves an explanation, because you would never guess its use from its name. `dpcvars`
stands for *device profile cvars*, and it is called that purely because the device profile team
happened to be the first group that needed a way to override cvars at startup. The name records who
asked for it, not what it does. What it actually does, and the reason to prefer it over
`-execcmds` for setting cvars, is run **before engine initialisation**, which means it can override
**read-only** cvars that `-execcmds` cannot touch.

That is a small thing, but it is a good illustration of a general rule about this engine: names
frequently describe history rather than intent, and the way you find out what something really does
is to read it.

**`-logcmds="log LogGarbage verbose"`** sets per-category log verbosity. `LogGarbage` at verbose
prints detailed timings for every phase of garbage collection *and* the UObject count, which is
otherwise awkward to get out of a running game.

Honest caveat: verbose logging costs real time, since the text has to be formatted, flushed and
written, and can itself appear as hitches. But it appears in the profiler as logging, so it is
visible rather than mysterious, which is the distinction that matters.

**`-handleensurepercent=0`**. An `ensure()` that fires does a stack walk and writes a dump, which can
take the better part of a second. Insights shows you none of that work; you just get an unexplained
hole in the timeline that no amount of scrolling explains. The default is 100 (handle all of them).
Set it to 0 when you are measuring performance rather than stability.

**`-attachpix`** is required, or PIX will not pick up the process.

## When Insights is not enough

Unreal Insights is **telemetry**. It shows you what somebody instrumented. That is enormously
valuable and also a hard limit: if the slow thing has no trace event around it, Insights will show
you a gap and nothing else.

A **sampling profiler** takes the opposite approach. It needs only symbols for your executable, and
it captures the entire C++ call stack thousands of times per second: everything, instrumented or
not. When you have a hitch that Insights has no explanation for, this is the tool.

- **Superluminal** is paid, inexpensive, and very good.
- **PIX for Windows** is free, from Microsoft, and built for game developers.
- Platform vendors ship their own for consoles, mobile and XR.

Roughly double the overhead, and you can run one alongside Insights so you have both views of the
same moment. Insights gained its own sampling profiler in 5.7.

## Make it repeatable

A measurement you cannot reproduce is an anecdote. Three cheap habits:

- **`BugIt` / `BugItGo`**, introduced in [Iteration Speed](./iteration_speed.md). `BugIt` puts a
  command on your clipboard that returns any player to your exact camera position and rotation. For
  profiling this is how you take the same capture twice.
- **Record your screen alongside the capture.** OBS is free. With `stat unitgraph` visible you can
  line up a spike in the recording with a spike in the trace and see what was actually on screen when
  it happened. Insights can also embed screenshots directly into a trace.
- **Add a console command that jumps to a known point.** If you profile the same cinematic or the
  same encounter repeatedly, spending twenty minutes on a command that skips straight there pays for
  itself within a day.

## The rest of this section

- **[Reading a Capture](./profiling_reading_a_capture.md)**: how to read Unreal Insights without
  bouncing off it, and what real findings look like.
- **[UObjects and Garbage Collection](./profiling_uobjects.md)**: object counts, GC cost, and how to
  find out what is keeping something alive.
- **[Physics](./profiling_physics.md)**: the default that costs almost every project time, and the
  tool that shows it to you.
- **[GPU](./profiling_gpu.md)**: GPU captures, Nanite bins, shadows, translucency and foliage.
