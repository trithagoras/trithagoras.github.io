---
layout: post
title:  "Roguelike 001 - Making an ASCII Roguelike in C++"
date:   2021-04-20 20:30:00 +1000
categories: jekyll update
permalink: /roguelike-001/
---

## Table of Contents
- [The Motivation](#the-motivation)
- [Getting Started](#getting-started)
- [The Main Loop](#the-main-loop)
- [The Game State](#the-game-state)
- [The Game Class](#the-game-class)
- [Entity, Mob, and Player Class](#entity-mob-and-player-class)
- [The Floor Class](#the-floor-class)
- [The Game View](#the-game-view)
- [Player Input and Movement](#player-input-and-movement)

<br>

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

The main game itself will be presented in a class. Although the class should only be instantiated once, it still acts as a neat collection of **state**. It also allows us to reinstantiate it in the case we want to start a new game, for example.

All drawing operations will be presented in a collection of functions outside any class. Good user-friendly software will separate the view (drawing) from the controller (game); see [MVC](https://en.wikipedia.org/wiki/Model–view–controller).

Good C++ design is also to not use classes if you don't need to, and as such, all drawing functions will be grouped in a namespace, as there is no need for keeping state.

<br>

## The Game State
The state of a program is recursively defined as a set of the state of all of its variables.

We consider the **basic** data type (int, bool, char, float, etc.) and the **composite** data type (arrays, structs, classes, etc.). We recognise that any composite data type can be recursively constructed with basic data types.

With this knowledge, plus the knowledge that the state of any basic data type variable is simply its value, we can conclude that the state of any composite data type is recursively defined as the state of all of its fields.

As an example, consider the following class:

{% highlight c++ %}
class Entity {
    std::string name, description;
    Floor current_floor;
    int y, x;
};
{% endhighlight %}

And an entity s defined as such (noting <?> here represents an object we don't necessarily need to know the value of):

`s.name = "Gold Coin"; s.description = "shiny"; s.current_floor = <?>; s.y = 4; s.x = 10`

Then we can say the state of Entity s is the following tuple:

( "Gold Coin", "shiny", `State(s.current_floor)`, 4, 10)

This is important, as now we can construct classes by considering the state we wish to keep. In our Game class, we want at least 25 *floors*, each with a set of *entities* that exist on that floor, and a reference to our *player* object. We can now consider our Game class.

<br>

## The Game Class

So now we know we want to keep track of the player here as well as all the floors. We also want the class to consist of the logic functions, but these do not compose state, they simply act to modify existing state.

{% highlight c++ %}
class Game {
    int next_input;
    int current_floor;
    std::array<Floor, 25> floors;

    Player player;

    void main_loop();
    void update();
    void get_input();
    void end();

public:
    Game();
    void start();
};
{% endhighlight %}

*The Player and Floor class will be explained in a future section*

The idea of this class is simple. In `main()`, a `Game` object will be instantiated and then started using `start()`. This method will initialise any other variables, then call the `main_loop()` method. The body of this function is the main game loop mentioned earlier, and `update()` and `get_input()` are shown here.

Before we can get into any of the drawing, we will create the Player class and the Floor class. We will create them as basic as possible for now, and extend them in future tutorials.

<br>

## Entity, Mob, and Player Class
For a larger scale game, the Entity-Component-System design is more effective than the gameobject inheritence design, but for Chasms, we mostly know what we want as a final product, and as such, can work with the somewhat easier inheritence pattern.

To begin, we will need a base gameobject class. I've come to calling this class the **Entity** class. Entity was actually described in the [Game State](#the-game-state) section above.

Next, we will need an inherited class called the Mob (mobile object). This will act as the base for all monsters, creatures, humanoids, and indeed, the player character. For now, we will create an empty Mob class that simply inherits Entity.

Finally, the Player class can also simply inherit the Mob class with no extra members for now.

<br>

## The Floor Class
Chasms takes place within a 25-floor dungeon. As such, the dungeon can entirely be described as an ordered array of 25 instances of a Floor class.

Indeed, in the Game class above, we have an `std::array<Floor, 25>` which describes this.

For a floor class, we can get away with just having a width, height, and a completely randomly generated terrain matrix which can be extended in future tutorials. A terrain matrix would be a 2D array of ints which would describe if a position is empty or solid. Since we do not necessarily know the width and height of a floor (random), we will make the terrain matrix a 2D vector created as such:

{% highlight c++ %}
Floor Floor::generate_floor() {
    auto f = Floor();
    f.height = 100;
    f.width = 100;

    for (int row = 0; row < f.height; row++) {
        f.terrain.push_back(vector<int>());
        for (int col = 0; col < f.width; col++) {
            auto t = rand() % 2;
            f.terrain.at(row).push_back(t);
        }
    }

    return f;
}
{% endhighlight %}

*For now, we have hardcoded height and width to be 100, but this method allows extendability.*

We can then call `generate_floor()` in a loop to populate `Game.floors`. It is now time to draw all our work to the screen.

<br>

## The Game View
In `Game.start()`, we call `view::start()`, which initialises ncurses to modify the terminal to be ready to draw and catch keystrokes. This is done as such:

{% highlight c++ %}
void start() {
    // init ncurses
    initscr();
    keypad(stdscr, true);   // allows us to capture arrow keys
    noecho();               // stops echoing our keypresses
    curs_set(0);            // hides the cursor

    // init colors
    start_color();
    init_color(COLOR_GRAY, 500, 500, 500);  // creates a gray color
    init_pair(COLOR_GRAY, COLOR_GRAY, COLOR_BLACK); // initialises the gray color to draw
}
{% endhighlight %}

`COLOR_GRAY` is a constant I defined to the integer 8. This is because ncurses predefines 8 colors for us as integers [ 0, 7 ]. Also `view` is the name we have given the namespace containing all drawing methods and constants.

In the `view::draw(Game &g)` method, we simply refresh the screen (by starting the method with `erase()` and ending with `refresh()`), and draw all the game windows (map window, secondary window, status window).

Each window has its own draw function associated with it. For now, we're only interested in the map window.

Here is the method to draw our game view.

{% highlight c++ %}
void draw_map(const game::Game &g) {
    auto view_radius = 10;
    int center_row = MAP_Y + (MAP_ROWS/2);
    int center_col = MAP_X + (MAP_COLS/2);

    auto &floor = g.get_floors().at(g.get_current_floor());

    for (int row = -view_radius; row <= view_radius; row++) {
        for (int col = -view_radius; col <= view_radius; col++) {

            int py = g.get_player().get_y() + row;
            int px = g.get_player().get_x() + col;

            if (floor.coordinate_exists(py, px)) {
                int t = floor.get_terrain().at(py).at(px);

                int rely = center_row + row;
                int relx = center_col + col*2;

                attron(COLOR_PAIR(COLOR_GRAY));
                switch (t) {
                    case EMPTY:
                        mvaddstr(rely, relx, ".");
                        break;
                    case WALL:
                        mvaddstr(rely, relx, "#");
                        break;
                }
                attroff(COLOR_PAIR(COLOR_GRAY));
            }
        }
    }

    // draw player last (to appear on top)
    mvaddstr(center_row, center_col, "@");
}
{% endhighlight %}

This is a relatively large code block, but its function is to draw terrain around with respects to the player's position in the world space, keeping the player always in the centre of the window. Notice the view-radius is variable, and this variable very well may become a field of the Mob class in the future (once vision is implemented).

We describe the function `Floor.coordinate_exists(int y, int x)` as if y is in range [0, height] and x is in range [0, width].

We also describe the constants `EMPTY` to be the integer 0, and `WALL` to be the integer 1. This is what populates our `Floor.terrain`. Empty space is drawn using the '.' character, while walls are drawn with '#'.

The only thing we need to do now is add movement and we'll have completed tutorial 001.

<br>

## Player Input and Movement
Returning to the Game class, we recall our `get_input()` method which is called last in the main loop. This method is entirely defined as:

{% highlight c++ %}
void Game::get_input() {
    next_input = getch();
}
{% endhighlight %}

Easy, right?

This stores our keystroke in a field to be processed at the next `update()` call. Speaking of which, our `update()` method is only comprised of a call to the `process_input()` method, which we've defined as follows:

{% highlight c++ %}
void Game::process_input() {
    switch (next_input) {
        case 'q':
            end();
            break;
        case KEY_LEFT:
            player.translate(0, -1);
            break;
        case KEY_RIGHT:
            player.translate(0, 1);
            break;
        case KEY_UP:
            player.translate(-1, 0);
            break;
        case KEY_DOWN:
            player.translate(1, 0);
            break;
    }
}
{% endhighlight %}

*Where translate is a Mob method that translates our x and y values*.

That was the final step. Congratulations! Now, you should have a game in the same state as what was presenting in the above gif. Stay tuned for future parts :)