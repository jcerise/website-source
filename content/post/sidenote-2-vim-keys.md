+++
title = "RLDBAR Sidenote #2: Vim Keys and Diagonal Movement"
date = 2017-07-27T22:10:20-06:00
description = "Vim key movement, and moving diagonally"
draft = false
toc = false
categories = ["technology"]
tags = ["RLDBAR", "go", "programming", "roguelike", "roguelike movement"]
+++

In this second sidenote to the main RLDBAR tutorial, we're going to make a very small, but very useful change: diagnonal movement, and allowing users to move via VIM keys. The first, diagonal movement, opens up many more tactical options over just being able to move in the cardinal directions (north, south, east, west), and will give the player more flexibility. Adding in support for VIM key movement (h, j, k, l, y, u, b, n) will give our game a bit more reach to users without full keyboards (not that we've implemented number pad support yet... :P ). Lets get it done!

<!-- more -->

As I already mentioned, this is a very small, even trivial, change. Lets just go ahead and look at the code (this is the handleInput() function from our main file):

{{< highlight go >}}
func handleInput(key int, entity *ecs.GameEntity) {
	// Handle basic character movement in the four main directions, plus diagonals (and vim keys)

	var (
		dx, dy int
	)

	switch key {
	case blt.TK_RIGHT, blt.TK_L:
		dx, dy = 1, 0
	case blt.TK_LEFT, blt.TK_H:
		dx, dy = -1, 0
	case blt.TK_UP, blt.TK_K:
		dx, dy = 0, -1
	case blt.TK_DOWN, blt.TK_J:
		dx, dy = 0, 1
	case blt.TK_Y:
		dx, dy = -1, -1
	case blt.TK_U:
		dx, dy = 1, -1
	case blt.TK_B:
		dx, dy = -1, 1
	case blt.TK_N:
		dx, dy = 1, 1
	}
    
    ...
}
{{< /highlight >}}

And thats it. All we're doing is allowing the player to use the standard VIM movement keys (l for right, h for left, k for up, and j for down) for cardinal navigation (in addition to the arrow keys). This makes our movement more flexible, and more VIM friendly (sorry emacs fans).

The next addition is the four cases for the y, u, b, and n keys. These will allow for diagonal movement. Y for example, will move our player -1 on the x axis, and -1 on the y axis, which translates to diagonally up and to the left. N, on the other hand, translates to  diagonally down and to the right.

Pretty straight forward, but very powerful, especially since any monsters we create in the future will be able to move diagonally. We wouldn't want them to have an unfair advantage, now would we?

That's about all for this sidenote, I don't have a release for this one, as it was such a small change, but you can always check out the latest code for the in progress roguelike on my [repo](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial). Hopefully, I'll be back on the main tutorial track here shortly, and until then, happy developing!
