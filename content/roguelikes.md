+++
menu = "main"
weight = 3
date = "2017-06-19T17:05:00-06:00"
description = "All about Roguelike Development"
title = "Roguelike Development"

+++

***Update 8/3/2020***

[Gogue](https://github.com/gogue-framework) has come a long ways since my last update. It's a relatively full featured Go Roguelike framework. I also moved my tutorial writing over to the wiki on the Gogue project. There is a (work in progress) tutorial found [here](https://github.com/gogue-framework/gogue/wiki), as well as instructions on getting setup. You can also find the source for the tutorial Roguelike created in the above mentioned tutorial [here](https://github.com/gogue-framework/gogue-rogue-example). 

I'm leaving this tutorial series up for posterity, but if you're looking for more up to date information, I highly suggest you check out [Gogue](https://github.com/gogue-framework)!



***Update 5/6/2018***

****
Obviously, I never finished this. Maybe at some point I will, but for now, its incomplete. I did however, take what I learned and created a simple Roguelike library in Go, which I call [Gogue](https://github.com/gogue-framework). It takes most of the concepts from my tutorials here, and puts them in an easy to use library.

---

I am currently participating in [r/roguelikedev's](http://reddit.com/r/roguelikedev) 'RoguelikeDev Does the Complete Python Tutorial'. The goal of this is to, as a group,
run through the entire ['Complete Roguelike Tutorial in Python'](http://www.roguebasin.com/index.php?title=Complete_Roguelike_Tutorial,_using_python%2Blibtcod), from RogueBasin, written by Jotaf. This tutorial is a classic, in my opinion (and a few years ago, 
I took the opportunity to run through it myself, you can find my code on [github](https://github.com/jcerise/DungeonCrawler)), and is very well suited to new coders, or
those who are looking to get a good overview of how a Roguelike is constructed.

For this project, I decided, since I have already run through the tutorial in Python, that I would learn a new language in the process. I chose [Go](https://golang.org/),
as I've been meaning to dive into it for a while. From there, I chose the excellent [BearLibTerminal](http://foo.wyrd.name/en:bearlibterminal) for my terminal display 
needs, as there is a Go package ready to use. I'll be posting my progress here, as well as a weekly blog post walking through the steps I'm taking to follow along (and
any interesting Go tidbits I pick up along the way).

[Part 1: Setup]({{< relref "post/roguelike-dev-week-0.md" >}}) - [code](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v0.0.1)

[Part 2: Movement]({{< relref "post/roguelike-dev-week-1-part-1.md" >}}) - [code](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v0.0.2)

[Part 3: Entities and the Map]({{< relref "post/roguelike-dev-week-1-part-1.md" >}}) - [code](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v0.0.3)

[Part 4: A more interesting Map]({{< relref "post/roguelike-dev-week-2.md" >}}) - [code](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v0.0.5)

[Part 5: Field of Vision]({{< relref "post/roguelike-dev-week-3-part-1.md" >}}) - [code](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v.0.0.6)

[Part 6: Preparing for Combat]({{< relref "post/roguelike-dev-week-3-part-2.md" >}}) - [code](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v0.0.7)

[Part 7: ECS Refactor]({{< relref "post/roguelike-dev-part-7.md" >}}) - [code](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/0.0.8)

[Part 8: Combat] - [code](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/0.0.9)

[Part 9: Basic UI] - [code](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v.0.0.11)

======

Sidenotes:

[Sidenote 1: Cameras]({{< relref "post/sidenote-1-cameras.md" >}}) - [code](https://github.com/jcerise/roguelikedev-does-the-complete-roguelike-tutorial/releases/tag/v0.0.4)

[Sidenote 2: Vim Keys and Diagonal Movement]({{< relref "post/sidenote-2-vim-keys.md" >}})
