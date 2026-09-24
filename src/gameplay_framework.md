# The Gameplay Framework

Every Unreal game, whether it's a shooter, a puzzle game or a walking simulator, runs on the same
small set of classes. The engine spawns them in a fixed order when you press Play, and each one has
one job: persistence, rules, shared state, per-player state, a body, a brain or a UI. That set of
classes is the **Gameplay Framework**. Once you know which class does which job, you know where
almost any piece of gameplay code belongs, and you can read most Unreal projects.

This chapter goes through the classes roughly in the order they come into existence, then puts them
together. You can use every one from C++, Blueprint or both, and you can leave most of them at
their defaults. The multiplayer parts are there if you need them and cost you nothing if you don't.

Epic's own introduction is the *Introduction to the Gameplay Framework* video on the Epic Developer
Community, and Alex Forsythe's *The Unreal Engine Game Framework: From int main() to BeginPlay*
follows the same path from the engine's side. Both are worth watching after reading this.

## Game Instance: persistence

`UGameInstance` is created when the game starts and destroyed when it quits. It is **not** an actor,
it is not part of any level, and it survives every map change. That makes it the place for data
that must outlive a level: the player's profile, which save slot is loaded, progress across levels.

It is never replicated. The server and every client each have their own Game Instance, and they
know nothing about each other.

The trap is that "survives everything" is convenient, so everything ends up in it, and six weeks
later your Game Instance has three thousand lines and does twelve unrelated jobs. The fix is
**subsystems**. Choose the base class by how long the system should live:

| Subsystem base | Lifetime |
|---|---|
| `UEngineSubsystem` | As long as the engine runs, including in the editor |
| `UEditorSubsystem` | As long as the editor runs |
| `UGameInstanceSubsystem` | As long as the game runs; survives level changes |
| `UWorldSubsystem` | Created and destroyed with each `UWorld` |
| `ULocalPlayerSubsystem` | One per local player, which is what makes split-screen work |

The engine creates, initializes and destroys them for you. You override `Initialize()` and
`Deinitialize()`, and anything can find them:

```cpp
USaveGameSubsystem* Saves = GetGameInstance()->GetSubsystem<USaveGameSubsystem>();
```

No singletons, no manual ownership, and no editing a shared class every time someone adds a system.
The base classes are in `Engine/Source/Runtime/Engine/Public/Subsystems/`.

### Worlds and levels

Underneath the Game Instance sits the world: one `UWorld` per loaded map, made up of a persistent
`ULevel` plus any streamed-in levels, each holding `AActor`s. Travelling to another map destroys the
`UWorld`, and everything in it goes with it: the Game Mode, Game State, Pawns and HUD are all
created again from scratch. Only the Game Instance and its subsystems carry on. (Seamless travel can
carry a Player Controller and Player State across in multiplayer, but don't design around it.)

## Game Mode: the rules

`AGameModeBase` is the first actor spawned when a level loads, and it **only exists on the
server**. It holds the rules: who may join, where players spawn, what happens when someone scores,
when the match is over. On a client, `GetWorld()->GetAuthGameMode()` returns `nullptr`. That isn't
a bug. It's how the framework stops clients from deciding the rules.

There are two classes to choose from:

- **`AGameModeBase`**: lightweight and generic. The right choice for single-player and most
  projects.
- **`AGameMode`**: a subclass for match-based multiplayer, with a built-in match state (waiting
  to start, in progress, finished) and the flow around it.

`StartPlay()` is called once at the start of play, before any actor's `BeginPlay`, which makes it
the place to set up global rules.

### Default classes

The Game Mode decides which classes make up the rest of the framework:

| Property | What it spawns |
|---|---|
| `DefaultPawnClass` | The player's body |
| `PlayerControllerClass` | The brain driving that body |
| `GameStateClass` | Shared, replicated match data |
| `PlayerStateClass` | Per-player, replicated data |
| `HUDClass` | The on-screen UI for each player |
| `SpectatorClass` | The body used while not playing |

Switch Game Mode and all of those change with it. That's why a main menu level and a gameplay
level usually have different Game Modes.

You set the project-wide default in **Project Settings → Maps & Modes**, and override it per level
with **World Settings → GameMode Override**. The default classes are set in your Game Mode's
constructor or Blueprint class defaults.

### How a player gets a body

When a player joins, the Game Mode runs a chain of calls. Each step is virtual, so you can replace
any one of them:

1. `PostLogin`: the player has joined, and their Player Controller exists.
2. `RestartPlayer`: this controller needs a body.
3. `ChoosePlayerStart`: pick an `APlayerStart` in the level. This is where team spawns and spawn
   protection go.
4. `SpawnDefaultPawnFor`: spawn `DefaultPawnClass` at that spot. This is where class selection
   goes.
5. The controller **possesses** the new pawn.

`ChoosePlayerStart` and `SpawnDefaultPawnFor` are `BlueprintNativeEvent`s, so in C++ you override
their `_Implementation` versions. All of it runs on the server. The declarations are in
`Engine/Source/Runtime/Engine/Classes/GameFramework/GameModeBase.h`, and reading `RestartPlayer`
there is a good way to see the whole chain in one place.

## Game State and Player State: shared data

The Game Mode decides the rules. The state actors hold the data those rules act on, and unlike the
Game Mode, **they replicate to every client**.

- **`AGameStateBase`** holds data about the whole game that everyone needs to agree on: team
  scores, the match timer, shared objectives. It also keeps `PlayerArray`, the list of every
  player's Player State, which is what a scoreboard reads from.
- **`APlayerState`** holds data about one player that other players can see: name, score, team,
  ping. It belongs to the player, not the body, so **it survives the Pawn dying and respawning.**

A typical flow for a player dying looks like this. The Pawn's health reaches zero. The Game Mode
decides what that means and updates the Player State (one fewer life). If the result has to survive
a level change, the Game Instance records it. Then the Game Mode restarts the level or returns to
the menu.

Game-wide platform features such as achievements, presence and cloud saves go through the **Online
Subsystem**, which abstracts Steam, Xbox and PlayStation services. Even a single-player game usually
needs some of it.

## Pawn: the body

An `APawn` is anything that can be possessed: a body in the world. Like any actor, it's mostly a
collection of components. The engine gives you two starting points:

- **`ADefaultPawn`**: a sphere collision component, a static mesh and a simple flying movement
  component. Good for prototyping and for spectating.
- **`ACharacter`**: a capsule collision component, a skeletal mesh (so it can be animated) and the
  `UCharacterMovementComponent`, which handles walking, falling, swimming and flying, with network
  prediction built in. It's the usual base for a humanoid player.

Neither comes with a camera. Adding a `UCameraComponent` (usually on a `USpringArmComponent` for
third person) is up to you.

### Collision

Every collidable component has an **object type**, which says what kind of thing it is (`Pawn`,
`WorldStatic`, `PhysicsBody`, ...). It also has a response to each **collision channel**: **block**,
**overlap** or **ignore**. Blocking produces *hit* events, and overlapping produces
*begin/end overlap* events. Traces (line traces and shape sweeps) query the same channels on
demand, for example `Visibility` for a line of sight or `Camera` for a spring arm. You can add
your own object types and trace channels in **Project Settings → Collision**, and presets bundle a
full set of responses under one name. Traces are powerful and cheap one at a time, but a few
thousand per frame is not cheap. [Profiling Physics](./profiling_physics.md) shows what that looks
like in a capture.

### Movement

`UCharacterMovementComponent` is extremely capable, and it's also one of the most complex classes
in the engine. It assumes an upright humanoid with a fixed capsule, it does its own network
prediction outside the rest of the framework, and extending it usually takes a programmer. For
anything that isn't a person walking, look at the **Mover** plugin
(`{UE-Root}/Engine/Plugins/Experimental/Mover`). Mover builds movement from small, swappable
movement modes with rollback-friendly networking. It's still marked **experimental** in 5.8 (check
`IsExperimentalVersion` in `Mover.uplugin`), so weigh that before building a shipping game on it.

## Controller: the brain

An `AController` has no body. It **possesses** a Pawn and drives it. The Pawn is the body, and the
Controller is what decides what the body does. Usually one controller drives one pawn, though games
like RTSes spread one player's intent across many units.

- **`APlayerController`** is the human player. It receives input, owns the camera manager and the
  HUD, and exists on the server and on the owning client only. Other clients never see your
  Player Controller.
- **`AAIController`** is an AI's brain. It runs behaviour trees and perception, and **only exists
  on the server**. Clients see the pawn it drives, never the controller.

The split matters as soon as something dies. In a deathmatch you die, the Game Mode spawns a new
Pawn, and **the same controller possesses it.** A score stored on the Pawn is gone. A score stored
on the controller survives. In multiplayer it belongs on the Player State, because that's the one
other players can see. AI controllers don't get a Player State by default. Set
`bWantsPlayerState` on the `AAIController` if your bots should show up on the scoreboard.

### Possession

`Possess(APawn*)` takes control and `UnPossess()` lets go. `OnPossess` and `OnUnPossess` are the
hooks you override. For a pawn placed in the level, `AutoPossessPlayer` and `AutoPossessAI` decide
who takes it at start.

Getting into a vehicle is just possession: the controller un-possesses the character and possesses
the tank. The same brain and the same player now drive a different body with different rules, and
getting out reverses it. Possession happens on the server, and the controller-pawn link replicates.

### The camera

Each Player Controller spawns an `APlayerCameraManager`, which decides what the player sees. It
looks through a **view target**, which is usually the possessed pawn but doesn't have to be. Death
cams, spectating, security cameras and cutscenes are all `SetViewTargetWithBlend()`, which moves the
view without changing possession. The camera manager also handles blends, camera shakes, fades and
post-processing. In short: possession decides what you drive, and the view target decides what you
see.

### Input

**Enhanced Input** is the input system in UE5. It's a plugin, and it's enabled by default in new
projects. The old Action and Axis mappings in Project Settings still work, but they are being phased
out. Enhanced Input is built from three kinds of asset:

- **Input Actions** (`UInputAction`): something the player can do, with a value type (bool, 1D, 2D
  or 3D axis).
- **Input Mapping Contexts** (`UInputMappingContext`): which keys trigger which actions. You can
  add and remove contexts at runtime, and they stack by priority.
- **Triggers** (pressed, hold, tap, ...) and **modifiers** (dead zone, scale, negate, swizzle) that
  shape the raw input before your code sees it.

Mapping contexts belong to the local player: `UEnhancedInputLocalPlayerSubsystem`, a
`ULocalPlayerSubsystem` from earlier, is where you call `AddMappingContext` and
`RemoveMappingContext`. So split-screen players keep separate bindings. A typical setup adds the
gameplay context in `BeginPlay`, pushes a UI context at a higher priority when a menu opens, and
swaps in a vehicle context when entering a vehicle. The actions are bound in
`SetupPlayerInputComponent`, on either the pawn or the controller.

## HUD: the UI

`AHUD` is the per-player UI owner. It exists only on the owning client and never replicates. In
practice, the UI itself is built with **UMG** widgets, and the HUD class, or the Player Controller,
is where they get created and added to the viewport. It gets its data by reading the state actors,
for example iterating `GameState->PlayerArray` to fill a scoreboard.

## Putting it together

### What happens when you press Play

1. The engine starts and creates the **Game Instance**, which lives until you quit.
2. The map loads: a **`UWorld`** and its **levels**.
3. The **Game Mode** is spawned first, on the server only.
4. The Game Mode spawns the **Game State**, which replicates to everyone.
5. For each player that joins, a **Player Controller** and **Player State** are created.
6. The Game Mode spawns a **Pawn** at a Player Start, and the controller possesses it.
7. `BeginPlay` runs on the actors in the world.

### Who does what

- The **Game Mode** decides what should exist and what the rules are.
- The **Game State** and **Player State** hold what everyone has to agree on.
- The **Controller** decides what the player (or AI) wants.
- The **Pawn** does it in the world.
- The **HUD** shows it.
- The **Game Instance** remembers it after the level is gone.

### Who exists where

| Class | Server | Owning client | Other clients |
|---|:---:|:---:|:---:|
| `UGameInstance` | ✓ (own copy) | ✓ (own copy) | ✓ (own copy) |
| `AGameModeBase` | ✓ | — | — |
| `AGameStateBase` | ✓ | ✓ | ✓ |
| `APlayerController` | ✓ | ✓ | — |
| `APlayerState` | ✓ | ✓ | ✓ |
| `APawn` / `ACharacter` | ✓ | ✓ | ✓ (if it replicates) |
| `AHUD` | — | ✓ | — |
| `AAIController` | ✓ | — | — |

### Authority

In multiplayer, every replicated actor exists as several copies, and exactly one of them is in
charge. `HasAuthority()` asks "am I the copy that decides?" The **server's** copy has
`ROLE_Authority` and is the source of truth. The **owning client's** copy is an
`ROLE_AutonomousProxy` and predicts ahead so input feels immediate. **Every other client's** copy is
a `ROLE_SimulatedProxy` and only shows what it is told.

Rules of thumb:

- The server handles spawning, damage, scoring and possession.
- The client handles input, UI and cosmetic effects.
- To ask the server for something, send a server RPC. To tell clients something, use replication.

In single player you are always the authority, which is why code that ignores all this works
perfectly until the day someone adds multiplayer.

### Finding each other

Each piece can reach the others through the built-in accessors: `GetGameInstance()`,
`GetWorld()->GetAuthGameMode()`, `GetWorld()->GetGameState()`, `GetPlayerState()`,
`GetController()` and `GetPawn()`, plus the `UGameplayStatics` equivalents for Blueprint. Beyond
that, it's up to you: cast to your own subclass, go through an interface, or bind a delegate.
Interfaces and delegates keep the classes decoupled. Casting is simpler, and fine within one module.
