+++
categories = ["technology"]
date = "2017-07-02T20:25:13-06:00"
description = "Part 3: Game Entities and the Map"
draft = false
tags = ["RLDBAR", "go", "programming", "roguelike"]
title = "RoguelikeDev Builds A Roguelike, Part 3"
toc = false

+++
Welcome back to my series following along with RoguelikeDev Builds a Roguelike! This is the second part of this weeks posts, the first being concerned with creating a player representation and moving it around using player input. In the second part of this week, we're going to start thinking about the overall structure of our game, as well as get an intial version of the gameplay map, which is the world the player will be interacting with. We'll accomplish this using the idea of an 'Entity', and applying that to (almost) eveverything in our game.

Lets go!
<!-- more -->

If you recall from last time, we were creating a set of player coordinates, playerX, and playerY, and the applying those to a function, drawPlayer(), which would, when given the coordinates and a character to represent the player, draw the player to the screen. We could then update those coordinates to move the player around. This worked fine, but lets consider a few things, primarily, how do we go about drawing a large number of different things to the screen? Enemies, items, traps, flora, fauna, etc? Surely, we could (extremely) naively create variables for each of them, and draw them out one by one like we do for the player, but that sounds awful. What we really need is some way to represent a generic "thing" in our game. Every "thing" has some common properties, like a location, a character, and a color. So, what we really want to do, is make every thing the player sees and interacts with (including the player!) a "thing".

Henceforth, I'm going to use the term Entity to refer to anything in our game (with a few exceptions we'll get to later in this post). An Entity will have an (x, y) location, a color, and a character. This will allow us to create as many Entities as we want, to represent anything in our game. The player is a white @, an orc could be a dark green O, a purple dragon could be a purple D, a scroll of fireball a beige |, and so on and so forth. Using the idea of Entities allows us a great deal of flexibility. I could go on about object oriented design here, but these posts are not meant to deal with that, so go read up on it if you're curious.

Okay, so our plan is to create an Entity representation, and make everything in our game an Entity. Lets see how to do that. Before I get into the code and specifics, I would like to say that this is going to be one of many "Great Refactors" throughout the course of this series. Refactoring your code from time to time is healthy and natural, and the ladies love it. Anyways...

Go does not have a traditional Class representation that many other object oriented languages have. This was one of the first major differences in Go that I had to wrap my head around. Lacking classes, though, Go does provide us two very powerful things: structs, and interfaces. We'll get into interfaces later, and for now we're going to focus on structs. For all intents and purposes (read: as far as I have found), a struct in Go is more or less parallel to the class keyword you may be familiar with in other languages. It allows you to define fields for the given struct, and then associate methods with that struct. Sounds like a class with a different name. For our purposes, it provides exactly what we need: a way to encapsulate some functionality in a reusable way. Enough talking, lets write some code.

I created a new package at this point, as I like to have tidy code; you are more than welcome to include this in your main game file, either way works. In my new package, entity, I created a file called entity.go. The first thing we'll need to do is create our struct:

{{< highlight go >}}
type GameEntity struct {
    X int
    Y int
    Layer int
    Char string
    color string
}
{{< /highlight >}}

That's it. Pretty simple. You may notice that each member variable starts with a capital letter. Good eye. In Go, to make something public (accessible outside of the scope where it was defined), you begin the name with a capital letter. To make it private (accessible only within its defining scope), you begin its name with a lowercase letter. I chuckled to myself when I learned this, because its wonderfully simple, and easy to read at a glance. I like Go for small details like this. Anyways, we'll need to access these variables outside of the scope of the struct, so I made them all public (for now). X and Y will represent the location of the Entity, Layer will represent the current rendering layer for BearLibTerminal (more on that later), Char is the character we want to visually represent the Entity, and Color is what color it will be displayed as. Pretty straight forward stuff.

Now that we have our struct representing our Entity, lets make sure it has feature parity with our current player code. To do that, we'll need some way to draw it to the screen, as well as move it around. For good measure, we should also have a method to clear it from the screen (in case it dies, is destroyed, turns invisible, or we just need to re-draw the screen). We're going to create three functions, and associate them with our Entity struct:

{{< highlight go >}}
func (e *GameEntity) Move(dx int, dy int) {
    // Move the Entity by the amount (dx, dy)
    e.X += dx
    e.Y += dy
}

func (e *GameEntity) Draw() {
    // Draw the Entity to the screen
    blt.layer(e.Layer)
    blt.Color(blt.ColorFromName(e.Color))
    blt.Print(e.X, e.Y, e.Char)
}

func (e *GameEntity) Clear() {
    // Remove the entity from the screen
    blt.Layer(e.Layer)
    blt.Print(e.X, e.Y, " ")
}
{{< /highlight >}}

Lets start with Move() (again, notice the uppercase first letter, meaning its a public method). Our function declaration looks a little different than others we've seen in the past, mainly due to the

{{< highlight go >}}
(e *GameEntity)
{{< /highlight >}}

bit that's hanging out in the signature. This is how we "associate" a function with a struct. We're basically saying that the function expects to have access to a variable named "e", which is of type GameEntity. The closest comparison I can draw to this is Pythons 'self', if you're familiar with how that works. It says, this method is to be used on the struct type I specify. This method can then be called using dot notation, with the object calling being passed in as "e". Not too deviant. Anyways, on to the Move() function.

Move() takes two arguments, dx, and dy. These represent the change in X and Y, respectively (deltaX, and deltaY is another way to think of it). We take these deltas, and apply them to the Entities X and Y, thus altering its position. If we pass in dx: 0, dy: -1, we're moving the Entity up the screen by one (Since our coordinate system has (0,0) in the top left corner). If we pass in dx: 1, dy: 0, we're moving the Entity to the right by one, etc. This is a simple and effective way to move our Entity around the screen.

Next up is Draw(). This code should look familiar from our drawPlayer() function last time; its the same code, with a few minor changes. We're using our struct ("e") instead of hardcoding values. It sets the Layer (there's that pesky Layer thing again, I'm going to have explain it at some point...but that's a problem for future me), then sets the color, both from the structs values, and finally, calls Print() to print the whole thing to screen.

Finally, Clear(). Again, it sets the Layer, and then simply replaces the character on the terminal with an empty space. This effectively "erases" the Entity from the screen. We'll see in a bit how this will be handy.

Alright, that does it for our GameEntity struct! Now, lets see how we can use it.

The first thing we need to do is import our new package (skip this if you didn't create a new package for our Entity struct):

{{< highlight go >}}
import (
    blt "bearlibterminal"
    "strconv"
    "entity"
)
{{< /highlight >}}

Next, lets define a couple of handy variables:

{{< highlight go >}}
var (
    player *entity.GameEntity
    entities []*entity.GameEntity
)
{{< /highlight >}}

We're defining two variables, player and entities. Player will, predictably, keep track of our player Entity. Entities will be a slice (Go for ArrayList) of type GameEntity (Entity, since I can't keep my references in this blog post straight, sorry about that). You may also notice a '*' in front of the struct type. In Go, that denotes that we are making these as pointers to the type, in this case entity.GameEntity. Pointers are beyond the scope of this series, but I'll do my best to explain why we're doing this as I go.

Now, lets modify our init() function to create a couple of entities:

{{< highlight go >}}
func init() {
    ...

    // Create a player Entity, and an NPC Entity, and add them to our slice of Entities
    player = &entity.GameEntity{X: 1, Y: 1, Layer: 1, Char: "@", Color: "white"}
    npc := &entity.GameEntity{X: 10, Y: 10, Layer: 0, Char: "N", Color: "red"}
    entities = append(entities, player, npc)
}
{{< /highlight >}}

The player variable gets assigned to an Entity, and we create a new variable called NPC to represent a second entity. Again, there's a bit of new notation in these variable assignments, the '&' in front of the name. This denotes that we are going to use the value assigned by reference, rather than by value. Basically, a pointer at the object created. The upshot of this is that, later in the code, when we modify something on the player Entity, it will be updated on the original player, in the way we expect. If we had created this variable by value, we would have gotten a copy assigned, and trying to modify it later would modify the copy, rather than the original, not what we want. Pointers 101 dismissed.

 The last step here is to append both the player and NPC Entities to our entities slice, so we can keep track of them. Because we created them by reference, the variables in the slice will point to our original objects.

Three steps left: create a function to draw all our of entities to the screen at once, update our movement controls to use our new Entity struct, and update our game loop to in the same manner. Lets start by creating a function to draw all our entities to the screen:

{{< highlight go >}}
func renderEntities() {
    // Draw every Entity present in the game. This gets called on each iteration of the game loop.
    for _, e := range entities {
        e.Draw()
    }
}
{{< /highlight >}}

Because we have a central Draw() method with our struct, we can simply iterate over the entities slice, and call Draw() on each one. This will draw the Entity to the screen at its current (X, Y) position. Nice! As a side note, this is Go's version of a for each loop. We're saying use a blank placeholder (_) for the index, since we don't care about that, and assign each Entity in entities to e.

Next, lets update our movement controls to use our Entity struct:

{{< highlight go >}}
func handleInput(key int) {
    // Handle basic character movement in the four main directions

    var (
        dx, dy int
    )

    switch key {
    case blt.TK_RIGHT:
        dx, dy = 1, 0
    case blt.TK_LEFT:
        dx, dy = -1, 0
    case blt.TK_UP:
        dx, dy = 0, -1
    case blt.TK_DOWN:
        dx, dy = 0, 1
    }

    player.Move(dx, dy)
}
{{< /highlight >}}

We set a couple of ints, dx and dy, which should look familiar from our Move() method on our struct, and then we assign what they should be based on the key input (the logic is more or less the same as before). Finally, we call player.Move(), with the new values. Done.

Finally, lets tweak our main game loop to put all of this together:

{{< highlight go >}}
func main() {
    // Main game loop

    renderEntities()

    for {
        blt.Refresh()

        key := blt.Read()

        // Clear each entity off the screen, so we can re-draw them
        for _, e := range entities {
            e.Clear()
        }

        if key != blt.TK_CLOSE {
            handleInput(key)
        } else {
            break
        }

        // Re-draw all of our Entities, since they may have moved
        renderEntities()
    }

    blt.Close()
}
{{< /highlight >}}

First up, we draw every Entity to the screen (this will initialize our game screen when the game first starts). Then we go into our main game loop, which refreshes the screen, and checks if any input was used. Then, we clear all the Entities, since they may be about to move. Finally, we handle any input, and then re-draw everything in its new (maybe) positions.

Whew! That was a lot of work, but at this point, we have a working generic implementation that will allow us to create as many Entities as we want, and draw them all to the screen. Pat yourself on the back, that was no mean feat.

Now, lets create a world for these Entities to reside in.

The first thing I'm going to do is create a new package (yes, I like packages) called 'gamemap'. This is going to contain our world building logic. Inside this package, lets create a file called gamemap.go. In this file, lets define a new struct, called Tile:

{{< highlight go >}}
type Tile struct {
    Blocked bool
    BlocksSight bool
}
{{< /highlight >}}

This struct will represent a single tile in our map (which, if you haven't guessed yet, is a grid of tiles). It has two properties we care about for a map tile: whether the Tile blocks movement, and whether the Tile blocks sight. A floor Tile should block neither sight nor movement, as Entities should be able to move about on them. A wall Tile should block both sight and movement, as you can't more, nor see, through walls. A chasm Tile might block movement, but not sight, and a cloud of smoke might block sight but not movement.

Next, lets create another struct, called GameMap:

{{< highlight go >}}
type GameMap struct {
    Width int
    Height int
    Tiles [][]*Tile
}
{{< /highlight >}}

Our GameMap will keep track of how wide and tall it is, as well as containing a 2D array of Tiles. Since we are working on a 2D grid, a 2D array makes perfect sense for storing our map data. The outer array will represent the x coordinates, and the inner, the y. It will be easy to access a given Tile inside the map array like map[x][y].

We're going to define two functions to attach to the GameMap struct, one to initialize it, and one to check if a tile blocks movement or not. The first, initialization, will be a temporary bit of logic, to make sure everything is working, and give us something quick to play with. We're going to outline the screen with wall Tiles, and fill in the interior with floor Tiles. This will make a large, screen sized room for the player to roam around in, and will nicely exemplify our movement blocking mechanics.

{{< highlight go >}}
func (m *GameMap) InitializeMap() {
    // Set up a map where all the border (edge) Tiles are walls (block movement, and sight)
	// This is just a test method, we will build maps more dynamically in the future.
	m.Tiles = make([][]*Tile, m.Width)
	for i := range m.Tiles {
		m.Tiles[i] = make([]*Tile, m.Height)
	}

	for x := 0; x < m.Width; x++ {
		for y := 0; y < m.Height; y++ {
			if x == 0 || x == m.Width- 1 || y == 0 || y == m.Height- 1 {
				m.Tiles[x][y] = &Tile{true, true}
			} else {
				m.Tiles[x][y] = &Tile{false, false}
			}
		}
	}
}
{{< /highlight >}}

We're using Go's built in Make to initialize our Tile array, with an outer length of the Width of the map. Then, we begin to iterate over each outer cell. In each X cell, we create a new array of length Height. This gives us a fully initialized 2D array of size (Width x Height). We then iterate over each cell in the 2D array, and check two things: Is x or y equal to zero, or is x equal to the width of the map - 1 or y equal to the height of the map -1. If it is, we know we are on an edge (either the right or left edge, or the top or bottom). If this is true, we want this Tile to be a wall Tile, so we create a new Tile that both blocks sight and movement. If this check fails, we create a floor Tile (does not block sight nor movement).

At the end of this method, we should have a (Width x Height) 2D array that is filled with wall and floor Tiles.

Next, lets write a function to check if a given Tile is blocked:

{{< highlight go >}}
func (m *Map) IsBlocked(x int, y int) bool {
	// Check to see if the provided coordinates contain a blocked tile
	if m.Tiles[x][y].Blocked {
		return true
	} else {
		return false
	}
}
{{< /highlight >}}

This function simply takes an (x ,y) coordinate pair, and returns if the Tile, at that coordinate in the GameMap, has the Blocked property set to true. We'll use this a little later to determine where the player can and can't move.

Alright, that should do it for our gamemap package! Now lets go implement what we have in the rest of our program.

In our main file, we need to: import our gamemap package (skip if you did not create a package), create a new variable to represent and track our GameMap, initialize our map, and finally, draw our map to the screen. Sounds like a lot of work, lets get to it.

First, import our package:

{{< highlight go >}}
import (
	blt "bearlibterminal"
	"strconv"
	"entity"
	"gamemap"
)
{{< /highlight >}}

We also need to add a couple of constants to set our map width and height. For now we'll just use the window width and height, but in the future, our maps will be much larger than the window:

{{< highlight go >}}
const (
	WindowSizeX = 100
	WindowSizeY = 35
	MapWidth = WindowSizeX
	MapHeight = WindowSizeY
	Title = "BearRogue"
	Font = "fonts/UbuntuMono.ttf"
	FontSize = 24
)
{{< /highlight >}}

Next, we need to add a new var that will store our GameMap:

{{< highlight go >}}
var (
	player *entity.GameEntity
	entities []*entity.GameEntity
	gameMap *gamemap.Map
)
{{< /highlight >}}

Then, we need to set and initialize our new gameMap variable in our init() function:

{{< highlight go >}}
func init() {
    ...
    // Create a GameMap, and initialize it
    gameMap = &gamemap.Map{Width: MapWidth, Height: MapHeight}
    gameMap.InitializeMap()
}
{{< /highlight >}}

Now that we have that done, we need a way to actually draw our map to the screen. We're going to create a new function, renderMap, that will iterate over each Tile in our GameMap, and Print it to the screen. This will happen before we render our Entities, so the entities are rendered on top of the map. This is where we actually assign characters to the wall ("#") and floor (".") Tiles. I will also be setting a color for each one:

{{< highlight go >}}
func renderMap() {
    // Render the game map. If a tile is blocked and blocks sight, draw a '#', if it is not blocked, and does not block
    // sight, draw a '.'
    for x := 0; x < gameMap.Width; x++ {
    	for y := 0; y < gameMap.Height; y++ {
    		if gameMap.Tiles[x][y].Blocked == true {
    			blt.Color(blt.ColorFromName("gray"))
    			blt.Print(x, y, "#")
    		} else {
    			blt.Color(blt.ColorFromName("brown"))
    			blt.Print(x, y, ".")
    		}
    	}
    }
}
{{< /highlight >}}

This function is pretty simple. All it does is loop over each Tile in the 2D array, and check the Blocked method from the GameMap struct to see if the Tile is a floor or Wall. It then prints the appropriate symbol and color.

Great, so now we have initialized our map, and can draw it. Whats left? Well, actually drawing the map! I'm going to create a new, simple convenience method to render everything (GameMap and Entities) at once, so we only have to call one function in our game loop, instead of two. Clean code is happy code.

{{< highlight go >}}
func renderAll() {
	// Convenience function to render all entities, followed by rendering the game map
	renderMap()
	renderEntities()
}
{{< /highlight >}}

As discussed above, renderAll() calls renderMap() first, followed by renderEntities(), as we want the Entities rendered on top of the map. Now, we just modify our game loop to include this method:

{{< highlight go >}}
func main() {
	// Main game loop

	renderAll()

	for {
		blt.Refresh()

		key := blt.Read()

		// Clear each Entity off the screen
		for _, e := range entities {
			e.Clear()
		}

		if key != blt.TK_CLOSE {
			handleInput(key, player)
		} else {
			break
		}
		renderAll()
	}

	blt.Close()
}
{{< /highlight >}}

Notice that all we changed was swapping out the renderEntities() calls for renderAll() calls. This means that we re-draw the entire map and Entity list every time through the game loop.

Done and done! If you run this now, you should see what resembles a large room, with our '@' player (movable, of course), and an (for now) immobile NPC. You should be able to bump into the walls, but not move through, and move freely across the floor.

We accomplished quite a bit in this post, and our "game" is starting to actually resemble something playable at this point. To recap, we added an Entity struct to represent every object in the game (except map objects, which are represented by a Tile struct). We refactored our main game code to use this new struct by making our player, plus a new NPC,  as Entities. Finally, we created a GameMap struct which stores information about the map the player moves around on, and initialized it to an empty room, making sure that the player cannot pull a magic act and move through walls (yet...). As usual, you can find all the code for this post under the v0.0.3 release on my github [repo](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v0.0.3).

Join me next week when we'll talk about fleshing out our map to make it more...dungeony. Until then, happy developing!

[This way to Part 4]({{< relref "post/roguelike-dev-week-2.md" >}})

=====

Sidenote for this post: [Sidenote 1: Cameras]({{< relref "post/sidenote-1-cameras.md" >}})
