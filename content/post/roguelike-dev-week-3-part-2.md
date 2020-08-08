+++
title = "RoguelikeDev Builds a Roguelike, Part 6"
date = 2017-07-23T22:20:22-06:00
description = "Part 6: Preparing for Combat"
draft = false
toc = false
categories = ["technology"]
tags = ["RLDBAR", "go", "programming", "roguelike", "combat"]
+++

Welcome back to my series about building a Roguelike in Go (and following along with RoguelikeDev's dev-along)! Last time, we added a field of vision algorithm to our game, putting the player in the dark, except the immediate area surrounding them. In this installment, we're going to start adding the framework for monsters that inhabit the dark corners of the caverns. In particular, our goal will be to randomly place Game Entities, representing things such as Goblins, Troll, and Orcs, around our generated caverns, make sure they get to take actions, and allow the player to interact with them. We'll be adding proper combat in a later post, but for now, lets add some monsters!

<!-- more -->

A pertinent question that we should ask, before we dive in, is how are we going to represent the monsters that will inhabit the depths of our caverns? You may recall that we have a struct for just such a thing: GameEntity. Our GameEntity struct is exactly what we need to create monsters. GameEntity's sole purpose is to represent 'things' in the game world, whether that be items, chests, NPCs, monsters, or the player. So far, we've only used the GameEntity struct to represent our player (and a short lived NPC), but now we're going to start leveraging it much, much more.

We need to do some small additions to our GameEntity struct before we move forward. We're going to add a name property, as well as property that indicates whether this entity blocks movement (in the exact same way that a Tile does):

{{< highlight go >}}
type GameEntity struct {
	X int
	Y int
	Layer int
	Char string
	Color string
	Name string
	Blocks bool
}
{{< /highlight >}}

Nothing fancy there, just a string and a bool.

Now that we have that additional data attached to our entities, lets put some thought into how we want to go about creating new monster entities, and adding them to our game world. The easiest approach to this, and the one we're going to stick with for the time being, is to just randomly place some created entities around the map. There are a few challenges with this approach however: 1) We don't want to place entities outside the bounds of where the player can go, and 2) we don't want to place entities on top of one another. Lets setup the solution for the second issue first.

What we need is a way to see if there is a blocking entity at a given location. The way this should work is that, whenever we go to place an entity on our map, we will feed the intended coordinates into a function that checks the location of every entity currently placed, and make sure that one of them is not present at the location we are trying to use. This should be pretty easy to do, as we are also going to be keeping a (more or less) global list of entities that are present on the map. A logical place for this code to live in our entity package. This new function doesn't need to be attached to the GameEntity struct, but from an organizational standpoint, it makes sense to put it in the same file:

{{< highlight go >}}
func GetBlockingEntitiesAtLocation(entities []*GameEntity, destinationX, destinationY int) *GameEntity {
	// Return any entities that are at the destination location which would block movement
	for _, e := range entities {
		if e.Blocks && e.X == destinationX && e.Y == destinationY {
			return e
		}
	}
	return nil
}
{{< /highlight >}}

We pass in a slice containing every entity currently present on the map, and the intended (x, y) coordinates for our new entity. We then check to see if there is an entity present, and if it blocks. If it does, we simply return that entity, or nil if nothing is present at the desired location.

Alright, we've now got a way to check valid entity locations, lets set about actually creating and placing a bunch of monster entities on our map!

At this point, I've decided to refactor my previous code a bit. We're going to be dealing with map related functions (generating and placing monsters based on the type of map), so it makes sense that the monster generation and placement code lives in the gamemap package. Previously, I had one file, gamemap.go, which contained all the map logic. I decided, to keep things tidy, to break the gamemap.go file into three files: gamemap.go, which contains the core, common map logic; arena.go, which contains all code related to arena maps; and cavern.go, which contains all logic specific to caverns. You can look [here](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/tree/master/src/gamemap) to see the specifics of how this is now laid out. This is strictly an organizational change, so don't feel obligated to follow my lead, if you don't want to.

Anyways...

We're going to add a new function that will generate and place monsters in our cavern maps (I'm ignoring arenas, since that's more for testing at the moment). Since we're dealing with cavern maps, I've decided to call my function 'populateCavern'. Lets look at the code, and I'll explain it afterwards:

{{< highlight go >}}
func populateCavern(mainCave []*Tile) []*entity.GameEntity {
	// Randomly sprinkle some Orcs, Trolls, and Goblins around the newly created cavern
	var entities []*entity.GameEntity
	var createdEntity *entity.GameEntity

	for i := 0; i < 2; i++ {
		x := 0
		y := 0
		locationFound := false
		for j := 0; j <= 50; j++ {
			// Attempt to find a clear location to create a mob (entity for now)
			pos := rand.Int() % len(mainCave)
			x = mainCave[pos].X
			y = mainCave[pos].Y
			if entity.GetBlockingEntitiesAtLocation(entities, x, y) == nil {
				locationFound = true
				break
			}
		}

		if locationFound {
			chance := rand.Intn(100)
			if chance <= 25 {
				// Create a Troll
				createdEntity = &entity.GameEntity{X: x, Y: y, Layer: 1, Char: "T", Color: "dark green", Blocks: true, Name: "Troll"}
			} else if chance > 25 && chance <= 50 {
				// Create an Orc
				createdEntity = &entity.GameEntity{X: x, Y: y, Layer: 1, Char: "o", Color: "darker green", Blocks: true, Name: "Orc"}
			} else {
				// Create a Goblin
				createdEntity = &entity.GameEntity{X: x, Y: y, Layer: 1, Char: "g", Color: "green", Blocks: true, Name: "Goblin"}
			}

			entities = append(entities, createdEntity)
		} else {
			// No location was found after 50 tries, which means the map is quite full. Stop here and return.
			break
		}
	}

	return entities
}
{{< /highlight >}}

populateCavern() takes a list of Tiles called mainCave. You may remember from the post on map generation that our cavern generation algorithm keeps track of the main cave area used during the final steps of the process. We're going to pass that slice into our populateCavern() function, as it contains a list of all clear, non-wall Tiles in the playable area, which is exactly what we need.

We start off by creating a for loop. This outer loop will determine how many monsters we generate. I've set mine to 2, but you can tweak that value to your liking. Next, we step into an inner loop, that runs no more than 50 times. This loop is going to attempt to find a valid, clear location for our new (yet to be created) monster. The reason for this loop is simple: we are going to randomly generate coordinates within our cavern, and then check if it is a clear (no blocking entity present) location. If it is not, we will iterate, and try again, with a new set of random coordinates. We will keep looping and looking until a valid location is found. If the loop runs for more than 50 iterations, its pretty safe to assume that there are no valid locations left, and the loop should break. This will prevent our game from endlessly looking for a valid location, when one may not even exist.

The location checker loop is pretty straight forward. We grab a random position within the mainCave, and get the (X, Y) values from the Tile at that location. We then use our new GetBlockingEntitiesAtLocation() function, passing in the randomly generated (x, y) coordinates. If that function returns nil, we have found a valid location, and we set the locationFound variable to true, and break out of the loop. If no valid location is found, we try again. If all 50 iterations are used up, the loop will end, with locationFound still having a value of false.

Next, if a location was in fact found, we generate a random number between 1 and 100. This will give us a 25% chance of creating a Troll, a 25% chance of creating an Orc, and a 50% chance of creating a Goblin. We then create the appropriate entity, using our new name and blocks properties.

Finally, we append the newly created entity onto the local entities slice, which will be returned.

We've now got some entities, in valid locations, for our map. Lets quickly look at the code in GenerateCavern() that calls this new function:

{{< highlight go >}}
func (m *Map) GenerateCavern() (int, int, []*entity.GameEntity) {
    ...

  // Populate the cavern with some nasty critters
	entities := populateCavern(mainCave)

	// Finally, choose a starting position for the player within the newly created cave
	pos := rand.Int() % len(mainCave)
	return mainCave[pos].X, mainCave[pos].Y, entities
}
{{< /highlight >}}

You can see I've changed the signature a bit, as GenerateCavern() now returns a slice of GameEntity, alongside the valid coordinates for the player. Beyond that, we simply call the populateCavern() function, and assign the slice of returned entities to the 'entities' variable, and then return that alongside the players (x, y) position.

Finally, lets get those entities drawn to the screen! Back in our main file, in our init() function, we're going to add a small update:

{{< highlight go >}}
func init() {
  ...

  playerX, playerY, mapEntities := gameMap.GenerateCavern()
	player.X = playerX
	player.Y = playerY

	entities = append(entities, mapEntities...)

  ...
}
{{< /highlight >}}

We're now grabbing the returned GameEntity slice from our GenerateCavern() function. Earlier, we created a slice called entities (of which our player is a part) that was intended to contain every entity currently on the map. Here, we are simply appending the mapEntities slice to the (sort of) global entities slice. This will ensure that our entities from our map generation get drawn correctly, along side our player, each turn!

![Monsters on the map](/static/week-3-part-2-monsters-dumb.png)

(You may notice that there is console output on my game. This is the topic of an upcoming SideNote, so stay tuned if that interests you.)

We've successfully generated, placed, and drawn several (not yet) threatening monsters onto our game map. But...they don't really do much, and if the player attempts to interact with them, you'll find you can walk right through them as if they're not even there. These are some cut-rate monsters... Lets fix that!

As it stands right now, the player takes an action (currently only moving is implemented), and then is allowed to take another action, and another, and another, ad nauseum. We need a way to let the monsters take a turn, after the player has done something. Essentially, we are going to consider one action by the player to be his turn. So, the player moving north one Tile, or the player drinking a potion, or resting, these are all actions that would end the players turn. Once the player has acted, we want to let the monsters on the map do something as well. We're going to need some way of keeping track of the current turn, or game state.

Lets do that first. We'll create a couple of new constants, one called PlayerTurn, and one called MobTurn. What we're aiming for is basically an enum that we can reference to check or set the current state of the game, the players turn, or the monsters turn. Since Go doesn't have direct support for an enum type, I'm going to use a language feature called 'iota'. The iota keyword will auto-increment its value each time we call it, meaning that if we call it three times in sequence, each call will have a value of (1 + last_iota_value). This is super handy for creating enum like variable sets:

{{< highlight go >}}
const (
	WindowSizeX = 100
	WindowSizeY = 35
	MapWidth = 100
	MapHeight = 35
	Title = "BearRogue"
	Font = "fonts/UbuntuMono.ttf"
	FontSize = 24
	PlayerTurn = iota
	MobTurn = iota
)
{{< /highlight >}}

Now, I'm going to set a new variable called gameTurn:

{{< highlight go >}}
var (
	player *entity.GameEntity
	entities []*entity.GameEntity
	gameMap *gamemap.Map
	gameCamera *camera.GameCamera
	fieldOfView *fov.FieldOfVision
	gameTurn int
)
{{< /highlight >}}

Finally, in our init() function, we're going to set the gameTurn to the player:

{{< highlight go >}}
func init() {

    ...

    // Set the current turn to the player, so they may act first
	gameTurn = PlayerTurn

    ...
}
{{< /highlight >}}

The gameTurn variable will regularly get flipped from state to state. Once the player takes an action, it flips to MobTurn, and the monsters all get to take one action. Once they are done, it will flip back to PlayerTurn, etc. This sets up a nice turn by turn flow for the game, with every entity getting one action per turn.

Now, in our main game loop, we need to implement that logic. Turns out, its pretty simple:

{{< highlight go >}}
func main() {
    ...

    if key != blt.TK_CLOSE {
			if gameTurn == PlayerTurn {
				handleInput(key, player)
			}
		} else {
			break
		}

		if gameTurn == MobTurn {
			for _, e := range entities {
				if e != player {
					if gameMap.Tiles[e.X][e.Y].Visible {
						// Check to ensure that the entity is visible before allowing it to message the player
						// This will change soon, as entities will act whether the player can see them or not.
						fmt.Printf("The %s waits patiently...", e.Name)
					}
				}
			}
			gameTurn = PlayerTurn
		}
    ...
}
{{< /highlight >}}

When we check for player input, we also check to see if it is in fact the players turn. If it is, we allow the input to be processed. If not, we skip over the action and continue. If it is the players turn, we call handleInput as normal, but there's one small change in there:

{{< highlight go >}}
func handleInput(key int, player *entity.GameEntity) {
    ...

    // Check to ensure that the tile the player is trying to move in to is a valid move (not blocked)
	if !gameMap.IsBlocked(player.X + dx, player.Y + dy) {
		target := entity.GetBlockingEntitiesAtLocation(entities, player.X + dx, player.Y + dy)
		if target != nil {
			fmt.Printf("You harmlessly bump into the %s", target.Name)
		} else {
			player.Move(dx, dy)
		}
	}

	// Switch the game turn to the Mobs turn
	gameTurn = MobTurn
}
{{< /highlight >}}

After we check if the desired location is blocked, we also check to see if there is a blocking entity present. If there is, we stop the movement, similar to how we stop movement into a wall, and print a message letting the player know that they have bumped into something nasty. Finally, we switch the game turn to the monsters.

Back in the main game loop, we check if its the monsters turn. If it is, we loop (naively right now, we'll make this better later) through every entity on the map, being sure to skip the player, and let them take an action. Right now that action consists of printing out that its waiting patiently, but later this will be things like moving, attacking, etc.

And that's it! If you fire up the game now, you should be able to find some monsters, which should act in reaction to your movement, and allow you to harmlessly bump into them.

This has laid the groundwork for our upcoming combat system. We've allowed the player to interact with other entities, a theme we're going to continue to develop, as its one of the core mechanics of the game. We also set up a system to randomly fill our caves and dungeons with all sorts of nasty creatures, just waiting to hassle the player.

As usual, if you want to see the code for this post, you can find it [here](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v0.0.7). Next time, we'll be fleshing out the combat system, and adding some nice GUI components. Until then, happy developing!

[This way to Part 7: ECS Refactor]({{< relref "post/roguelike-dev-part-7.md" >}})
