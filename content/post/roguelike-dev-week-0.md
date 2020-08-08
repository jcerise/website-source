+++
categories = ["technology"]
date = "2017-06-21T23:54:35-06:00"
description = "Part 1: Setup"
draft = false
tags = ["go", "roguelike", "programming", "RLDBAR"]
title = "RoguelikeDev Builds a Roguelike, Part 1"
toc = false

+++
So it begins! This is the first in a series of posts following my attempts at making a Roguelike, in Go, using BearLibTerminal. I'll be following along with the `RoguelikeDev builds a Roguelike` posts as I progress.

So, without further ado, lets begin part 0: Setup!`

<!--more-->
Part 0 is dedicated to setting up your dev environment, and making sure everything is running correctly. There won't be a lot of code this week, although the end result will be a working terminal with the traditional 'Hello,  World!' text. As a bonus, we'll display that text in a color other than boring old white.

Since I chose to use Go to write my roguelike in, rather than Python, the very first thing we should do is install Go. Depending on what OS you're using, the process will be a little different, but generally not difficult. I'm using Arch Linux, so for me, installing Go simply consisted of:

{{< highlight bash >}}
sudo pacman -S go
{{< /highlight >}}

You can check [the docs](https://golang.org/doc/install#install) for specifics on how to get Go installed for your operating system.

Once Go is installed, and you have verified that it works, we'll need to make sure our GOPATH is configured correctly. GOPATH is where Go will look for packages included outside of the main project structure (bear with my explantions, I'm still learning Go). Typically, this is set to $HOME/go, and thats where I left it on my system. You can read more about GOPATH [here](https://golang.org/cmd/go/#hdr-GOPATH_environment_variable).

The next step is to install BearLibTerminal. BearLibTerminal has a Go package we can use out of the box, and thats what we will be including in our project. Download BearLibTerminal [here](http://foo.wyrd.name/en:bearlibterminal#download). Once downloaded and unpacked, we need a few files. We'll need `include/BearLibTerminal.Go`, and `Linux64/LibBearLibTerminal.so`, and `include/C/BearLibTerminal.h`. The C header file is important, as thats what powers the whole library, so don't forget that. We're now going take those files, and place them in `GOPATH/src/bearlibterminal`. This will allow us to use BearLibTerminal in our project, by simply including it.

Great! Now, we're more or less setup to start developing! A quick note, I'm using [Gogland](https://www.jetbrains.com/go/), Jetbrains Go IDE (still under active development). Its currently free, but won't remain that way for much longer, I'd wager. Any IDE or text editor will do, though.

Lets set up our project. I've called my project BearRogue (super creative, I know), and my file structure is pretty basic right now.

{{< highlight bash >}}
bearrogue
--src/
    bearrogue.go
--font/
    UbuntuMono.ttf
--README.md
{{< /highlight >}}

Right, lets get right into it. We need to declare our package, and include BearLibTerminal, first and foremost:

{{< highlight go >}}
package main

import (
    blt "bearlibterminal"
    "strconv"
)
{{< /highlight >}}

If your GOPATH is configured correctly, and you've put the BearLibTerminal files there as instructed, Go should be able to find BearLibTerminal without any trouble. The `strconv` package will be used to do some type conversions later.

With our packages included, lets set up some basic constants to make our lives easier:

{{< highlight go >}}
const (
    WindowSizeX = 100
    WindowSizeY = 35
    Title = "BearRogue"
    Font = "fonts/UbuntuMono.ttf"
    FontSize = 24
)
{{< /highlight >}}

These constants should be pretty straight forward. The two WindowSize constants will set the size of our terminal window, Title sets the text to display in the title bar, Font tells BearLibTerminal what font we want to use, and FontSize what size to render that font. A note, I'm using a HDPI display, so adjust these to fit your screen accordingly.

Now that we've got some constants defined, lets use them to initialize BearLibTerm. I'm going to use an `init` method, as that will be called before our `main` method. I'll explain whats going after the code snippet:

{{< highlight go >}}
func init() {
    blt.Open()

    // BearLibTerminal uses configuration strings to set itself up, so we need to build these strings here
    // First, lets set up the string for window properties (size and title)
    size := "size=" + strconv.Itoa(WindowSizeX) + "x" + strconv.Itoa(WindowSizeY)
    title := "title='" + Title + "'"
    window := "window: " + size + "," + title

    // Next, set up the font config string
    fontSize := "size=" + strconv.Itoa(FontSize)
    font := "font: " + Font + ", " + fontSize

    // Now, put it all together
    blt.Set(window + "; " + font)
    blt.Clear()
}
{{< /highlight >}}

The first thing we do is Open a new BearLibTerminal:

{{< highlight go >}}
blt.Open()
{{< /highlight >}}

This initializes a new window for us to use. The next thing we do is set up the window configuration. BearLibTerminal uses config strings to set itself up, so we build two of those, one for window properties, and one for font properties, and then apply to the terminial window using the Set method. Our config strings, respectivley, look like this after we've created them:

{{< highlight bash >}}
window: size=100x35,title=BearRogue
font: fonts/UbuntuMono.ttf, size=24
{{< /highlight >}}

You can read more about BearLibTerminals configuration strings [here](http://foo.wyrd.name/en:bearlibterminal:reference:configuration)

Finally, we use the Set method to apply our config strings, and then we clear the terminal, making sure there is nothing displayed.

Now that we have initialized and configured our terminal window, its time to actually display it, along with some text! We're going to do this in our `main` method, which is where our game loop will live for the time being. Our game loop for now is going to be very simple, its just going to check if the user has closed the window. If not, it will keep itself open.

{{< highlight go >}}
func main() {
    blt.Print(1, 1, "Hello, world!")
    blt.Refresh()

    for blt.Read() != blt.TK_CLOSE {
        // Do nothing here for now
    }

    // If the user hit the close button, close the window and exit
    blt.Close()
}
{{< /highlight >}}

I've introduced a few new BearLibTerminal functions here. First is Print(), which will print the text we specify to the terminal, at the (x, y) coordinates provided. In this case, it prints "Hello, World!" to the coordinates (1, 1), which is the top left corner of the window. Next is Refresh() which effectively redraws the content of the terminal. No changes will be displayed until we call Refresh() again. Read() returns the next event from the event queue (which keeps track of user input). Hitting the close button on the window adds a TK_CLOSE event to the event queue, which, when we call Read(), will be returned. Finally, Close() will close the current window.

So, we are printing some text to the console with Print(), drawing that text to the console with Refresh(), endlessly looping until a TK_CLOSE event is pushed onto the event queue, and then using Close() to end the program. Easy!

At this point, if you run your program with:

{{< highlight bash >}}
go build bearrogue.go
{{< /highlight >}}

And run the resulting executable, you should see a window, with the title BearRogue, and the text "Hello, World!" displayed in the top left corner. Nice, we're well on our way to having a functioning roguelike!

For a bonus, lets change the color of the text. Before we print the text "Hello, World!", add this line, in your `main` function:

{{< highlight go >}}
blt.Color(blt.ColorFromName("darker green"))
{{< /highlight >}}

The Color() function will change the color of all output to the terminal, until it is changed again. So, this will make our "Hello, World!" print out in a dark shade of green. BearLibTerminal is quite flexible in color names, but I'll let you experiment with that, and read the specifics [here](http://foo.wyrd.name/en:bearlibterminal:reference#color).

So, now we've got a colorful message, but wouldn't it be nice if it was perfectly centered on the screen? Lets add a function to print out centered text:

{{< highlight go >}}
func printCenteredText(text string) {
    x := WindowSizeX / 2 - (len(text) / 2)
    y := WindowSizeY / 2
    blt.Print(x, y, text)
}
{{< /highlight >}}

Basically, we get the center point of the window (WindowSizeX / 2, WindowSizeY / 2), and then half the width of the text to print (len(text)). If we didn't get half the width of the text, our text would print off center to the right, as it would start printing from the center point, not the center of the text. To use our new function, simply replace the Print() call in `main` with:

{{< highlight go >}}
printCenteredText("Hello, World")
{{< /highlight >}}

Now, running our program again should yield a cliched, dark green message, perfectly centered in our terminal window!

To recap, this week we installed Go, installed BearLibTerminal, and set up a small Go program that creates a BearLibTerminal window, and prints a colored, centered, "Hello, World" message. Not too shabby. If you want to view the entire code for this week, you can find it under the Week 0 release on my [repo](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v0.0.1). I'll see you next week where we'll be setting up the basics of our game! Happy developing!

[This way to Part 2]({{< relref "post/roguelike-dev-week-1-part-1.md" >}})
