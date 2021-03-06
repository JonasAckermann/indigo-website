---
id: scene-management
title: Scenes & Scene Management
---

## What are Scenes?

As soon as you decide to build a game of any complexity, you immediately hit the problem of how to organize your code so that everything isn't lumped together. What you'd really like, hopefully, is a nice way to think about each part of your game in logical groups of state and functions i.e. All the things to do with your menu screen in one place, separated from your game over screen.

Most game engines have some sort of concept for these groupings, they're often called scenes, and Indigo is no exception.

## Working example: "Snake!"

If you'd like to dive right in, the Snake implementation uses scenes to manage it's screens. To help you find your way around, here are a few points of interest:

- Initial declaration of the [list of scenes](https://github.com/PurpleKingdomGames/indigo/blob/master/demos/snake/snake/src/snake/SnakeGame.scala#L38).

- The "Start" [scene](https://github.com/PurpleKingdomGames/indigo/blob/master/demos/snake/snake/src/snake/scenes/StartScene.scala), which is one of the simpler scenes in the game.

- The point where the "Start" scene decides to [jump](https://github.com/PurpleKingdomGames/indigo/blob/master/demos/snake/snake/src/snake/scenes/StartScene.scala#L27) to the "Controls" scene (where the player chooses a keyboard layout) after the space bar has been pressed.

## A quick look under the hood

Scenes give the appearance of building a separate game per scene, with only a nod to the underlying mechanics.

In fact, you could build you're own scene system right on top of one of Indigo's other entry points if you were so inclined.

The way it works, is that all scene data for every scene is held in the game's model and you provide a way to take it out and put it back again. Then the update and presentation functions are delegated to the currently running scene, and scene navigation is controlled by events. That's it.

## Building a game with Scenes

To use scenes, you need to use the `IndigoGame` entry point (extend an object in the usual way and fill in the blanks), which builds on `IndigoDemo` (which builds on `IndigoSandbox`) and adds the additional mechanics for using scenes.

A slightly abridged version of `IndigoGame` looks like:

```scala
trait IndigoGame[BootData, StartupData, Model, ViewModel] {
  def boot(flags: Map[String, String]): BootResult[BootData]
  def scenes(bootData: BootData): NonEmptyList[Scene[Model, ViewModel]]
  def initialScene(bootData: BootData): Option[SceneName]
  def setup(bootData: BootData, assetCollection: AssetCollection, dice: Dice): Startup[StartupErrors, StartupData]
  def initialModel(startupData: StartupData): Model
  def initialViewModel(startupData: StartupData, model: Model): ViewModel
}
```

The notable points in this interface compared with the others are:

1. Game initialization still happens here as normal.
1. That you _must_ provide an _ordered_ `NonEmptyList` (see below) of `Scenes`.
1. That you _can_ provide the name of the first scene Indigo should use, otherwise the scene at the head of the list will be used.
1. The usual update and present functions now live inside your scene definitions.

## Building a Scene

A scene is built by creating an object (or class) that extends `Scene[StartupData, GameModel, ViewModel]`. Here's the trait:

```scala
trait Scene[StartupData, GameModel, ViewModel] {
  type SceneModel
  type SceneViewModel

  val name: SceneName
  val modelLens: Lens[GameModel, SceneModel]
  val viewModelLens: Lens[ViewModel, SceneViewModel]
  val eventFilters: EventFilters
  val subSystems: Set[SubSystem]

  def updateModel(context: FrameContext[StartupData], model: SceneModel): GlobalEvent => Outcome[SceneModel]
  def updateViewModel(context: FrameContext[StartupData], model: SceneModel, viewModel: SceneViewModel): GlobalEvent => Outcome[SceneViewModel]
  def present(context: FrameContext[StartupData], model: SceneModel, viewModel: SceneViewModel): SceneUpdateFragment
}
```

As you can hopefully see, mostly this is very much like a normal game, but for a few exceptions:

1. No initialization, animations or fonts, that all happens in the main game (shown in the previous section).
1. Scenes have a name, which is important for navigation.
1. Scenes have their own models and view models, more on that later.
1. The game can have global sub systems, and scenes can also have their own subsystems too.

## Funny types

There's a couple of funny types in the code snippets above, namely `NonEmptyList` and `Lens`. The main thing to stress (if you're familiar with them already) is that they are minimal implementations within Indigo itself, and not the fully featured versions you might find in specialist libraries.

This choice - right or wrong - was made because most of Indigo is vanilla Scala* and requires nothing out of the ordinary to work beyond Scala.js, but just occasionally we can do much better with a cleverer type.

The two used above are `NonEmptyList` and `Lens`. The latter is discussed in the next section. A `NonEmptyList` is a `List` that cannot be empty, i.e. it will always have at least one element, and so the normally unsafe (i.e. throws an exception if the list is empty) `.head` method becomes a safe operation. You can think of the encoding as being like this:

```scala
final case class NonEmptyList[A](head: A, tail: List[A]) {
  def toList: List[A] = head :: tail
}
```

> *There is absolutely nothing stopping you from using all your favorite libraries, such as Cats or Monocle.

## Navigating between scenes

The non-empty list of scenes in the original declaration is static, and cannot be added to later in the game. It is also ordered.

Here's the one from Snake:

```scala
def scenes(bootData: GameViewport): NonEmptyList[Scene[SnakeGameModel, SnakeViewModel]] =
    NonEmptyList(StartScene, ControlsScene, GameScene, GameOverScene)
```

Snake also declares it's initial scene like this:

```scala
def initialScene(bootData: GameViewport): Option[SceneName] =
    Option(StartScene.name)
```

But this isn't actually necessary since `StartScene` is at the head of the list.

To move between scenes, you use events, defined simply as:

```scala
sealed trait SceneEvent extends GlobalEvent
object SceneEvent {
  case object Next                         extends SceneEvent
  case object Previous                     extends SceneEvent
  final case class JumpTo(name: SceneName) extends SceneEvent
}
```

`Next` and `Previous` proceed forwards and backwards respectively through the list of scenes until they run out. They do not loop back on themselves.

`JumpTo` moves to whichever scene you specify with the `SceneName`. This turns out of be very convenient since scenes are normally objects, and you can just call something like `JumpTo(GameOverScene.name)`.

## State handling with Lenses

The final thing to know about with `Scene`s, is how they manage state.

Essentially the model of the game contains the state for all scenes. This applies to both model and view model, but we'll just talk about the model from now on. Consider a game with the following model:

```scala
final case class MyGameModel(sceneA: SceneModelA, sceneB: SceneModelB)
```

So if we want to run the game and show scene B, then what we need to do is pull `sceneB` out of the model, update our game based on its values, and then put it back, like this:

```scala
val scene: SceneB = ???
val model: MyGameModel = ???

model.copy(
  sceneB = scene.updateSceneModel(context, model.sceneB)(e)
)
```

Easy. But this is a trivial example where scene B's model is literally a field inside the game model, but how about a deeply nested update? Or something like this:

```scala
final case class MyGameModel(name: String, health: Int, isHuman: Boolean, inventory: Map[String, Item])
final case class SceneModelB(health: Int, inventory: Map[String, Item])
```

We can clearly construct `SceneModelB` from `MyGameModel`, and we may still use `copy` but it's going to be a bit more involved. What if we also then wanted to update something in the inventory? Is there an elegant way to do that?

### Lenses

To formalize this sort of relationship, Indigo has a just-about-good-enough `Lens` implementation. Lenses are a really interesting subject and if you'd like to know more you could take a look at something like [Monocle](https://github.com/optics-dev/Monocle).

Minimally, a lens implements a `get` and a `set` function, like so:

```scala
def get(from: A): B
def set(into: A, value: B): A
```

- `get` looks at, in our case, `MyGameModel` and extracts / returns `SceneModelB`.
- `set` puts a `SceneModelB` into a `MyGameModel` and returns the new `MyGameModel` instance.

Lets try it out! Lets start with the simple `copy` example from earlier:

```scala
val sceneBLens =
  Lens(
    (model: MyGameModel) => model.sceneB,
    (model: MyGameModel, newSceneB: SceneModelB) => model.copy(sceneB = newSceneB)
  )

sceneBLens.get(model) // SceneModelB
sceneBLens.set(model, newSceneModelB) // MyGameModel
```

Or the more complicated example might look like:

```scala
  Lens(
    (model: MyGameModel) => SceneModelB(model.health, model.inventory),
    (model: MyGameModel, newSceneB: SceneModelB) =>
      model.copy(
        health = newSceneB.health,
        inventory = newSceneB.inventory,
      )
  )
```

### Lens composition

We also posed the question of how you update things inside other things, for this we have to compose lenses together, for example:

```scala
final case class Sword(shininess: Int)
final case class Weapons(sword: Sword)
final case class Inventory(weapons: Weapons)

val mySwordLens = 
  inventoryLens andThen weaponsLens andThen swordLens

mySword.get(inventory) // a sword
mySword.set(inventory, betterSword) // an inventory with a better sword in it

val polishSword(s: Sword): Sword = ???
mySword.modify(inventory, polishSword)
// modify is the just the composition of get, set, and a function f.
```

### Limitations

As with all things in Indigo, the `Lens` implementation is the bare minimum needed - if you assume most models are just nested objects - and does not currently support things like prisms.

No doubt extra functionality will be added as soon as the need arises, but in the meantime, note that Indigo's lens definition says nothing about how it's implemented. If you had a complicated case, you could look at building your lenses using a Scala.js compatible lens library, and just use the Indigo Lens as an interface to the engine.
