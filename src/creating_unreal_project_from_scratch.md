#Alternative Approach

Let's peek behind the scenes to understand how project files and build processes interconnect. Instead of diving straight into the complexities, let me guide you through setting up a new project in Unreal Engine from scratch.

Begin by carving out a cozy space for your project on your disk drive. When it comes to naming your project, simplicity reigns supreme. This name isn't etched in stone as part of your public-facing brand—it merely serves as a convenient identifier. A short codename suffices.

- Set up a folder where we'll work our magic.
- Give your project a name.
    - Again, keep it simple, nothing too crazy.

We're gonna start with a .uproject file to lay down the basics and we'll need a Source folder for all our C++ goodness. Inside, we'll organize our code into different modules. Let's call the main one **{projectname}{modulename}**, where we'll stash all our core gameplay code.

- Drop a .uproject file in the main folder.
- Make a Source folder in there too.
	- Inside Source, make a new folder.
        - Try starting the name with your project name, like ${projectname}Core.
            - **{projectname}AI**
            - **{projectName}GamePlay**

## The .uproject File

A .uproject file, short for "Unreal Project File," serves as the entry point and configuration file for an Unreal Engine project. It's a JSON-formatted file that contains essential information about the project, such as its name, description, and the list of modules or plugins it uses. An Unreal Project File is recognized by the UnrealVersionSelector which is stored in the your registry by Unreal when installing Unreal. The following attributes can be found within a .uproject file:

- **FileVersion** Descriptor version number.
- **EngineAssociation** Specifies the engine to open the project with.
    - Allows for opening the correct engine version when double-clicking on a project file.
    - Differentiates between editor versions for upgrade/downgrade UI flow.
    - For Launcher users, it indicates a stable version like "5.0" or "5.1".
    - For Perforce or Git users, it's left blank, allowing determination of the engine based on the directory hierarchy.
    - For source build users with a foreign project, uses a random identifier for engine mapping.
    - For users with the engine mounted through a Git submodule, can be manually edited as a relative path.
- **Category** Category to show under the project browser.
- **Description** Description to show in the project browser.
- **Modules** List of all modules associated with this project.
- **Plugins** List of plugins for this project, which may be enabled or disabled.
- **TargetPlatforms** Array of platforms that this project is targeting.
- **EpicSampleNameHash** A hash used to determine if the project was forked from a sample.
- **PreBuildSteps** Custom steps to execute before building targets in this project.
- **PostBuildSteps** Custom steps to execute after building targets in this project.
- **bIsEnterpriseProject** Indicates if this project is an Enterprise project.
- **bDisableEnginePluginsByDefault** Indicates that enabled by default engine plugins should not be enabled unless explicitly enabled by the project or target files.

*Note:*

*When project file association is not working properly one can simply run **UnrealVersionSelector.exe -fileassociations** from the Unreal Engine Binaries.*
*You could refer to the following files for proper debugging capabilities.*
- *[UnrealVersionSelector.cpp](https://github.com/EpicGames/UnrealEngine/blob/release/Engine/Source/Programs/UnrealVersionSelector/Private/UnrealVersionSelector.cpp#L40)*
- *[DesktopPlatformWindows.cpp](https://github.com/EpicGames/UnrealEngine/blob/release/Engine/Source/Developer/DesktopPlatform/Private/Windows/DesktopPlatformWindows.cpp#L491https://)*

When the project file is loaded into memory it is stored within a `FProjectDescriptor` which contains all the information contained within a .uproject file. Each .uproject file when containing source files is accompanied with a list of modules and/or plugins available to the project. These modules have their own set of attributes as specification when and how they should be loaded by the engine. The following attributes can be found within a Module definition:

- **Name:** Name of this module.
- **Type:** Usage type of the module.
- **LoadingPhase:** Specifies when the module should be loaded during the startup sequence. 
- **PlatformAllowList:** List of allowed platforms for the module.
- **PlatformDenyList:** List of disallowed platforms for the module.
- **TargetAllowList:** List of allowed build target types for the module.
- **TargetDenyList:** List of disallowed build target types for the module.
- **TargetConfigurationAllowList:** List of allowed build configurations for the module.
- **TargetConfigurationDenyList:** List of disallowed build configurations for the module.
- **ProgramAllowList:** List of allowed programs for the module.
- **ProgramDenyList:** List of disallowed programs for the module.
- **AdditionalDependencies:** List of additional dependencies required for building this module.
- **bHasExplicitPlatforms:** When true, an empty PlatformAllowList is interpreted as 'no platforms,' expecting explicit platforms to be added in plugin extensions.

The 2 most important attributes of this definition are the Module Type and the Loading Phase of the module. The Module Type will tell the engine in which environment the module should be loaded,this defaults to Runtime but other options are available such as: 
- **Runtime:** Loads on all targets, except programs.
- **RuntimeNoCommandlet:** Loads on all targets, except programs and the editor running commandlets.
- **RuntimeAndProgram:** Loads on all targets, including supported programs.
- **CookedOnly:** Loads only in cooked games.
- **UncookedOnly:** Loads only in uncooked games.
- **Developer:** Deprecated due to ambiguities. Only loads in the editor and program targets.
- **DeveloperTool:** Loads on any targets where bBuildDeveloperTools is enabled.
- **Editor:** Loads only when the editor is starting up.
- **EditorNoCommandlet:** Loads only when the editor is starting up, but not in commandlet mode.
- **EditorAndProgram:** Loads only on editor and program targets.
- **Program:** Only loads on program targets.
- **ServerOnly:** Loads on all targets except dedicated clients.
- **ClientOnly:** Loads on all targets except dedicated servers.
- **ClientOnlyNoCommandlet:** Loads in editor and client but not in commandlets.

The Module Loading Phase tells the engine when the Module should be loaded into memory during startup:

- **EarliestPossible:** As soon as possible, allowing loading from pak files or immediately after PlatformFile setup.
  - Used for plugins required for file reading (e.g., compression formats).
- **PostConfigInit:** Loaded before the engine is fully initialized, right after the config system initialization.
  - Necessary for very low-level hooks.
- **PostSplashScreen:** Loaded before coreUObject for setting up manual loading screens.
  - Used for chunk patching systems.
- **PreEarlyLoadingScreen:** Loaded before the engine is fully initialized for modules that need to hook into the loading screen before it triggers.
- **PreLoadingScreen:** Loaded before the engine is fully initialized for modules that need to hook into the loading screen.
- **PreDefault:** Loaded right before the default loading phase.
- **Default:** Loaded at the default loading point during startup, after game modules are loaded.
- **PostDefault:** Loaded right after the default phase.
- **PostEngineInit:** Loaded after the engine has been initialized.
- **None:** Indicates the module should not be automatically loaded.

The following is an example descriptor:

```json
{
	"FileVersion": 3,
	"EngineAssociation": "5.0",
	"Category": "",
	"Description": "",
	"Modules": [
		{
			"Name": "PatrolCore",
			"Type": "Runtime",
			"LoadingPhase": "Default"
		}
	]
}
```