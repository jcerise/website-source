+++
title = "RoguelikeDev Builds a Roguelike, Part 5"
date = 2017-07-13T22:30:50-06:00
description = "Part 5: Field of Vision"
draft = false
toc = false
tags = ["RLDBAR", "go", "programming", "roguelike", "field of vision"]
categories = ["technology"]
+++

Welcome to part 5 of RoguelikeDev builds a Roguelike! In this weeks installment, we're going to be talking about field of vision (fov for short), and how we can apply that to our game in progress. Field of vision is how far the player can see. If something is beyond the field of vision of the player, it will not be visible. This adds a nice exploration element to our game, as the game map starts completely unexplored. It also will help add a sense of danger, once we start adding content, as the player will never know whats around the next corner.

<!-- more -->

Lets start off by discussing what field of vision (or view), actually is. Fov is literally the distance that the player can see on our map. If we say that the player has a fov of 6, that would mean, for the purposes of our game, that the player is able to see 6 spaces in every direction, essentially forming a circle of vision around him. As the player moves around the map, the fov moves with him, so that the player can always see Tiles that are 6 spaces away or closer. Obviously, we can tweak the distance the player can see, but for now, I'm going to leave it at a default value of 6.

As the player moves around the map, we also want to keep track of which Tiles have already been explored ('seen' by the player, or within the players fov). The purpose of this is that we want to show the player Tiles that have already been visited, but grayed out. This will create the effect of the player knowing where he has already been, while maintaining the mystery of what could be lurking outside of the fov. You can think of this grayed out area as a fog of war type system. It will help the player by showing the map that has been explored, while also hiding any Entities that may be lurking in already explored areas.

The approach I chose for fov in my game is a technique called ray casting. Ray casting is just that: you cast out a ray for each degree of a circle (360, or fewer, if you want less accuracy), and see what it intersects. If the ray intersects a floor tile, it should keep going (remember, floor tiles do not block movement or vision). If the ray intersects with a Tile that blocks sight (blocks_sight = true), then it should stop, as the player will not be able to see beyond that. Finally, if a ray would intersect the edge of the map, it should also stop.

So, we will cast 360 rays around the player. For each ray, we will add the players current X position to sin(i degrees) and the players current Y position to cos(i degrees), x times, where x is the radius of the players fov (in this case 6). Each step, we check for collision with a Tile that blocks sight. If one is not encountered, we will mark the Tile as explored and visible, and move on, until either the ray passes beyond the players fov, or a wall/other blocking Tile is encountered.

Once we are done deciding what is in the players fov, we will clear the map, and redraw everything, with Tiles in the fov being white, and explored Tiles being gray. Everything else will not get drawn, since it has not yet been explored.

Alright, that's enough talking about it, lets actually code it up!

We're going to start (as usual), by creating a new package, called fov. In that package, we're going to define a struct, called FieldOfVision:

{{< highlight go >}}
type FieldOfVision struct {
    sinTable map[int]float64
    cosTable map[int]float64
    torchRadius int
}
{{< /highlight >}}

The sinTable and cosTable maps will contain our pre-calculated values for sin and cos, which we'll see how to initialize in a minute. TorchRadius will be the radius (distance) that the player can see around him. We're going to add a setter for torchRadius, so future gameplay elements can modify how far the player can see (like a +10 Rod of Blazing Sunlight, for example). Now that we have our struct, lets attach the aforementioned setter method for torchRadius, and an initialize function to it:

{{< highlight go >}}
func (f *FieldOfVision) Initialize() {

	f.cosTable = make(map[int]float64)
	f.sinTable = make(map[int]float64)

	for i := 0; i < 360; i++ {
		ax := math.Sin(float64(i) / (float64(180) / math.Pi))
		ay := math.Cos(float64(i) / (float64(180) / math.Pi))

		f.sinTable[i] = ax
		f.cosTable[i] = ay
	}
}

func (f *FieldOfVision) SetTorchRadius(radius int) {
	if radius > 1 {
		f.torchRadius = radius
	}
}
{{< /highlight >}}

Our Initialize() function serves to set up the pre-calculated values for our sin and cos maps, as discussed earlier. We simply calculate one value for each degree of a circle, to be sin(i / 180 / PI) and cos(i / 180 / PI). This will give us an easy way to cast our rays out from the player in increments of 1 degree later on.

The TorchRadius() function is just a simple setter for torchRadius, but it never lets the value go below 1. We don't the player to be left completely in the dark. Yet, anyways.

Before we move on, we need to add a couple of properties to our Tile struct:

{{< highlight go >}}
type Tile struct {
	Blocked bool
	Blocks_sight bool
	Visited bool
	Explored bool
	Visible bool
	X int
	Y int
}
{{< /highlight >}}

Pretty simple, we just added a Explored and Visible bool, which should be pretty self explanatory. If not, we'll be jumping into their uses momentarily.

Alright, now that we have initialization out of the way, lets get into the meat of the fov code, the actual ray casting. This function will not only handle casting of rays, it will also be responsible for marking a Tile as explored or visible. I'm gonna dump the whole function here, and then explain it step by step afterwards.

{{< highlight go >}}
func (f *FieldOfVision) RayCast(playerX, playerY int, gameMap *gamemap.Map) {
	// Cast out rays each degree in a 360 circle from the player. If a ray passes over a floor (does not block sight)
	// tile, keep going, up to the maximum torch radius (view radius) of the player. If the ray intersects a wall
	// (blocks sight), stop, as the player will not be able to see past that. Every visible tile will get the Visible
	// and Explored properties set to true.

	for i := 0; i < 360; i ++ {

		ax := f.sinTable[i]
		ay := f.cosTable[i]

		x := float64(playerX)
		y := float64(playerY)

		// Mark the players current position as explored
		gameMap.Tiles[playerX][playerY].Explored = true

		for j := 0; j < f.torchRadius; j++ {
			x -= ax
			y -= ay

            roundedX := int(Round(x))
            roundedY := int(Round(y))

			if x < 0 || x > float64(gameMap.Width - 1) || y < 0 || y > float64(gameMap.Height - 1) {
				// If the ray is cast outside of the map, stop
				break
			}

			gameMap.Tiles[roundedX][roundedY].Explored = true
			gameMap.Tiles[roundedX][roundedY].Visible = true

			if gameMap.Tiles[roundedX][roudnedY].Blocks_sight == true {
				// The ray hit a wall, go no further
				break
			}
		}
	}
}

func Round(f float64) float64 {
    return math.Floor(f + .5)
}
{{< /highlight >}}

To start with, our RayCast() function takes three arguments, the current player X and Y position, and a pointer to the current GameMap. The players X and Y are the origin for our ray casting, and we'll use the GameMap's Tiles property to keep track of which tiles are visible and explored (thus why its a pointer).

We start things off by setting up a loop that will iterate 360 times. This can be tweaked if we want less accuracy, by incrementing i by +2, or +3, etc. This will be more efficient, but the fov will not be as accurate. Something to play with later. We then grab the value for this degree from our pre-calculated sin and cos tables, and then convert the players X and Y into float64, as that's the type of our sin and cos values. Lastly, we set the Tile that the player currently occupies as explored.

Next, we iterate x times, where x is the torchRadius (in our case, 6). This is where we will actually begin casting the rays from the player (origin). We subtract the sin value from player.X and the cos value from player.Y. This will give us an approximation of one tile away, in the direction of whatever degree we are currently calculating. We then check to make sure that Tile is not outside of the map (if it is, quit early, as the edge of the map technically blocks sight). Then, we mark the Tile as both explored and visible. If it is a wall, it will be illuminated, but nothing beyond it will be visible. If its floor, it will be illuminated, and the Tiles beyond will be visible as well. Finally, we check to see if the Tile blocks sight. If it does, we'll break out of the loop, as nothing beyond the wall Tile is visible.

**[Update: 7/17/17]** As a note, I was originally just casting my float64s back to ints, which seemed to generally work alright, but as user VedVid on Reddit pointed out, Go will just remove the precision when casting this way, so 5.999999999 becomes 5. Obviously, this throws out a lot of accuracy in how we are getting the tiles that are visible. So, I added a very simple little round function, that will at least give us the correct rounded number when we cast back to an int. Round(5.9999999) will return 6, whereas Round(5.23457) will return 5, as we would expect. This should result in much more accurate fov caclulations. Thanks, VedVid!

That's it, we're basically just checking 360 degrees around the player, and marking visible tiles as visible and explored. Tiles that are neither visible nor explored will not be displayed to the player.

Cool, so now lets actually use our new FieldOfView struct. Back in our main file, we'll need to initialize a new FieldOfView object:

{{< highlight go >}}
var (
    ...
    fieldOfVision *fov.FieldOfVision
)

func init() {
    ...

    fieldOfVision = &fov.FieldOfVision{}
    fieldOfVision.Initialize()
    fieldOfVision.SetTorchRadius(6)
}
{{< /highlight >}}  

Once we have our fieldOfVision var initialized and the torchRadius set, we're ready to start using it. We're going to be altering our renderMap() function:

{{< highlight go >}}
func renderMap() {
	// Render the game map. If a tile is blocked and blocks sight, draw a '#', if it is not blocked, and does not block
	// sight, draw a '.'

	// First, set the entire map to not visible. We'll decide what is visible based on the torch radius.
	// In the process, clear every Tile on the map as well
	for x := 0; x < gameMap.Width; x++ {
		for y := 0; y < gameMap.Height; y++ {
			gameMap.Tiles[x][y].Visible = false
			blt.Print(x, y, " ")
		}
	}

	// Next figure out what is visible to the player, and what is not.
	fieldOfView.RayCast(player.X, player.Y, gameMap)

	// Now draw each tile that should appear on the screen, if its visible, or explored
	for x := 0; x < gameCamera.Width; x++ {
		for y := 0; y < gameCamera.Height; y++ {
			mapX, mapY := gameCamera.X + x, gameCamera.Y + y

			if gameMap.Tiles[mapX][mapY].Visible {
				if gameMap.Tiles[mapX][mapY].IsWall() {
					blt.Color(blt.ColorFromName("white"))
					blt.Print(x, y, "#")
				} else {
					blt.Color(blt.ColorFromName("white"))
					blt.Print(x, y, ".")
				}
			} else if gameMap.Tiles[mapX][mapY].Explored {
				if gameMap.Tiles[mapX][mapY].IsWall() {
					blt.Color(blt.ColorFromName("gray"))
					blt.Print(x, y, "#")
				} else {
					blt.Color(blt.ColorFromName("gray"))
					blt.Print(x, y, ".")
				}
			}
		}
	}
}
{{< /highlight >}}

The very first thing we do, is clear every single Tile off the visible map. We do this by setting its visible property to false, and replacing it with a ' ' character. This is necessary, as every iteration through the game loop, we will need to recalculate our fov (this is not entirely true, and a good spot for some future improvement; only recalculate the fov when the player has moved, for example. Our current approach is a bit naive). Recalculating the fov will set a new batch of Tiles as visible, and if we didn't clear the old ones, we would have a line of visible Tiles behind the character as movement occurred, and also get some strange artifacts when the map is drawn.

After we clear the map, we actually recalculate the fov. We do this simply by calling our RayCast() function, which will not only cast our rays, but also mark every tile that is visible and explored for us.

From there, its just a matter of drawing out all the Tiles on the map that are either explored, or visible. As I stated earlier, visible Tiles will be "illuminated" by being drawn white, where as Tiles that have been explored, but are not currently in the players fov (not visible), will be grayed out, and drawn gray. This gives a nice effect of the player knowing whats out where he has explored, but not being able to see it anymore. You'll notice we have no case for if a Tile is not explored, nor visible. These tiles simply do not get drawn.

Lets fire up the game! You should see something looks a bit like this:

![Field of Vision 1](/static/fov-1.png)

And, after you move around a bit and explore:

![Field of Vision 2](/static/fov-2.png)

Very nice! Its pretty clear where you've been, and what you can currently see. And its also pretty clear what hasn't been explored yet!

One last thing we need to take care of, in preparation for the next part (adding monsters), is making sure that any entities other than the player are not visible, unless they are within the players field of vision (like our hapless, red NPC in the far left corner of the map). To do this, we simply need to check if the Tile they occupy is visible. If its not, don't draw it:

{{< highlight go >}}
func renderEntities() {
	// Draw every Entity present in the game. This gets called on each iteration of the game loop.
	for _, e := range entities {
		cameraX, cameraY := gameCamera.ToCameraCoordinates(e.X, e.Y)
		if gameMap.Tiles[e.X][e.Y].Visible {
			e.Draw(cameraX, cameraY)
		}
	}
}
{{< /highlight >}}

Easy. Now any entities outside the line of sight will not be drawn. The player will have to hunt them down to see them (or wait until they hunt him down...).

And that wraps up Field of Vision. Our game is getting closer and closer to actually being fun, at this point. We've added some element of exploration by making sure the player can only see a small area around itself, and made sure that already explored Tiles are duly documented as the player delves ever deeper. If you would like to see the source for this post, you can view it on my [repo](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v.0.0.6) under the v0.0.6 release.

Join me next time, when we will start adding monsters and such to the dungeon, and outline the framework for combat! Until then, happy developing.

[This way to part 6]({{< relref "post/roguelike-dev-week-3-part-2.md" >}})
