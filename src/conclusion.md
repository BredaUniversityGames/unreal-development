# Conclusion

That's the round trip. A project created from a text file, configured with C# build rules, compiled from the command line, cooked, and run as a standalone game, with no solution file involved at any point.

Along the way you saw what each piece does: the `.uproject` file that makes a folder a project, the `.Target.cs` and `.Build.cs` files that tell UnrealBuildTool what to compile, the primary module that gives the engine an entry point, and the batch files that turn six long commands into two short ones. If a build breaks now, you know which of those to go and look at.

Happy coding!

Where to go from here:

- **[Iteration Speed](./iteration_speed.md)**: Live Coding, driving your own code from the console,
  and the other things that shorten the gap between writing a line and seeing it run.
- **[Profiling](./profiling.md)**: how to find out why something is slow instead of guessing,
  using Unreal Insights and the GPU capture tools.
- **[For Designers and Artists](./for_designers_and_artists.md)**: the same project structure this
  book just built, seen from the Content Browser side. Worth sending to the people you work with.

PS: I've also added some additional workflow tools that I had no idea where to put, in the appendix
section, or you could just click [here](./additional_workflow_items.md).