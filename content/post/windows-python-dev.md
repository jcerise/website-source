+++
title = "An enjoyable Python Development Environment on Windows 10"
date = 2020-08-03T23:40:45-06:00
description = "Windows 10 Pro, WSL2, Poetry, Pycharm"
draft = false
toc = false
categories = ["technology"]
tags = ["python", "WSL", "development"]
[[copyright]]
  owner = "Jeremy Cerise"
  date = "2020"
  license = "cc-by-nc-sa-4.0"

+++

One of the great things about Python is the fact that it can be developed, and runs on, many different platforms. There is no one platform that is better for writing Python programs, though certainly, there are some that make it a bit easier. While Linux is and will always be, my preferred environment for developing Python, recently, I've been re-visiting Windows as my primary development platform. With the WSL, and some new fancy terminal emulators, I'm finding that I don't miss my Linux box nearly as much as I used to. In this article, I'll go over my preferred setup on Windows, using the WSL2, Windows 10 Pro, and Pycharm.

<!--more-->

Certainly, you could just install Python for Windows, install Pycharm, and use Windows or Pycharms terminals to do most of what you need. And this is a fine approach, but it makes for a much less interesting article, and there are, at least for some of my use cases, limitations to doing things this way. What we're going to go over in this article is WSL2 powered Python setup, where we won't even be installing Python on Windows at all. Lets get into it.

Note, this is intended to be an overview, not an in-depth tutorial. I may expand on some of these concepts later though. 

First things first, you need to enable the WSL on your Windows install, and then upgrade to WSL2. There are many, many tutorials on accomplishing this, so I'll just reference [MIcrosofts official Docs](https://docs.microsoft.com/en-us/windows/wsl/install-win10) on the subject and leave it at that. Its pretty straight forward.

Next, you'll need to pick a Linux distro to use the WSL. I chose Ubuntu 20.04, as I'm decently familiar with Debian based distros. This tutorial assumes Debian (mainly the presence of apt for package management), but if you're more comfortable with a different distro, the general steps will remain the same.

My version of Ubuntu comes with Python 3.8.2 installed by default, and this is what we'll be using. You can use Pyenv or similar if you need to juggle python versions (and I may write another article about that...), but for our needs, 3.8 will serve just fine.

The first thing we're going to do is install [Poetry](https://python-poetry.org/). Again, I will leave the particulars to their own tutorial. Poetry is a really nice package manager, which handles creation of virtual environments for us, and is generally pleasant to use. We're installing this on our WSL environment, not in Windows.

One note, a lot of Poetry tutorials and such recommend setting the `virtualenvs.in-project` setting to true. This creates the new poetry virtualenv inside the poetry project directory. This can be a convenience, but for our needs, we're going to leave it set to False. I'll explain why in a bit.

Now, we're ready to create a project. Using Poetry, this is trivial. We need to make sure that we're creating this on the Windows filesystem, and not the WSL filesystem (this will make it easier to access our code using Pycharm, though it will be slower to access via WSL tools like Vim). So, somewhere like `/mnt/c/Users/<your_username>/python_projects`, lets create our new project. This location indicates that we are on the Windows C drive, under the Users directory.

```bash
poetry new wsl-test
```

Followed by:

```bash
poetry install
```

This will create our new project and virtual environment, including a tests directory, project directory, and a `.toml` config file. You can verify this by going to the folder location using Windows File Explorer.

Now for the fun part. Up until now, we haven't really done much interesting, aside from play around inside the WSL a bit. Next we're going to wire up Pycharm to use our Poetry virtualenv as the environment for our project, under Windows. This will allow us to run all our code via the WSL, without actually having Python installed on Windows.

Pycharm has a relatively recent feature that allows for using WSL Python interpreters in your Windows environment, meaning that we can utilize the Poetry environment we just created as our project interpreter in Pycharm. To do this, we need to first open our project in Pycharm. Once you've done that, go to `settings->project->python interpreter`. From there, hit the gear icon, and add a new interpreter. You should see an option on the left for WSL, with a little Tux icon. Clicking that will auto-populate with your distro, or if you have multiple, allow you to choose which distro you want to use. 

The `Python Interpreter Path` option below that is what we're really interested in. Point this to the location of your poetry virtualenv (mine is located at the default Poetry location of`/home/jcerise/.cache/pypoetry/virtualenvs`. If you're not sure where yours is located, in your WSL shell, run `poetry env info`, and take note of the path variable). Hit `OK`, and let Pycharm index the interpreter. 

Remember earlier, when I said that we were disabling putting our virtual envs inside our project directory? Well, this is the reason. If you try and select a Python binary from your virtual envs `bin` directory in Pycharm, that resides within your project, well...it just doesn't work. For some reason. So, to avoid this, just let Poetry store your venvs outside the project, and all is well.

Pycharm will now pick up the entire Poetry environment, allowing you to run your project from within Pycharm, as if it was running on Windows. Pretty sweet. 

You can even run GUI applications this way, if you install an Xserver on Windows, but thats for another blog post.

And there we go, a full fledged Python development environment on Windows 10, without actually installing Python on Windows 10.

#### A note about Terminal emulators

When using the above setup, you'll probably still find yourself using a terminal to interact with your WSL distro frequently. Microsoft released a pretty nice terminal called Windows Terminal (which you can find in the Windows store) that makes doing this pretty painless. But, while fine, its missing a few things I really like. Namely, quake style dropdown functionality, tied to a global hotkey. 

I can't live without it. The convenience of having my terminal always at my fingertips without having to switch windows is great. I've been using it for years on Linux (Guake, Yakuake, etc), and I really miss it when I don't have it. Thankfully, there are a couple options for this on Windows, and they both support WSL terminals.

[ConEmu](https://conemu.github.io/) is the first I tried, and it works fine. You can configure it to launch a WSL terminal by default on opening, and it has nice configs built in for global hotkey open, and dropdown style appearance. I found it be a little slow when using WSL with it though. Scrolling through a large file with Vim and watching it try and catch up rendering long after I released the j key is an example. YMMV.

The emulator I settled on is [Hyper](https://hyper.is/). Hyper is an electron based terminal emulator that runs on multiple platforms, and ticks all the boxes I require through a nice plugin ecosystem. Dropdown, hotkey summoned, and even transparency. Plus, none of the performance issues that ConEmu seems to have (and it appears to use fewer system resources, which is a plus). I recommend giving it a try, its more flexible in its configuration than Windows Terminal.

