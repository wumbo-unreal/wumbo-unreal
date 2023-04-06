---
tags: 
  - unreal
  - Lyra
  - experiences
title: "Lyra Deep Dive - Chapter 2: Experiences"
categories: 
  - Lyra
excerpt: "Lyra introduces the concept of Experiences which are essentially modular game modes. In this chapter, we'll walk through various data assets that define a Lyra Experience."
---

<img src="https://img.shields.io/badge/Unreal%20Engine-5.1-informational" alt="Written for Unreal Engine 5.1"> <img src="https://img.shields.io/badge/-C%2B%2B-orange" alt="C++">

This is the second chapter in the [Lyra Deep Dive](https://unrealist.org/lyra-part-1) series.

## What are Experiences?
In Lyra, an experience is an extensible and modular combination of a GameMode and GameState that can be asynchronously loaded and switched at runtime. In a typical shooter game, Deathmatch and Capture-the-Flag both would be implemented as different experiences. Since experiences are completely modular, they don't even need to be in the same genre! Lyra demonstrates this by having one of the experiences completely transforms the game into a top-down party game.

Most of the code related to Lyra Experiences can be found in the `/LyraGame/GameModes/` directory.

## Source Code
[View the source code for this chapter ❭](https://github.com/the-unrealist/lyra-deep-dive/tree/chapter2-experiences)

I recommend looking at the [diff](https://github.com/the-unrealist/lyra-deep-dive/compare/chapter1-introduction...chapter2-experiences) to see what's changed since the previous chapter.

## Plugins
The Lyra Experiences system is driven by the combination of the **Game Features** and **Modular Gameplay** plugins.

With the Game Features plugin, experiences are contained as standalone game feature plugins and loaded on demand. The Modular Gameplay plugin allows experiences to add components to actors, modify game state, add data sources, and much more.

Both plugins are commonly used together to make actors extensible via plugins and avoid coupling actors with features.

### Game Features
With the Game Features plugin, the game can dynamically load and unload plugins at runtime. Game feature plugins can even be placed in separate standalone chunks to be distributed as downloadable content (DLC).

When you enable the Game Features plugin for your project and restart the editor, you'll see a couple of new plugin templates.

<img src="/assets/images/game-feature-plugins.png" alt="Screenshot of two additional plugin templates: Game Feature (Content Only) and Game Feature (with C++)."/>

These templates include the required `UGameFeatureData` data asset for your standalone game feature. This data asset describes what actions to perform when the feature is loaded and how to find additional primary data assets within the plugin. Keep in mind that all game feature plugins must go in the `/Plugins/GameFeatures/` directory to be detected.

The default set of available actions are: Add Components, Add Cheats, Add Data Registry, and Add Data Registry Source. This can be extended with custom actions. In fact, Lyra has custom actions such as Add Widget, Add Input Binding, and more in the `/LyraGame/GameFeatures/` directory. We'll take a deeper look at these in a future chapter.

You can control the initial state of a game feature by editing the plugin in the Plugins window. In Lyra, all game features have the initial state of **Registered**. This is because with the Lyra Experiences system, the feature will be loaded only when the server selects the experience.

<img src="/assets/images/shooter-core-registered.png" alt="Screenshot of ShooterCore game feature plugin in Lyra with the initial state set to registered."/>

### Modular Gameplay
The Modular Gameplay plugin enables actors to register themselves as receivers for components and senders of custom extension events.

Actors you want to make modular will need to register themselves with the `UGameFrameworkComponentManager`. With the `AddGameFrameworkComponentReceiver` function, the actor notifies the Modular Gameplay subsystem that it is accepting new actor components. This is typically done in the `PreInitializeComponents` function of the actor.

```cpp
void AMyModularActor::PreInitializeComponents()
{
    Super::PreInitializeComponents();
    UGameFrameworkComponentManager::AddGameFrameworkComponentReceiver(this);
}
```

Actors also need to unregister themselves when they're no longer accepting components. This is typically done in the `EndPlay` function of the actor.

```cpp
void AMyModularActor::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    UGameFrameworkComponentManager::RemoveGameFrameworkComponentReceiver(this);
    Super::EndPlay(EndPlayReason);
}
```

Modular actors can create custom extension points by broadcasting custom events to any object subscribed to the actor class via `UGameFrameworkComponentManager`.

Most modular actors will want to send the `GameActorReady` event in `BeginPlay` so that extensions can execute code only when the actor is active.

```cpp
void AMyModularActor::BeginPlay()
{
    UGameFrameworkComponentManager::SendGameFrameworkComponentExtensionEvent(this, UGameFrameworkComponentManager::NAME_GameActorReady);
    Super::BeginPlay();
}
```

To create an extension, call `AddExtensionHandler` in the `UGameFrameworkComponentManager` subsystem. This will subscribe to all actors of the desired class for extension events. It must be an actor subclass and not the root `AActor` class, which means you _cannot_ subscribe to every single actor. This is intentionally checked to prevent significant performance impact to the game.

```cpp
// This snippet is placed where you want to start handling actor extensions.
if (UGameFrameworkComponentManager* ComponentManager = UGameInstance::GetSubsystem<UGameFrameworkComponentManager>(GameInstance))
{			
    TSharedPtr<FComponentRequestHandle> ExtensionRequestHandle = ComponentManager->AddExtensionHandler(
        AMyModularActor::StaticClass(),
        UGameFrameworkComponentManager::FExtensionHandlerDelegate::CreateUObject(this, &ThisClass::HandleActorExtension));
}
```

Store `ExtensionRequestHandle` somewhere. Call `Unregister` on it to stop listening for extension events.

In the handler delegate, you typically would check the event name and execute code on the actor if it's an event you wish to handle.

```cpp
void UMyActorExtension::HandleActorExtension(AActor* Actor, FName EventName)
{
    if (EventName == UGameFrameworkComponentManager::NAME_GameActorReady)
    {
        // Do something when the actor is ready.
    }
    
    // Handle other extension events here.
}
```

### Modular Gameplay Actors
The `ModularGameplayActors` plugin contains subclasses of the [standard gameplay framework actors](https://docs.unrealengine.com/5.1/en-US/gameplay-framework-quick-reference-in-unreal-engine/) that are registered for extension via the Modular Gameplay plugin. While you can always register directly with `UGameFrameworkComponentManager` in your actors, this plugin makes it so that you don't need to do it manually.

These actors are provided by this plugin:
* `AModularAIController`
* `AModularCharacter`
* `AModularGameModeBase`
* `AModularGameMode`
* `AModularGameStateBase`
* `AModularGameState`
* `AModularPawn`
* `AModularPlayerController`
* `AModularPlayerState`

This plugin does not come with Unreal Engine out of the box, so we need to manually create it. You can find this plugin in [the source code for this chapter](https://github.com/the-unrealist/lyra-deep-dive/tree/chapter2-experiences/LyraStarterGame/Plugins/ModularGameplayActors).

## Experience Definition Data Asset
TODO

## Action Sets
TODO

## User-Facing Experience Definition Data Asset
TODO

## Asset Manager
TODO: Primary asset types to scan in project settings and game feature asset.

Explain that experience definitions must go into /Game/System/Experiences directory  and user facing definitions in /Game/System/Playlists in Lyra by default, unless explicitly added under Specific Assets, or another directory is added. And LyraExperienceActionSet must be defined to be detected in dropdown list when the uproperty meta specifier is set.

## Chunks & DLC
TODO: explain paks and how to create a DLC experience

## Next Steps
In the next chapter, we will explore the lifecycle of experiences including how they are loaded and applied to all players.