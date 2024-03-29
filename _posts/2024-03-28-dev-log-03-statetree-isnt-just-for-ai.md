---
tags: 
  - unreal
title: "Dev Log 03: StateTree isn't just for AI"
categories:
  - Dev Logs
---

# Dev Log 03: StateTree isn't just for AI

<img src="https://img.shields.io/badge/Unreal%20Engine-5.4-informational" alt="Written for Unreal Engine 5.4"> <img src="https://img.shields.io/badge/-Dev%20Log-red" alt="Dev Log">

StateTree is a hierarchical state machine introduced in Unreal Engine 5.

<img src='/assets/images/plugin-1.png' alt="StateTree plugin description in the Plugin window" />

The base `StateTree` plugin provides a general-purpose hierarchical state machine, but it does not provide any schema or even a processor to actually execute a StateTree.

Unreal Engine comes with a handful of plugins with their own schemas and processors.

| Plugin | Purpose | Processor |
| --------------- | ------- | -------- |
| `GameplayStateTree` | StateTree for actors. | `StateTreeComponent` actor component|
| `GameplayInteractions` | StateTree for smart objects. | TBD |
| `MassAI` | StateTree for Mass entities. | TBD |

Unreal Engine uses StateTree exclusively for AI, and even placed the asset type under the *Artificial Intelligence* category.

<img src='/assets/images/st-picker.png' alt="StateTree asset type categorized under Artificial Intelligence" />

**I'm here to let you know that it's not just for AI!**

Once again, StateTree is a *general purpose* state machine built into Unreal Engine with its own editor. You don't need a third-party plugin or develop your own state machine. In fact, Unreal Engine 5.4 has made it even more useful with linked assets and themes.

## Why do I need a state machine?

First, let's take a look at my game's frontend. The player goes through this sequence when launching my game:

@startmermaid
%%{ init: { 'flowchart': { 'curve': 'monotoneY' } } }%%
graph TD
    launch([Launch]) --> splash
    splash(Startup Movies) --> title_screen
    title_screen["Title Screen <br/> 'Press 🅐 to Start'"]
    title_screen -- Pressed 🅐 --> signin
    signin{Platform<br/>sign-in}
    signin -- Failed --> error_modal_signin
    error_modal_signin[Begin offline mode]
    error_modal_signin --> oobe
    signin -- Success --> oobe
    oobe{First run?} -- No --> quick_resume
    quick_resume{Has existing save?}
    quick_resume -- No --> title_menu
    quick_resume -- Yes --> confirm_load_save
    confirm_load_save[Prompt load save]
    confirm_load_save -- No --> title_menu
    confirm_load_save --> lobby
    title_menu[Title Menu<br/><ul style="text-align: left;"><li>Play</li><li>Settings</li><li>Exit</li></ul>]
    oobe -- Yes --> settings_firstrun
    settings_firstrun[Accessibility Settings]
    settings_firstrun -- Save / Cancel --> charcreate
    charcreate[Character Creation]
    title_menu -- Settings --> settings
    settings -- Save / Cancel --> title_menu
    settings[Settings]
    title_menu -- Exit --> exit
    exit([Exit Game])
    charcreate -- Cancel --> title_menu
    charcreate -- Confirm --> lobby
    lobby([Co-op Lobby])
    title_menu -- Play --> has_slots
    has_slots{Saves exist?}
    has_slots -- No --> charcreate
    has_slots -- Yes --> loadgame
    loadgame[Select Save]
    loadgame -- Create New Game --> charcreate
    loadgame -- Save Selected --> lobby
    settings ~~~ oobe
    lobby -- Cancel --> title_menu
    lobby --> start[Start Game]
@endmermaid

I want to bring my players into the game as soon as possible. Launching the game for the first time opens accessibility settings. On this screen, players may switch to graphics and audio settings if they wish.

Then it proceeds to the character creation screen. Next, the game creates a new save and enters the co-op lobby with the save slot name as a URL parameter.

If it's not the first time, then it will prompt the player if they want to load their most recent save. Declining will bring the player to the main menu. This means for most players, they will never see the main menu at all!

But widgets have to have references to other widgets. The main menu widget has to be responsible for showing the settings and save selection menus. Some widgets have to check whether a save slot exists and then create a different widget based on this information. I could go on and on…

**Without a state machine, this produces [Blueprint spaghetti](https://blueprintsfromhell.tumblr.com/) that is fragile, hard to maintain, and prone to bugs!**

## Using StateTree for the frontend

Here's what my frontend looks as a StateTree. Each step in the flowchart above is implemented as a self-contained discrete state with tasks, conditions, and transitions.

<img src='/assets/images/st-frontend.png' alt="My game's frontend flow implemented as a StateTree showing a hierarchy of states"/>

The *StateTree Component* schema provides an actor as context data. Since my use case involves input and widgets, I set the actor type to PlayerController. The startup level in my game uses a special PlayerController actor that has a StateTree Component. This makes the StateTree immediately begin executing when the game loads.

> The funny thing is I actually spent weeks building a UMG StateTree with its own custom schema and a set of tasks. Unfortunately, what I ended up with is more or less the same thing as the built-in GameplayStateTree plugin. The only real difference between my plugin and GameplayStateTree is that the processor is implemented as a subsystem rather than as an actor component.
>
> I even wrote a whole article about this, but it doesn't feel right to publish it when I realized the better solution is to just use GameplayStateTree.

Most tasks complete in a success or failed state. For example, a player wanting to back out of character creation causes the state to fail. This will trigger a transition to bring the player back to the main menu.

As for the main menu, there's no success or failure condition. Instead, clicking on a menu button raises a StateTree event. A transition is set up for each event to enter another state that actually does something. The main menu widget only *reports* player intent, and leaves it to the StateTree to decide what to do next.

<img src='/assets/images/st-mainmenu-events.png' alt="Details panel for state with Create Main Menu task and transitions for Select Quit, Select Settings, and Select Play events" />

Widgets are clearly not tasks, so how did I raise a StateTree event? My solution is to create a base widget class with an event dispatcher that top-level widgets must subclass from.

Tasks bind to this event dispatcher after creating the widget. To simplify this even further, I created a base task class called *CreateWidgetAsync* that takes in a soft class reference as a parameter. By subclassing from this task, I don't need to reimplement the same logic over and over.

<img src='/assets/images/st-createwidgetasync.png' alt="Blueprint graph for CreateWidgetAsync. Enter State event calls Create Widget Async macro and binds to the widget's On Widget Event which then calls State Tree Send Event." />

Notably, it does not call *Finish Task* unless there was an error. This is how a widget remains visible indefinitely.

By the way, **always use soft references in task parameters!** Avoid hard references to any blueprint widget (other than the base class). If you're not careful, the StateTree's memory footprint will skyrocket.

## StateTree tips & tricks
Here are some things about StateTree I wish I knew about earlier.

### Rename tasks
Did you know you could rename tasks? I didn't for an embarassingly long time. This really helps with identifying which widget to target in a property binding.

Just click on the task name to edit it.
 
<img src='/assets/images/rename-task.png' alt="Task picker with a portion of the task named highlighted" />
<br />
<img src='/assets/images/property-binding.png' alt="Property binding showing the new task name in the picker" />

This also works for conditions and evaluators.

### Organize blueprint nodes
In each one of your blueprint nodes (tasks, conditions, or evaluators), be sure to go to *Class Settings* and override the display name and category. This will make it easier to find your nodes in the picker.

<img src='/assets/images/st-blueprint-options.png' alt="Blueprint options with Blueprint Display Name set to Wait for Input and Blueprint Category set to Frontend" />

### Parameter types
The `Category` specifier sets the *type* of a parameter.

<img src='/assets/images/st-parameter-types.png' alt="Blueprint variables under the Context, Input, Parameter, and Output categories" />
<br />

<img src='/assets/images/st-parameters.png' alt="Aforementioned parameters displayed as bindable properties in task details. One labeled Context, one labeled In, one labeled Out." />

There are 3 special types of parameters:

|Category|Behavior|
|----|-------|
|`Context`|A value is required. Automatically links to context data in the StateTree with the same type, but may be overridden with a binding.|
|`Input`|A value is required unless marked optional with `meta=(Optional)`. This value can only be set with a binding.|
|`Output`|This value can only be bound to other properties.|

Parameters in all other categories appear normal.

### Condition operators and indentation
Adding more than one condition will reveal operators. Click on it to switch between AND/OR.

<img src='/assets/images/state-tree-condition-operator.png' alt="Condition details with a button with a popup containing the OR AND logical operators" />

There is also an invisible button right before the operator button. Click on it to change the indentation of a condition. Operators apply to conditions within the same indentation.

<img src='/assets/images/state-tree-condition-indent.png' alt="Popup with numbers 0 to 3 are shown below an empty space next to the logical operator button" />

### Subtree transitions
The *Tree Succeeded* and *Tree Failed* transitions inside a subtree will surface to the linked state and no further. However, if a subtree was entered by a transition instead of a linked state, then these transitions will affect the whole StateTree.

## Ending thoughts
I'm not claiming this is the best approach, but it does work pretty well. The biggest benefit of using StateTree is that I can see the entire flow all in a single asset. So, yeah, I'm happy with what I have right now. :)
