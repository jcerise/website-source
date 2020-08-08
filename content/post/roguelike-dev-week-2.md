+++
categories = ["technology"]
date = "2017-07-11T13:41:06-06:00"
description = "Part 4: A more interesting map"
draft = false
tags = ["RLDBAR", "go", "programming", "roguelike", "cellular automata"]
title = "RoguelikeDev Builds a Roguelike, Part 4"
toc = false

+++

Welcome to Part 4 of RoguelikeDev builds a Roguelike! This week, we're going to be fleshing out the map that the player will be playing the game on. If you recall, last time, we created an 'arena' sort of map, with a large open space in the middle, surrounded by walls along the outside edges. Not very exciting, but it was something for the player to mover around in. This week, we're going to procedurally generate some (hopefully) realistic looking caverns that the player can explore. Certainly more exciting than a big empty room!

<!-- more -->

This week is all about making an interesting and dynamic map that the player can explore. One of the core tenants of Roguelikes is procedurally generated content, and one area where that is more noticeable is the game map. Every time the player plays our game, it would be awesome if their experience was different. We could certainly hardcode up some maps, and to be sure, there is value in that. But the real fun lies in random levels each and every time the game is played. Keeping true to this, we are going to code up a map generator that will be used whenever we need a new level of our dungeons (you can also extrapolate this concept out to any other map representation, overworld, buildings, etc). I've decided that, rather than use the standard dungeon generator that the official tutorial uses, we're going to use a cellular automata algorithm to generate some nice looking caverns. I'll probably do the standard dungeon generator as a sidenote in the future.

First things first, before we dive into the meat of things, there are some convenience changes we'll want to make, to make our cartographical lives easier. First, I'm going to add an (x, y) coordinate set to our Tile struct, and then I'm going to add a couple methods that we will leverage quite a bit later down the line:

{{< highlight go >}}
type Tile struct {
    Blocked bool
    Blocks_sight bool
    Visited bool
    X int
    Y int
}

func (t *Tile) isWall() bool {
    if t.Blocks_sight && t.Blocked {
        return true
    } else {
        return false
    }
}

func (t *Tile) is Visited() bool {
    return t.Visited
}
{{< /highlight >}}

Aside from adding an (x, y) set to the Tile, I've also added a Visited property. This will come in handy later when we need to check if we have already dealt with a Tile during map generation. The two convenience methods should be pretty self explanatory in their uses.

Alright, with that out of the way, lets do a quick refactor of some of our GameMap code:

{{< highlight go >}}
func (m *Map) InitializeMap() {
	// Initialize a two dimensional array that will represent the current game map (of dimensions Width x Height)
	m.Tiles = make([][]*Tile, m.Width)
	for i := range m.Tiles {
		m.Tiles[i] = make([]*Tile, m.Height)
	}

	// Set a seed for procedural generation
	rand.Seed( time.Now().UTC().UnixNano())
}

func (m *Map) GenerateArena() {
	// Generates a large, empty room, with walls ringing the outside edges
	for x := 0; x < m.Width; x++ {
		for y := 0; y < m.Height; y++ {
			if x == 0 || x == m.Width- 1 || y == 0 || y == m.Height- 1 {
				m.Tiles[x][y] = &Tile{true, true, false, x, y}
			} else {
				m.Tiles[x][y] = &Tile{false, false, false, x, y}
			}
		}
	}
}
{{< /highlight >}}

As you can see, I broke out the actual map generation code from the initialize logic. Initialize now just creates an empty 2D array of Tiles (though none of the Tiles have been initialized yet, or even created). It also sets a seed for our randomization. More on that later.

Next, I broke our old map generation code into a new method, called GenerateArena(). This can be called on an initialized GameMap, and will create our familiar large room surrounded by walls. I'm keeping this code around in case we want to create a map like this in the future.

Alright, with the little stuff out of the way, we've paved the path forward to creating a new map generator. I'm going to create a new function, attached to the Map struct, called GenerateCavern (public, as we'll want to call it outside of our map package):

{{< highlight go >}}
func (m *Map) GenerateCavern() (int, int) {}
{{< /highlight >}}

The two ints that get returned will be a valid starting location for the player, so we can be sure it spawns in a valid location.

We're going to be using whats known as a cellular automata to generate our cave. Cellular automata are most famous from Conways game of Life. I'm not going to get into that here, so Google it if you're curious. A cellular automata is essentially a way of representing "cells" or in our case, Tiles in the map. We will start by generating a random assortment of wall and floor Tiles. Then, we will check each Tile, and see if it should stay in its current state, or transition to the other state. In a traditional cellular automata, this would be called the cell living or dying, but we're going to frame as the Tile being a floor or a wall. We'll set our rules for each Tiles state such that if a wall tile is all by itself (no neighbors that are walls), perhaps it should not be a wall, and if a Tile is surrounded by walls, perhaps it too, should be a wall. This probably sounds a little esoteric, so lets just jump in, and I'll explain as we go (with pretty pictures!).

The first thing we need to do is fill up our empty 2D GameMap with a random assortment of Tiles:

{{< highlight go >}}
// Step 1: Fill the map space with a random assortment of walls and floors. This uses a roughly 40/60 ratio in favor
// of floors, as I've found that to produce the nicest results.
for x := 0; x < m.Width; x++ {
	for y := 0; y < m.Height; y++ {
		state := rand.Intn(100)
		if state < 50 {
			m.Tiles[x][y] = &Tile{true, true, false, x, y}
		} else {
			m.Tiles[x][y] = &Tile{false, false, false, x, y}
		}
	}
}
{{< /highlight >}}

All this does, is loop over every element of the GameMap array, generate a random number between 0 and 100, and then give a 50% chance of creating a wall or floor Tile. The 50% is a very tweak-able percentage here, and I encourage you to play with it and see how it changes the overall outcome. I generally find that about 40% works best. Running our GenerateCavern() function should yield something like this:

![Cellular Automata, Step 1](/static/week-2-step-1.png)

Again, tweak the percentage and see the changes. It will have a larger impact later, but for now, with a 50% chance, it covers roughly half the map with walls, and half with floor. This random looking mess is going to be the basis for our cavern.

Next up, we're going to go through each Tile we created in the previous step, and see if it should be a wall or a floor, based a couple of very simple rules. First, if 4 or more of the Tiles immediate neighbors are walls, or two or less neighbors up to 2 tiles away are walls, then that Tile becomes (or stays) a wall. If neither of those are true, the tile will become (or stay) a floor. Basically, going back to the cell analogy, if a cell is dead (floor), check to see if it has enough alive (wall) neighbors to be become alive again, and if a cell is alive (wall), check to ensure it has enough alive neighbors to remain alive.

{{< highlight go >}}
// Step 2: Decide what should remain as walls. If four or more of a tiles immediate (within 1 space) neighbors are
// walls, then make that tile a wall. If 2 or less of the tiles next closest (2 spaces away) neighbors are walls,
// then make that tile a wall. Any other scenario, and the tile will become (or stay) a floor tile.
// Make several passes on this to help smooth out the walls of the cave.
for i := 0; i < 5; i++ {
	for x := 0; x < m.Width; x++ {
		for y := 0; y < m.Height - 1; y++ {
			wallOneAway := m.countWallsNStepsAway(1, x, y)

			wallTwoAway := m.countWallsNStepsAway(2, x, y)

			if wallOneAway >= 5 || wallTwoAway <= 2 {
				m.Tiles[x][y].Blocked = true
				m.Tiles[x][y].Blocks_sight = true
			} else {
				m.Tiles[x][y].Blocked = false
				m.Tiles[x][y].Blocks_sight = false
			}
		}
	}
}
{{< /highlight >}}

Lets explain whats going on here. First up, we're going to make several passes over this code (5 to be precise). This will ensure that each time through, we "smooth" out the newly forming cavern a little bit more. This is also what will allow our cave formations to take shape. Any openings that are present after the first pass will be made more prominent in subsequent passes, as the algorithm tries to maintain groups of like Tiles.

Next, we set up a loop over every Tile in the GameMap, and call a new function, countWallsNStepsAway(). That function, which hopefully should be self explanatory from the name, looks like this:

{{< highlight go >}}
func (m *Map) countWallsNStepsAway(n int, x int, y int) int {
	// Return the number of wall tiles that are within n spaces of the given tile
	wallCount := 0

	for r := -n; r <= n; r++ {
		for c := -n; c <= n; c++ {
			if x + r >= m.Width || x + r <= 0 || y + c >= m.Height || y + c <= 0 {
				// Check if the current coordinates would be off the map. Off map coordinates count as a wall.
				wallCount ++
			} else if m.Tiles[x + r][y + c].Blocked && m.Tiles[x + r][y + c].Blocks_sight {
				wallCount ++
			}
		}
	}

	return wallCount
}
{{< /highlight >}}

This function simply returns how many neighbors, n steps away from the provided Tile, are walls. We call that function twice, one for immediate (1 step away) neighbors, and once for neighbors two steps away. We then compare those returned values to the rules we laid out above. If it meets the criteria for neighbor walls, it becomes a wall, otherwise, we make it a floor. After one pass, our caverns look like this:

![Cellular Automata, Step 2, 1](/static/week-2-step2-1.png)

And, after five passes, it looks like this:

![Cellular Automata, Step 2, 5](/static/week-2-step2-2.png)

You can see how our algorithm has broken out multiple separate caverns, and has "smoothed" them out, getting rid of jagged edges and odd outcroppings that don't fit in their surroundings. Its actually starting to look like a proper set of caverns!

Next, we'll do a few more passes, but this time only looking for bits of wall that are by themselves, or extremely isolated among floor tiles (this will get rid of the walls that are standing out in the middle of the caverns by themselves):

{{< highlight go >}}
// Step 3: Make a few more passes, smoothing further, and removing any small or single tile, unattached walls.
for i := 0; i < 5; i++ {
	for x := 0; x < m.Width; x++ {
		for y := 0; y < m.Height - 1; y++ {
			wallOneAway := m.countWallsNStepsAway(1, x, y)

			if wallOneAway >= 5 {
				m.Tiles[x][y].Blocked = true
				m.Tiles[x][y].Blocks_sight = true
			} else {
				m.Tiles[x][y].Blocked = false
				m.Tiles[x][y].Blocks_sight = false
			}
		}
	}
}
{{< /highlight >}}

This pass is pretty simple, nothing new of note, aside from the slightly different wall rule. After five more passes of this, our cave now looks like this:

![Cellular Automata, Step 3](/static/week-2-step3.png)

We've now got some nice, smooth, artifact free caverns. The bottom edge of the screen looks a little ragged, but we'll clean that up in a minute. The next thing we're going to do is seal up the edges of the map, in case any of our caverns has an opening off the edge of the map. We don't want the player to be able to wander outside our designated map area:

{{< highlight go >}}
// Step 4: Seal up the edges of the map, so the player, and the following flood fill passes, cannot go beyond the
// intended game area
for x := 0; x < m.Width ; x++ {
	for y := 0; y < m.Height; y++ {
		if x == 0 || x == m.Width - 1 || y == 0 || y == m.Height - 1 {
			m.Tiles[x][y].Blocked = true
			m.Tiles[x][y].Blocks_sight = true
		}
	}
}
{{< /highlight >}}

This is straight-forward, we just go around the edges, and make every edge Tile a wall.

![Cellular Automata, Step 4](/static/week-2-step4.png)

At this point, we've got a bunch of small to large caverns, all nicely shaped, and very natural looking (I think so, at least :). However, the astute may have noticed that if the player were to spawn in one cavern, they would have no way of reaching any of the others (no digging in my roguelikes, not fond of the mechanic). So, how do we solve that? Well, the best way is to ensure there is only one main cavern. We could tunnel between each cavern, and run a few more passes of smoothing, but I actually like the idea of choosing the largest continuous cavern, and filling in the rest. That way, we ensure the player can reach every reachable part of the cavern they land in. The approach towards doing this also uses a neat algorithm called a flood fill, which we'll go over next.

{{< highlight go >}}
// Step 5: Flood fill. This will find each individual cavern in the cave system, and add them to a list. It will
// then find the largest one, and will make that as the main play area. The smaller caverns will be filled in.
// In the future, it might make sense to tunnel between caverns, and apply a few more smoothing passes, to make
// larger, more realistic caverns.

var cavern []*Tile
var totalCavernArea []*Tile
var caverns [][]*Tile
var tile *Tile
var node *Tile

for x := 0; x < m.Width - 1; x++ {
	for y := 0; y < m.Height - 1; y++ {
		tile = m.Tiles[x][y]

		// If the current tile is a wall, or has already been visited, ignore it and move on
		if !tile.isVisited() && !tile.isWall() {
			// This is a non-wall, unvisited tile
			cavern = append(cavern, m.Tiles[x][y])

			for len(cavern) > 0 {
				// While the current node tile has valid neighbors, keep looking for more valid neighbors off of
				// each one
				node = cavern[len(cavern)-1]
				cavern = cavern[:len(cavern)-1]

				if !node.isVisited() && !node.isWall() {
					// Mark the node as visited, and add it to the cavern area for this cavern
					node.Visited = true
					totalCavernArea = append(totalCavernArea, node)

					// Add the tile to the west, if valid
					if node.X - 1 > 0 && !m.Tiles[node.X -1][node.Y].isWall() {
						cavern = append(cavern, m.Tiles[node.X -1][node.Y])
					}

					// Add the tile to east, if valid
					if node.X + 1 < m.Width && !m.Tiles[node.X + 1][node.Y].isWall() {
						cavern = append(cavern, m.Tiles[node.X + 1][node.Y])
					}

					// Add the tile to north, if valid
					if node.Y - 1 > 0 && !m.Tiles[node.X][node.Y - 1].isWall() {
						cavern = append(cavern, m.Tiles[node.X][node.Y - 1])
					}

					// Add the tile to south, if valid
					if node.Y + 1 < m.Height && !m.Tiles[node.X][node.Y + 1].isWall() {
						cavern = append(cavern, m.Tiles[node.X][node.Y + 1])
					}
				}
			}

			// All non-wall tiles have been found for the current cavern, add it to the list, and start looking for
			// the next one
			caverns = append(caverns, totalCavernArea)
			totalCavernArea = nil
		} else {
			tile.Visited = true
		}
	}
}
{{< /highlight >}}

Whoa, there's a lot going on there. Lets step through. First off, a flood fill algorithm is literally what it sounds like. It starts at a point, and will attempt to get each element nearby that shares a characteristic, while ignoring those that do not (flooding over an area until that area is filled). In our case, you can think of it like checking for an empty area (floor spaces), and expanding out, until it has found the maximum bounds of that area. At the end, we will have a list of all tiles in that area, and then the algorithm will move on, looking for the next open space that has not been checked yet.

We start out by defining a few variables, mainly slices to keep track of various things, like the current cavern, the size of the current cavern, and a list of all individual caverns we have found so far. Then, we begin by looping over every Tile element in our GameMap, and grabbing the Tile at each location. You'll notice I'm using pointers for everything, including my slices. This makes it much easier, as we are dealing with the actual object, rather than a copy of it.

Once we have a Tile, we use our two new convenience methods from earlier to see if, a) have we already visited this Tile, and b) is this Tile a wall. If either of those are true, we mark this Tile visited, and move on. We keep track of Tiles we've visited, so as not to double (or more) process them in the future. If a tile has not been visited, and is not a wall, we have found a Tile that is part of a cavern, so we add it to the cavern slice.

While there are Tiles present in the cavern slice, we will attempt to get every valid neighbor of each one. This is the flood in flood fill. Inside our cavern loop, we pop the latest Tile off the slice, make sure we don't have a visited or wall Tile once more, and then mark the Tile as visited (since we will not need to process it after this. We also push it onto the totalCavernArea slice, to keep track of Tiles in this current cavern. Then, we check each of its neighbors, north, south, east and west. If any of those are not visited and not walls, we add them to the current cavern. Then, we loop again, popping off a new Tile, and processing it in the same way.

In this way, we will eventually get every floor tile in the current cavern, and at the end, have a complete representation of that cavern in terms of the floor tiles that compose it. When we have exhausted (filled, to use the algorithms parlance) a cavern, we take that set of Tiles representing the cavern, and append it to the caverns slice, which is a slice of slices. Each sub slice in that slice represents a single complete cavern.

Once we are done here, we should have 12 entries in our caverns slice, one for each cavern present on our map. Keep in mind that you will have a different number.

Nice! We now have a comprehensive list of every cavern in our greater cavern system. At this point, you could do all sorts of neat things, but we're going to stick with our original plan, and fill in all but the largest cavern. This is pretty straight forward, but uses a new (to me at least) Go concept, sorting a slice.

{{< highlight go >}}
// Sort the caverns slice by size. This will make the largest cavern last, which will then be removed from the list.
// Then, fill in any remaining caverns (aside from the main one). This will ensure that there are no areas on the
// map that the player cannot reach.
sort.Sort(BySize(caverns))
mainCave := caverns[len(caverns) - 1]
caverns = caverns[:len(caverns) - 1]

for i := 0; i < len(caverns); i++ {
	for j := 0; j < len(caverns[i]); j++ {
		caverns[i][j].Blocked = true
		caverns[i][j].Blocks_sight = true
	}
}
{{< /highlight >}}

In order to sort our caverns slice, we'll need to define some custom logic. We want to sort it by the size of the sub slice, smallest to largest, so we can just pop off the last element of the slice, and be sure we have the largest cavern. In order to properly do this, we'll need to define a new type to sort by, called BySize:

{{< highlight go >}}
type BySize [][]*Tile

func (s BySize) Len() int {
	return len(s)
}

func (s BySize) Swap(i, j int) {
	s[i], s[j]= s[j], s[i]
}

func (s BySize) Less(i, j int) bool {
	return len(s[i]) < len(s[j])
}
{{< /highlight >}}

Our new type is a 2D slice of Tile pointers, just like our caverns slice. We then define three functions attached to it, to allow for our comparisons. This code is all pretty self explanatory, so I'll leave it at that. Suffice it to say, that when we call sort.Sort() and pass in our new sorting type and our caverns array, it will return a neatly sorted array of caverns, from smallest to largest. Then, we simply just pull the largest cavern off the caverns array, and then systematically fill in the rest.

This leaves us with a single, connected cavern:

![Cellular Automata, final](/static/week-2-step-final.png)

Now, all we need to do is place out player in a valid location within the main cave:

{{< highlight go >}}
// Finally, choose a starting position for the player within the newly created cave
pos := rand.Int() % len(mainCave)
return mainCave[pos].X, mainCave[pos].Y
{{< /highlight >}}

And that's that. Lets call our new method in our main file, in our init function, so our cave will be the default map:

{{< highlight go >}}
// Create a GameMap, and initialize it (and set the player position within it, for now)
gameMap = &gamemap.Map{Width: MapWidth, Height: MapHeight}
gameMap.InitializeMap()

playerX, playerY := gameMap.GenerateCavern()
player.X = playerX
player.Y = playerY
{{< /highlight >}}

Now, when you fire up the game, you should have a sweet, random cave to explore every time! I would like to note that my cave in my example is a little lackluster, as it didn't generate the largest cave possible, but that's the beauty of procedural generation. I find that the caves are quite complex if done on a much bigger map. My example is 100x35, but here's some examples on a 200x200 map:

![Cellular Automata, Complete1](/static/week-2-final1.png)

The variety it can create is pretty neat

![Cellular Automata, Complete2](/static/week-2-final2.png)

I'm generally pretty happy with the results

![Cellular Automata, Complete3](/static/week-2-final3.png)

So, that's it, we now have an exciting map for our player to explore and get lost in. As I mentioned earlier, play around with various knobs on the algorithms (number of passes, percentages, ratios, etc), and see what works best. By tweaking a few small things, you can get drastically different results. As usual, if you want to see the full code for this post, you can find it in my [repo](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v0.0.5)

Be sure and join me next time when we'll be tackling field of vision, so not only can our player get lost, they can get lost in the cold, lonely, dark. Until next time, happy developing!

[This way to Week 5]({{< relref "post/roguelike-dev-week-3-part-1.md" >}})
