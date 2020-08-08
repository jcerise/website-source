+++
categories = ["technology"]
date = "2017-07-05T22:14:21-06:00"
description = "A camera and larger maps for our in progress Roguelike"
draft = false 
images = ["https://source.unsplash.com/category/technology/1600x900"]
tags = ["RLDBAR", "go", "programming", "roguelike"]
title = "RLDBAR Sidenote #1: Cameras"
toc = false

+++

This is the first of what I anticipate to be several sidenotes to the main RoguelikeDev Builds a Roguelike shenanigans. These sidenotes will deal with things that are not covered in the main tutorial, but I still feel add value, either to gameplay, programming knowledge, or both. For the inaugural Sidenote, we'll be discussing adding a camera to our in progress roguelike. A camera will give us a couple of benefits, but the most immediate is that we can support maps larger than the terminal window. Impossibly large maps for our player to get lost in sounds great.
<!-- more -->

I'm going to try to keep this short, as its not a large topic, so here we go!

Lets briefly discuss our motivations, and the expected outcomes from what we're about to do. Currently, our player is confined to a world that is no bigger than the screen it is contained within. If you move your player outside the bounds of the screen, it just disappears into an inky black abyss. This is fine if we want to have small levels, and contain the player from going off screen (like we currently do). But, what if we wanted bigger maps? Wouldn't it be super nice to make a gigantic map, and only display the portion that the player can immediately see? Having scrolling maps certainly seems like it would add a lot to our game, in the long term. More content is never a bad thing.

So, what we want is the ability to create maps of an arbitrarily large size. We also want to display only the portion of that large map the player can currently see (so, within the bounds of the game window). We'll need to move the camera to follow the player, and as the player moves through the world, make sure we are only displaying the portion of the map that can currently be seen. Sounds like a lot of work, but in practice, its actually quite simple.

What we're going to do is create a GameCamera object that will always have the player at the center. As the player moves around the map, we will update the camera to keep the player in the center, and then we will draw any GameEntities and Tiles that would currently be displayed, based on the players current position. We will need a way to translate coordinates from the camera to the map, as the map will be much larger than the cameras view area.

Lets get started with creating a new package, called 'camera', that will contain our GameCamera:

{{< highlight go >}}
package camera

type GameCamera struct {
    X int
    Y int
    Width int
    Height int
}
{{< /highlight >}}

Our GameCamera has four basic properties: an (x, y) coordinate set, which will keep track of where the camera currently starts its viewport from, and a Width and Height. Nice and simple. Next, we need a way to "move" the camera around when the player moves:

{{< highlight go >}}
func (c *GameCamera) MoveCamera(targetX int, targetY int, mapWidth int, mapHeight int) {
    // Update the camera coordinates to the target coordinates
    x := targetX - c.Width / 2
	y := targetY - c.Height / 2

	if x < 0 {
		x = 0
	}

	if y < 0 {
		y = 0
	}

	if x > mapWidth - c.Width  {
		x = mapWidth - c.Width
	}

	if y > mapHeight - c.Height {
		y = mapHeight - c.Height
	}

	c.X, c.Y = x, y
}
{{< /highlight >}}

'targetX' and 'targetY' here are the current position of the player. The (x, y) position of our camera is the top left corner of a rectangle (of size width x height) centered on the player. So, in the first two lines, we take half the width and half the height, and subtract the players x and y, respectively. Next, we check for a few edge cases (literally). If the position of the camera is ever beyond the edges of the map (x < 0, y < 0 or x > mapWidth, y > mapHeight), we'll want to bound the camera to the nearest edge. For example, the player has moved to the far edge of the map, and the distance between the player and the edge is less than half the cameras width. In this case, we will stop moving the camera, and the player will appear to move towards the wall, without the game world shifting; the camera is effectively locked on the edge of the world.

Great, we have a way to move the camera. Now, we need a way to translate coordinates from the map to the camera, as the map will be larger than the camera. Lets take a quick look at that:

{{< highlight go >}}
func (c *GameCamera) ToCameraCoordinates(mapX int, mapY int) (cameraX int, cameraY int) {
    // Convert coordinates on the map, to coordinates on the viewport
	x, y := mapX - c.X, mapY - c.Y

	if x < 0 || y < 0 || x >= c.Width || y >= c.Height {
		return -1, -1
	}

	return x, y
}
{{< /highlight >}}

This function takes a set of map coordinates, and translates them to camera coordinates. We accomplish this by simply subtracting the map coordinates from the width and height of the camera, respectively. This will give us a set of coordinates, that we can then test to see if it is within the current view of the camera. If it is not, we'll return (-1, -1), so it will not be drawn to the screen (we'll be calling this function before we draw anything in the game, henceforth).

That about does it for our GameCamera struct. Lets take a look at how to go about using it!

The first thing we'll do is to set our MapWidth and MapHeight consts to 200 each. Because why not? Large maps are fun. Next, we'll need to add a new variable to represent our camera:

{{< highlight go >}}
var (
    ...
    gameCamera *camera.GameCamera
)
{{< /highlight >}}

Then, in our init() function, we need to initialize our new camera:

{{< highlight go >}}
func init() {
    ...
    // Initialize a camera object
    gameCamera = &camera.GameCamera{X: 1, Y:, Width: WindowSizeX, Height: WindowSizeY}
}
{{< /highlight >}}

I've initialized the initial camera position to (1, 1). This won't matter, as we'll quickly be changing that once the player starts moving. I've also, for now, set the width and height of the camera to match the size of the window.

Next up, lets actually implement our camera. In our renderAll() function, add the following line before the rest of the code:

{{< highlight go >}}
gameCamera.MoveCamera(player.X, player.Y, MapWidth, MapHeight)
{{< /highlight >}}

At this point, when we are calling renderAll(), we have already determined if the player has moved, and as such, should have new coordinates for it (or not, in which case we still have the players x, y coordinates regardless). We feed those into the MoveCamera() function, which will update the position of the camera to center the player (if it can; remember, if the player is near the map edge, it will not end up centered). Once we have moved the camera, we can go along rendering as normal.

Now, we've got a bit of house keeping to do. We need to update our GameEntities Draw() and Clear() functions to use camera coordinates, instead of map coordinates. Remember, if a GameEntity is outside of the bounds of the camera, we just won't draw it. Lets modify our renderEntities() function as follows:

{{< highlight go >}}
func renderEntities() {
	// Draw every Entity present in the game. This gets called on each iteration of the game loop.
	for _, e := range entities {
		cameraX, cameraY := gameCamera.ToCameraCoordinates(e.X, e.Y)
		e.Draw(cameraX, cameraY)
	}
}
{{< /highlight >}}

This will give us the coordinates within the cameras view that the GameEntity will appear (or not, if its outside the screen). We'll need to do the same thing before we call Clear() in our main game loop:

{{< highlight go >}}
// Clear each Entity off the screen
for _, e := range entities {
	mapX, mapY := gameCamera.ToCameraCoordinates(e.X, e.Y)
	e.Clear(mapX, mapY)
}
{{< /highlight >}}

Now that we're drawing our entities to the correct locations, lets look at what we need to change to properly draw the map (or rather, the portion of the map the camera can see):

{{< highlight go >}}
func renderMap() {
	// Render the game map. If a tile is blocked and blocks sight, draw a '#', if it is not blocked, and does not block
	// sight, draw a '.'
	for y := 0; y < gameCamera.Height; y++ {
		for x := 0; x < gameCamera.Width; x++ {
			mapX, mapY := gameCamera.X + x, gameCamera.Y + y
			if gameMap.Tiles[mapX][mapY].Blocked == true {
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

As you can see, we're now using the GameCamera as the basis for our drawing loop. We only want to draw things that appear within the bounds of the camera, so we only loop over the width and height of the camera, rather than the map. We then add the camera X and Y to the current x and y to get the actual map tile, and then we render the tile as usual. This works because the GameCamera will be a rectangle centered on the player, so GameCamera.X + x will give us the actual map location, as it simply applies the offset. If the GameCamera is currently at (2, 2), the first time through the loop, mapX and mapY would be (2, 2) (since x and y are 0). The next time around, mapX and mapY would be (3, 2), as we are now at y = 0, x= 1. Hopefully, you can follow that example along and convince yourself of what this is doing (since I'm not very good at explaining it).

Alright, that should be everything. We have a new GameCamera struct, which we are moving around each time the player moves, keeping the player centered inside it. We translate all of our draw calls to coordinates the camera can use, and we only draw the portions of the map that the camera can see. Awesome. If you fire up your game and start moving your player around, you should notice that as soon as the player moves beyond the center of the screen along either axis, the NPC, as well as the walls, will move out of sight slowly. And you should be able to move all 200 spaces to the other side of the room!

The code for this sidenote can be found on [release v0.0.4](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v0.0.4) on my github. Next time, we'll be back on the main tutorial track with dungeon generation! Until then, happy developing!
