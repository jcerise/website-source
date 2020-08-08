+++
categories = ["technology"]
date = "2017-06-27T22:33:11-06:00"
description = "Part 2: The Player and Movement"
draft = false
tags = ["RLDBAR", "go", "programming", "roguelike"]
title = "RoguelikeDev Builds a Roguelike, Part 2"
toc = false

+++
Welcome back to my series following along with RoguelikeDev Builds a Roguelike! This is part 2, and this week, we've got things split into two parts. The first part, which this post will be concerned with, involves displaying the player and the screen, and implementing basic four direction movement. Part two will consist of creating a generic 'Object' type that will represent things in the game, as well as creating an initial dungeon map.

Lets dive right in creating our player, and getting it moving!
<!--more-->

We left off last week with a terminal window, displaying some basic "Hello, World!" text. Certainly not the most exciting thing in the world, and in fact quite a ways from a playable game, but progress none-the-less. This time around, lets actually make it so that the player can interact with the game itself, in the form of moving a representation of the player around the screen.

The first thing we're going to need to do is come up with a way to reliably print a character representing the player to the screen. In this case, we will use the classic roguelike standby, the '@'. We've already gone over how to print things to the screen last week, using BearLibTerminals Print() method. We could certainly do something like:

{{< highlight go >}}
blt.Print(1, 1, '@')
{{< /highlight >}}

This would print out player character to the screen at coordinates (1, 1), or the top left corner of the screen. Its easy enough to replace the code that prints "Hello, World!" with this code, and call it good, but what if we want to move the player around (which we do)? Well, just printing it over and over into the same place certainly won't help us there, so we'll need to store some information about the players location within our game world. Lets add a couple of new variables to our program:

{{< highlight go >}}
var (
    playerX = 0
    playerY = 0
)
{{< /highlight >}}

We'll use these two variables to keep track of where the player is in the game world. A quick note, these are globally defined, which in my opinion is generally a no-no, so we'll be cleaning them up later, but for now, this is fine.

So, we've got our players location stored, and we can now use that to print the player character wherever the coordinates say. What we want now, is to use those new variables to update the players position whenever arrow keys are pressed. For example, if the player presses the UP arrow key, we would expect that the players '@' will move towards the top of the screen. This may sound a little tough, but in reality its quite simple. Since BearLibTerminal gives us a nice grid to work with (and we can assume that each square in the grid is 1x1), all we need to do is move the player by 1 unit in the desired direction, when an arrow key is pressed. 

To elaborate a bit more, when the player presses UP, we want to adjust playerY - 1, and when they press DOWN, we want to adjust playerY + 1. Likewise, when they press LEFT, we would adjust playerX - 1, and when RIGHT is pressed, playerX + 1. Hopefully this is obvious, but if not, we are simply moving the player around on an inverted two dimensional (x, y) plane ( (0, 0) is at the top left corner, rather than the bottom left, as you might expect). If we move towards the top of the screen, the y value of the point (our player) is decreasing, and approaching 0. If we move towards the bottom of the screen, our y value is increasing, and approaching WindowSizeY. Same for the x values: left is decreasing, and approaching 0, right is increasing and approaching WindowSizeX.

To tie this all together, we simply need to handle the key presses from the arrow keys, which, as we've already seen, can be done in BearLibTerminal using the Read() method. Lets create a new method to handle this, that takes in the value returned by the Read() method:

{{< highlight go >}}
func handleInput(key int) {
    switch key {
    case blt.TK_RIGHT:
        playerX ++
    case blt.TK_LEFT:
        playerX -- 
    case blt.TK_UP:
        playerY --
    case blt.TK_DOWN:
        playerY ++
    }
}
{{< /highlight >}}

The Read() method simply returns an integer, which we can then map back to one of BearLibTerminals key event types. The switch statement simply checks which key was pressed, and following our logic from earlier, adjusts the playerX and playerY values accordingly. Pretty simple!

Now, we've got a way to update the position of our player, lets add a new function that will draw the player the screen, using the playerX and playerY variables. This is going to be a convenience method, so we can clean up our code a bit. BearLibTerminal has a concept of layers when it draws, which is something that I will go into in more detail in a later post, but suffice it to say, much like setting the color we are printing to the terminal, we can also specify which layer we are working with. There is also a method that will clear everything from a specified area (which in our case, currently, is the entire screen). We will use this to clear the entire screen, and then re-draw our player at the specified position. Don't worry if you don't understand the concepts of layers and drawing, as we'll get more in depth into that at a later time. Lets add a function called drawPlayer:

{{< highlight go >}}
func drawPlayer(x int, y int, symbol string) {
    blt.Layer(0)
    blt.ClearArea(0, 0, WindowSizeX, WindowSizeY)
    blt.Print(playerX, playerY, symbol)
}
{{< /highlight >}}

We select the layer (0, in this case, is our only layer), we clear the screen from position (0, 0) to (WindowSizeX, WindowSizeY), which is our entire gameplay area for now, and then we Print() the players character, setting it at the current values of playerX and playerY.

Okay, we've now got a way to get player input, and adjust the position of the players character, and a way to easily clear the screen, and re-draw the player at the updated position. The final step here is to put it all together in our main game loop:

{{< highlight go >}}
func main() {
    blt.Color(blt.ColorFromName("white")
    drawPlayer(playerX, playerY, "@")

    for {
        blt.Refresh()
        
        key := blt.Read()

        if key != blt.TK_CLOSE {
            handleInput(key)
            drawPlayer(playerX, playerY, "@")
        } else {
            break
        }
    }

    blt.Close()
}
{{< /highlight >}}

Our main game loop has changed a bit, so lets walk through it. First up, we're setting the color to print as white, same as we did previously. Then, we're drawing the player to the screen, using the playerX and playerY variables (set to (0, 0)). Then we get into our main loop. Each iteration of the loop, we Refresh() the screen, and then call Read() to see if the player pressed a key. If they did, we check to see if it was TK_CLOSE. If it was, we break out of the loop, and exit the game. If it wasn't, we pass it to handleInput(). handleInput only knows how to handle four different keys (the arrow keys), so it will safely ignore any key press that is not an arrow key. If an arrow key was pressed, we adjust the playerX and playerY variables accordingly. Finally, we print the player to the screen (clearing the contents first in the drawPlayer() method) at the new coordinates provided by playerX and playerY. Whew, quite a few changes there!

At this point, you should be able to fire up your game, see the players '@' sitting in the top left corner, and press an arrow key to move it around. Awesome!

You may notice that as you move your character around, it is possible for it to leave the screen. This is obviously not something we want. Lets go about fixing that. Essentially what we need to do, is if the player would move out of the range of the screen, set it back to the last valid position before it left. To elaborate, if the player is attempting to move to, say, (-1, 0), or just to the left of the edge of the screen, we should set his position back to (0, 0). Likewise, if the player is trying to move to (WindowSizeX + 1, 0), we should probably place the character back at (WindowSizeX, 0). Basically, set the players position to the last valid position they were at before they tried to move. Lets add this code the bottom of our handleInput() function, just below the end of the switch statement:

{{< highlight go >}}
if playerX > WindowSizeX - 1 {
    playerX = WindowSizeX -1
} else if playerX < 0 {
    playerX = 0
}

if playerY > WindowSizeY - 1 {
    playerY = WindowSizeY - 1
} else if playerY < 0 {
    playerY = 0
}
{{< /highlight >}}

This will stop the player from ever moving beyond the bounds of the screen. Go ahead and try it out, you'll find the player bumps into an invisble wall right at each edge of the screen! 

To recap so far: We've successfully drawn the player to the screen, and can move the character around. We made a function to handle player input, as well as one to draw the player to the screen, at a provided set of coordinates. This is starting to look more like a game already! Next time we'll set up a map for the player to explore, as well as handle some architectural concerns that will make our lives easier once our game starts to grow.

As usual, you can find the full code for this post under the Part 2 release on my [repo](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v0.0.2). See you next time, and happy developing!

[This way to Part 3]({{< relref "post/roguelike-dev-week-1-part-2.md" >}})

=====

Sidenote for this post: [Sidenote 2: VIM keys and Diagonal Movement]({{< relref "post/sidenote-2-vim-keys.md" >}})
