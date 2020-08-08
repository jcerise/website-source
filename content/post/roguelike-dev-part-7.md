+++
title = "RoguelikeDev Builds a Roguelike, Part 7"
date = 2017-07-27T22:58:00-06:00
description = "Part 7: An Entity/Component/System Refactor to make our lives easier"
draft = false 
toc = false
categories = ["technology"]
tags = ["RLDBAR", "go", "programming", "roguelike", "entity component", "entity component system"]
+++

The time has finally come where our small game is going to become unmanageable when adding new features. Certainly, we could make it work (and in fact the official tutorial and revamped tutorial come up with some clever ways around this complexity), but I'm not a fan of headaches, or unmaintainable code. I've always wanted to write an entity component system setup from scratch, to better understand how they work, and and ECS can dramatically help us cut down on complexity of new features. So, that's the route we're going to take. A bit of warning in advance, I have no experience with ECS whatsoever, so this should be exciting! Lets get to it!

<!-- more -->

A bit of background first. At this point in our development cycle, we are about to start adding some more complex features to our game. Namely, actual combat, and monsters etc that the player will be fighting. Beyond that, we're also going to be soon adding support for items (consumable, equip-able, and otherwise). These two features on their own warrant a large amount of thought into how we are going to implement them. At this point, the official tutorial that RLDBAR is following takes the most obvious, and probably easiest to understand route of classic inheritance. Using this model, we could create a base Entity class (which we have already done), that contains a few common attributes that every entity in the game should have, things like name, character, color, HP, position, etc. Then, from this base entity class, we could create more specialized entity types, to say represent a Goblin. The Goblin entity would have all the base entity attributes, alongside some specific to a Goblin. From there, we could create an even more specific version of a Goblin, say a Goblin Chief, which would inherit all the base attributes of an entity, along with the attributes of a Goblin. Eventually, you can see how this would lead to quite a complex chain of inheritance, and a large number of classes. In some cases, we might even be tempted to use multiple inheritance (if the language supports it), which can lead to all sorts of problems, the least of which is confusing code.

Inheritance is certainly one way to solve this problem, but there must be a better way. The old idiom "composition over inheritance" immediately comes to mind. Rather than have an inheritance based system, what if we could use composition to create out entities? Basically, writing a bunch of smaller components that we can then attach to an entity, to describe how it should behave at any given time; sounds like a good approach. Turns out, this is a pretty common game design pattern, called Entity Component System. Using Entity Component System (henceforth known as ECS), we can define a base entity, which will be as simple as possible (literally just an ID, and a list of components), and then attach components to it (which themselves are just a collection of attributes). We will then write a series of systems that act upon the various components for each entity.

Using this setup, we can create any combination of entity imaginable. Say we want to create the Goblin from earlier. Previously, this involved creating a new class that inherited from Entity. Now, we can just create a new entity, and attach the appropriate components to it (appearance, position, movement, hitpoints, attacker, killable, and a basic AI component). If we wanted to make a Goblin Chief, we would simply create a new entity with the same components attached as the base goblin, but with maybe a more advanced AI, to represent the greater threat it poses. Similarly, we could also use this system to create items (an entity with an equip-able component, and an attackStats component, along with an appearance and position, could represent an axe found on the floor. A chest could be an entity with openable, inventory, and breakable components). Hopefully these examples aren't too contrived, and you can understand the power of a system like this.

We'll use a series of systems to tie this all together. Our systems will be small chunks of logic that check if the appropriate components are present on an entity, and if so, will act upon them accordingly. A movement system might check for the movement component, alongside position and appearance, and if those are present, adjust the x and y coordinates in the position component to represent the entity moving around the map.

To summarize, an entity is a small container with an ID, and a list of components. A component contains no logic, just data, and can optionally just be a flag (contains no data, it will simply be present or not). A system contains the logic for a specific component or set of components.

A quick note, there are a number of implementations of ECS out there (most notably, at least to me, is [Artemis](http://entity-systems.wikidot.com/artemis-entity-system-framework)). I've decided to roll my own, both to help me understand how they work, and also simply because I didn't find a good pre-built solution for Go :p

So, now that we have a basic idea of what an ECS is, and generally how they work, lets start looking at how to shoehorn one into our game in progress. We've got a bit of work to do, as this is a pretty major refactor. We've already got an Entity struct that we can leverage, but its currently tracking way too much data. That's going to be where we'll start, cutting the fat in our Entity class to better match our goal. Before we get to the code, I'm going to refactor our ecs package a bit. It will now have three main files, entity.go, components.go, and systems.go. Hopefully, each of these should be obvious as to their contents. Lets start by looking at our new, skinny entity struct:

{{< highlight go >}}
type GameEntity struct {
	gmUUID     uuid.UUID
	Components map[string]Component
	mux        sync.Mutex
}
{{< /highlight >}}

As you can see, we've eliminated all but two fields (plus a mutex field, but thats beyond the scope of this post, and can be safely ignored). gmUUID is a guid that we generate for each new entity we create. Components is a map of components (indexed by the components name). And that's it. I'm also going to outline some convenience methods attached to this struct that will make our lives easier later:

{{< highlight go >}}
func (e *GameEntity) HasComponent(componentName string) bool {
	// Check to see if the entity has the given component
	if _, ok := e.Components[componentName]; ok {
		return true
	} else {
		return false
	}
}

func (e *GameEntity) HasComponents(componentNames []string) bool {
	// Check to see if the entity has the given components
	containsAll := true
	if e != nil {
		for i := 0; i < len(componentNames); i++ {
			if !e.HasComponent(componentNames[i]) {
				containsAll = false
			}
		}
	} else {
		return false
	}
	return containsAll
}

func (e *GameEntity) AddComponent(name string, component Component) {
	// Add a single component to the entity
	e.Components[name] = component
}

func (e *GameEntity) AddComponents(components map[string]Component) {
	// Add several (or one) components to the entity
	e.mux.Lock()
	for name, component := range components {
		if component != nil {
			//fmt.Printf("Adding component: %s - %v\n", name, component)
			e.Components[name] = component
		}
	}
	e.mux.Unlock()
}

func (e *GameEntity) RemoveComponent(componentName string) {
	// Remove of a component from the entity
	_, ok := e.Components[componentName]

	if ok {
		delete(e.Components, componentName)
	}
}

func (e *GameEntity) RemoveComponents(componentNames []string) {
	e.mux.Lock()
	for i := 0; i < len(componentNames); i++ {
		_, ok := e.Components[componentNames[i]]

		if ok {
			delete(e.Components, componentNames[i])
		}
	}
	e.mux.Unlock()
}

func (e *GameEntity) GetComponent(componentName string) Component {
	// Return the named component from the entity, if present
	if _, ok := e.Components[componentName]; ok {
		return e.Components[componentName]
	} else {
		return nil
	}
}
{{< /highlight >}}

These functions should be pretty self explanatory, so I'll leave their descriptions brief. Basically, these are just ways to check if the entity has a given component (or list of components), add a component to the entity (or a list of components), and removing a component from the entity (or a list of components). All in all, our entity sturct is pretty simple.

Next, we're going to define an interface that will encompass all of our created components. Interfaces in Go work in a very simple fashion. An interface defines function definitions, and any struct that implements all of those definitions is said to be of that interface. The reason we're doing it this way, is because we are going to want a generic (Go doesn't support proper generics) way of putting a component (of many different types) onto an entity. Implementing the interface on all of our components will make them all "generic" components, in addition to their actual type, making it much easier for us to attach, detach, and check for them on entities. The following code is in components.go:

{{< highlight go >}}
type Component interface {
	IsAIComponent() bool
}
{{< /highlight >}}

Very basic. We define one function signature, IsAIComponent(), that will be true if the component is an AI component, and false otherwise. This will make a little more sense in a bit. All of our components will have this function attached, and therefore will be a Component.

We're now going to define a few basic components that most of our entities will have:

{{< highlight go >}}
/ Player Component
type PlayerComponent struct {
}

func (pl PlayerComponent) IsAIComponent() bool {
	return false
}

// Position Component
type PositionComponent struct {
	X int
	Y int
}

func (pc PositionComponent) IsAIComponent() bool {
	return false
}

// Appearance Component
type AppearanceComponent struct {
	Color     string
	Character string
	Layer     int
	Name      string
}

func (a AppearanceComponent) IsAIComponent() bool {
	return false
}
{{< /highlight >}}

Lets go through these one by one. First up is the PlayerComponent(). This is what I'm going to refer to as a flag component. The PlayerComponent struct doesn't have any data associated with it. This type of component simply indicates a state on the entity it is attached to. In this, it indicates that the entity is the player. Obviously, only one entity in the game will have this component (although their may be states in the future where this is not true, that's the beauty of the flexibility of ECS).

Next, we have a PositionComponent. This component has an x and a y field. This will, unsurprisingly, represent where an entity is in the game world. Any entity that is active on the map will have this component.

AppearanceComponent means that an entity has a physical representation on the map. It has a color, character, layer, and name, all things that allow us to display it the player. If an entity doesn't have an AppearanceComponent, it won't be shown at all (we'll be altering the state of this component when an entity is killed, to represent their newly corpse like appearance).

This should give a good overview of what components are, and the general idea of how I am defining them. Keeping this simple is the name of the game.

Now that we've got our new entity and component definitions all squared away, lets see how we can actually use them. The first thing I'm going to do is re-define our player entity. You may remember, previously, we have been creating our player entity by defining a list of values for various attributes on the entity itself. Now that we have our new system in place, this will look a bit different.

This is in our init() function, in our main file:

{{< highlight go >}}
// Create a player Entity, and add them to our slice of Entities
player = &ecs.GameEntity{}
player.AddComponent("player", ecs.PlayerComponent{})
player.AddComponent("position", ecs.PositionComponent{X: 0, Y: 0})
player.AddComponent("appearance", ecs.AppearanceComponent{Color: "white", Character: "@", Layer: 1, Name: "Player"})
player.AddComponent("movement", ecs.MovementComponent{})
player.AddComponent("controllable", ecs.ControllableComponent{})
{{< /highlight >}}

This is how we're going to define entities from here out. All we're doing is adding a list of components to the newly created entity. Each component has a name, and a component type. You may notice that there are some components here that I have not previously discussed, movement and controllable. You'll notice these are both flag components, as they take no data. These indicate that the player is directly controllable using the keyboard, and that it is capable of movement.

I'm setting the players position component to values of X: 0 and Y:0. This is because we generate the starting coordinates for the player from the map, if you recall. Lets see how setting that has changed:

{{< highlight go >}}
if player.HasComponent("position") {
		positionComponent, _ := player.Components["position"].(ecs.PositionComponent)
		positionComponent.X = playerX
		positionComponent.Y = playerY
		player.RemoveComponent("position")
		player.AddComponent("position", positionComponent)
}
{{< /highlight >}}

First, we check to make sure the player actually has a PositionComponent (this is a pattern we will repeat much in the near future). If it does, we grab the PositionComponent (using a type assertions to ensure Go that are actually pulling a PositionComponent), and then we set it to the x and y value from our map generation code. Then, we remove the existing PositionComponent, and replace it with our updated one (as we are not using pointers for the components). That's it, our player now has an updated PositionComponent, and will be reflected in that position when we draw it to the screen.

Speaking of rendering, our renderEntities() function will need a bit of an overhaul, now that we cannot directly access properties on each entity. In fact, we're going to completely remove renderEntities(), and replace it with our first system. We're going to create a new system, called SystemRender(). This system will take the master list of entities (unchanged through our refactor, we're still keeping track of them the same way), and render those who have a position and appearance component. This system will live in our systems.go file:

{{< highlight go >}}
func SystemRender(entities []*GameEntity, camera *camera.GameCamera, gameMap *gamemap.Map) {
	// Render all renderable entities to the screen
	for _, e := range entities {
		if e != nil {
			if e.HasComponents([]string{"position", "appearance"}) {
				pos, _ := e.Components["position"].(PositionComponent)
				app, _ := e.Components["appearance"].(AppearanceComponent)

				cameraX, cameraY := camera.ToCameraCoordinates(pos.X, pos.Y)

				if gameMap.Tiles[pos.X][pos.Y].Visible {
					blt.Layer(app.Layer)
					blt.Color(blt.ColorFromName(app.Color))
					blt.Print(cameraX, cameraY, app.Character)
				}
			}
		}
	}
}
{{< /highlight >}}

This code should look sort of familiar, as at its core, its no different than our previous rendering code. The main difference lies in how we're accessing the appearance and position information. Whereas previously, we would pull that information directly off the entity, we now pull it off of the entities components (if they are present). We first check each entity to make sure it is valid for this system (has a position and appearance), then we grab those components (again, using a type assertion on each). WE then use the information on each to draw the entity to the screen in the same manner we always have.

Now that we've got that fancy new system sitting there, ready to go, where and how do we call it? The answer is, just like our old renderEntities function, and in roughly the same place. Back in our main game file, in our main() function:

{{< highlight go >}}
func main() {
	// Main game loop

	messageLog.SendMessage("You find yourself in the caverns of eternal sadness...you start to feel a little more sad.")
	messageLog.PrintMessages(ViewAreaY, WindowSizeX, WindowSizeY)
	renderMap()
	ecs.SystemRender(entities, gameCamera, gameMap)

    ...

    renderMap()
		ecs.SystemRender(entities, gameCamera, gameMap)
		messageLog.PrintMessages(ViewAreaY, WindowSizeX, WindowSizeY)
	}

	blt.Close()
}
{{< /highlight >}}

That should look pretty familiar. Really, all we've done is replace the old call to RenderAll() (which called RenderEntities()) with a call to our new system function, passing in the entities, gameCamera, and gameMap. That's all. Nothing crazy here. Every time through our game loop, SystemRender() will be called, it will check each entity for the proper components, and, if present, render the entity to the screen.

One last thing we need to do is replace our monster generation with our ECS setup. This will be in cavern.go, in the gamemap package:

{{< highlight go >}}
if chance <= 25 {
			// Create a Troll
			createdEntity = &ecs.GameEntity{}
			createdEntity.SetupGameEntity()
			createdEntity.AddComponents(map[string]ecs.Component{"position": ecs.PositionComponent{X: x, Y: y},
				"appearance":     ecs.AppearanceComponent{Layer: 1, Character: "T", Color: "dark green", Name: "Troll"},
				"hitpoints":      ecs.HitPointComponent{Hp: 20, MaxHP: 20},
				"movement":       ecs.MovementComponent{}})
		} else if chance > 25 && chance <= 50 {
			// Create an Orc
			createdEntity = &ecs.GameEntity{}
			createdEntity.SetupGameEntity()
			createdEntity.AddComponents(map[string]ecs.Component{"position": ecs.PositionComponent{X: x, Y: y},
				"appearance":     ecs.AppearanceComponent{Layer: 1, Character: "o", Color: "darker green", Name: "Orc"},
				"hitpoints":      ecs.HitPointComponent{Hp: 15, MaxHP: 15},
				"movement":       ecs.MovementComponent{}})
		} else if chance > 50 {
			// Create a Goblin
			createdEntity = &ecs.GameEntity{}
			createdEntity.SetupGameEntity()
			createdEntity.AddComponents(map[string]ecs.Component{"position": ecs.PositionComponent{X: x, Y: y},
				"appearance":     ecs.AppearanceComponent{Layer: 1, Character: "g", Color: "green", Name: "Goblin"},
				"hitpoints":      ecs.HitPointComponent{Hp: 5, MaxHP: 5},
				"movement":       ecs.MovementComponent{}})
{{< /highlight >}}

We're using the AddComponents() function easily add multiple components at once to our created entities, but other than that, we create these in much the same way as our player. This highlights, I think, how easy it is to compose new entities.

At this point, we should have all of our entities drawing to the screen properly. Lets now tackle movement, before we wrap up. We're going to add a new component (hinted at above) called MovementComponent, and along with that, a new system, SystemMovement. My goal is to make all of our systems as generic as possible, so that we are not handling special cases that are unique to the player (you'll see this our SystemMovement system, as in theory, it will be possible to control any entity in the game; the result of a mind control spell, maybe?).We'll also add a flag component, to indicate if the entity is controllable or not. Lets add our component first:

{{< highlight go >}}
// Movement Component
type MovementComponent struct {
}

func (m MovementComponent) IsAIComponent() bool {
	return false
}

// Controllable Component
type ControllableComponent struct {
}

func (c ControllableComponent) IsAIComponent() bool {
	return false
}
{{< /highlight >}}

Nothing new here, its still not an AI component either. Now, lets code up the accompanying system. This will be very similar to how we're currently handling movement, except we're going to do some checks for components beforehand:

{{< highlight go >}}
func SystemMovement(entity *GameEntity, dx, dy int, entities []*GameEntity, gameMap *gamemap.Map, messageLog *ui.MessageLog) {
	// Allow a moveable and controllable entity to move
	if entity.HasComponents([]string{"movement", "controllable", "position"}) {
		// If the current entity is controllable, moveable, and has a position, go ahead and move it
		positionComponent, _ := entity.Components["position"].(PositionComponent)

		if !gameMap.IsBlocked(positionComponent.X+dx, positionComponent.Y+dy) {
			target := GetBlockingEntitiesAtLocation(entities, positionComponent.X+dx, positionComponent.Y+dy)
			if target != nil {
				// SystemAttack(entity, target, messageLog) - this is our Attack system; next time!
			} else {
				positionComponent.X += dx
				positionComponent.Y += dy

				entity.RemoveComponent("position")
				entity.AddComponent("position", positionComponent)
			}
		}
	} else {
		// Implement an AI component here (later)
	}
}
{{< /highlight >}}

First, we're checking for the movement, controllable, and position components. If an entity is missing any of these, we'll pass it off to an AI routine (next post), and let it decide what to do. If the entity is controllable, has movement, and has a position though, we need to handle that. We will have a dx and dy value, which will be passed in from our key handling function, and those will get applied as normal to move the controllable entity. Note, this does not apply just to the player, but any entity that happens to have the controllable component. This means that the player may be able to control multiple entities in a turn.

To use this system, we just need to call it. Back in our main game file, in our main() function:

{{< highlight go >}}
func main () {

    ...

    for {
		blt.Refresh()

		key := blt.Read()

		// Clear each Entity off the screen
		ecs.SystemClear(entities, gameCamera)

		if key != blt.TK_CLOSE {
			if gameTurn == PlayerTurn {
				if player.HasComponents([]string{"movement", "controllable", "position"}) {
					handleInput(key, player)
				}
			}
		} else {
			break
		}

		if gameTurn == MobTurn {
			var newEntities []*ecs.GameEntity
			for _, e := range entities {
				if e != nil {
					if !e.HasComponent("player") {
						ecs.SystemMovement(e, 0, 0, entities, gameMap, &messageLog)
					}
				}
			}
			gameTurn = PlayerTurn
		}

		renderMap()
		ecs.SystemRender(entities, gameCamera, gameMap)
		messageLog.PrintMessages(ViewAreaY, WindowSizeX, WindowSizeY)
	}

	blt.Close()
}
{{< /highlight  >}}

We're now calling our new movement system anywhere an entity may move. In particular, on the players turn, we check to make sure he has all the correct components, and then hand off control to the handleKeys function (which calls SystemMovement, once the movement keys have been resolved). For monsters, we're doing the same, but anticipating they will not be controllable, and thus will hand off control to an AI system (in a later post). We are now using a single unified system for every entity in the game that can move.

And with that, we should have a functional game again (except enemies will not move). The player and all other entities get drawn to the screen, and the player can move around as normal. And the best part, we now have a flexible system for extending our game in the future. This will be a huge asset to us as we continue to add more and more complex features down the road. As usual, you can check out the code for this post in the repo [here](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v0.0.8).

Next time, we're finally going to get to some combat, and give our monsters some proper AI, so that they will seek and out (possibly) destroy the player. Stay tuned, and until then, happy developing!
