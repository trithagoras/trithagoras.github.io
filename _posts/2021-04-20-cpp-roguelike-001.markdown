---
layout: post
title:  "Chasms 001 - Making an ASCII Roguelike in C++"
date:   2021-04-20 20:30:00 +1000
categories: jekyll update
---

Welcome to my tutorial, thoughts, and philosophy regarding easy and logical game design. This tutorial series will see the creation of my ASCII roguelike, Chasms, which is heavily inspired by [POWDER](http://www.zincland.com/powder/), and admittedly, is my only exposure to the roguelike genre.

Chasms will be written with C++, but do not be deterred. This series tries to avoid displaying and explaining each line of code, but rather delivers a higher level explanation of how a game executes its loop.

At the end of part 1, you can expect the game to look like this:

![Gameplay](/assets/001-gameplay.gif)

<br>

## The Motivation

As you can see in the above gameplay, Chasms is a command-line ASCII turn-based roguelike. By keeping it ASCII, we don't have to spend any time on graphics, and the player gets the benefit of playing straight from the terminal.

<br>

## Getting Started

Whether you are making the game in C++, C, Python, Java, or any other language, the first thing to consider is how we're delivering our "graphics". For this, of course, we will be utilising the [ncurses](https://en.wikipedia.org/wiki/Ncurses) library. This will allow us to draw directly to the terminal at any position, in any colour, as well as instantly read keystrokes.

For C/C++, simply add the following line to your CMakeLists.txt

{% highlight cmake %}
set(CMAKE_CXX_FLAGS -lncurses)
{% endhighlight %}

*Removing the `XX` if using C*

Python comes with the curses library by default, and has *okay* accompanying documentation [here](https://docs.python.org/3/howto/curses.html).

I've also tried to find ncurses bindings for C\# in the past but had no luck. I did, however, end up writing my own bindings by taking advantage of Dll Imports. e.g.

{% highlight c# %}
[DllImport("libncurses.5.4.dylib", EntryPoint = "mvaddstr")]
public static extern int AddString(int y, int x, string str);
{% endhighlight %}

*I might make a separate post about this one day :)*

Now, let's consider the main control flow for our game, which is the most important thing to get right.

<br>

## The Main Loop

All games are represented with a main loop. Our game will have the added benefit of omitting any taxing physics calculations present with many other genres of game.

Here is the order that the game will follow:
1. `update()` - all logic and input processing is done here
2. `draw()` - the screen is refreshed and redrawn to reflect the previous update
3. `get_input()` - the input is polled here, and stored to be processed in the next `update()` call.


## The Game Class

The main game itself will be presenting in a class. Although the class should only be instantiated once, it still acts as a neat collection of **state**.