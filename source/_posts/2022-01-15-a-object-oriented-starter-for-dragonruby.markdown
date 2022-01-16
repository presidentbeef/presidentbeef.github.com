---
layout: post
title: "DragonRuby: Object-Oriented Starter"
date: 2022-01-15 11:00
comments: true
categories: 
---

I enjoy playing with the [DragonRuby Game Toolkit](https://dragonruby.itch.io/dragonruby-gtk), but the [documentation](http://docs.dragonruby.org/) and many of the examples are very much intended for non-Rubyists. Additionally, as a game engine, it's more data/functionally-oriented than most Rubyists are used to. For example, the main game loop in the `tick` method needs to be implemented as a top-level method.

This post walks through structuring a game in a way that is a little more familiar to Rubyists.

_Caveat!_ I am new to DragonRuby myself and this is not meant to be the "correct" or "best" or even "great" way to organize your code. It's just a pattern I've started using and it might be useful for you!

(By the way, while DragonRuby is a commercial product, you can often [grab a free copy](https://dragonruby.itch.io/dragonruby-gtk). Keep an eye out for sales!)

### Starting Example

First, let's start with some code that isn't using any class definitions at all. Everything happens inside `tick`:

```ruby
def tick(args)
  # Set up player object
  args.state.player ||= {
    x: args.grid.w / 2,
    y: args.grid.h / 2,
    w: 20,
    h: 20,
  }

  # Move player based on keyboard input
  if args.inputs.keyboard.left
    args.state.player.x -= 10
  elsif args.inputs.keyboard.right
    args.state.player.x += 10
  elsif args.inputs.keyboard.down
    args.state.player.y -= 10
  elsif args.inputs.keyboard.up
    args.state.player.y += 10
  end

  # Render the "player" as a square to the screen
  args.outputs.solids << args.state.player
end
```

![DragonRuby Example - Moving a black square around](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/79w79at48p6lh0zeo08r.gif)

As a reminder, DragonRuby automatically loads up code from a file called `mygame/app/main.rb` and then calls `tick` 60 times per second.

`args.state` is like an OpenStruct where you can add on whatever attributes you want and store whatever you would like. In this case, we add a hash that we name `player`.

The code then checks for keyboard input and adjusts the position of the "player". (To keep things very simple, we don't worry about keeping the player on the screen.)

Finally, we render the "player" as a solid square.

This is simple enough and the code isn't too complicated. But, just for fun, let's slowly transform it to be a little more "object-oriented".

### Game Object

First, let's create a main "game" object to hold our logic, instead of putting it all in `tick`.

```ruby
class MyGame
  # Adds convenience methods for args, gtk, keyboard, etc.
  attr_gtk

  def initialize(args)
    args.state.player = {
      x: args.grid.w / 2,
      y: args.grid.h / 2,
      w: 20,
      h: 20,
    }
  end

  def tick
    if keyboard.left
      state.player.x -= 10
    elsif keyboard.right
      state.player.x += 10
    elsif keyboard.down
      state.player.y -= 10
    elsif keyboard.up
      state.player.y += 10
    end

    outputs.solids << state.player
  end
end

def tick args
  $my_game ||= MyGame.new(args)
  $my_game.args = args
  $my_game.tick
end
```

Now the `tick` method only sets up the global `$my_game` on the first tick, then sets `args` on each tick and calls the game's `tick` method.

(_Tangent alert!_ Is it necessary to set `args` on every tick? Not strictly - you could set `self.args = args` in `initialize` and it will work okay. But if you want to use DragonRuby's unit test framework, it may cause problems because each test has a fresh copy of `args`.)

Using `attr_gtk` allows the code to be a bit shorter. `args`, `state`, `keyboard`, [and more](http://docs.dragonruby.org/#----attr_gtk-rb) now have convenience methods for them.

### Instance Variables Instead of State

`args.state` is essentially a global variable space. This is a big convenience when a game is all top-level methods - otherwise you would have to figure out where to stash all your game state yourself.

However, it's not required to use it.

The code below uses `@player` to store the player hash, instead of `args.state`.

```ruby
class MyGame
  attr_gtk
  attr_reader :player

  def initialize(args)
    @player = {
      x: args.grid.w / 2,
      y: args.grid.h / 2,
      w: 20,
      h: 20,
    }
  end

  def tick
    if keyboard.left
      player.x -= 10
    elsif keyboard.right
      player.x += 10
    elsif keyboard.down
      player.y -= 10
    elsif keyboard.up
      player.y += 10
    end

    outputs.solids << player
  end
end

def tick args
  $my_game ||= MyGame.new(args)
  $my_game.args = args
  $my_game.tick
end
```

One thing that has thrown me off with DragonRuby is understanding just how much "regular" Ruby I can use. For the most part, other than how the `tick` method is used as the main game loop, you can use the Ruby language constructs you are comfortable with.

### Splitting Things Up

No big change here, but as a game grows it's easier to split the steps of each game loop into different methods.

```ruby
class MyGame
  attr_gtk
  attr_reader :player

  def initialize(args)
    @player = {
      x: args.grid.w / 2,
      y: args.grid.h / 2,
      w: 20,
      h: 20,
    }
  end

  def tick
    handle_input
    render
  end

  def handle_input
    if keyboard.left
      player.x -= 10
    elsif keyboard.right
      player.x += 10
    elsif keyboard.down
      player.y -= 10
    elsif keyboard.up
      player.y += 10
    end
  end

  def render
    outputs.solids << player
  end
end

def tick args
  $my_game ||= MyGame.new(args)
  $my_game.args = args
  $my_game.tick
end
```

### Player Class

Final step in this post - let's move the "player" out to a separate class.

In this example, it might not make a lot of sense. But in most games there will be a lot of state and logic you might want to associate with the "player" or any other objects in the game. Having it be its own class helps keep the logic in one place.

```ruby
class MyGame
  attr_gtk
  attr_reader :player

  def initialize(args)
    @player = Player.new(args.grid.w / 2, args.grid.h / 2)
  end

  def tick
    handle_input
    render
  end

  def handle_input
    if keyboard.left
      player.x -= 10
    elsif keyboard.right
      player.x += 10
    elsif keyboard.down
      player.y -= 10
    elsif keyboard.up
      player.y += 10
    end
  end

  def render
    outputs.solids << player
  end
end

class Player
  attr_sprite

  def initialize(x, y)
    @x = x
    @y = y
    @w = 20
    @h = 20
  end
end

def tick args
  $my_game ||= MyGame.new(args)
  $my_game.args = args
  $my_game.tick
end
```

Here the code uses another DragonRuby convenience. `attr_sprite` adds a bunch of [helper methods](http://docs.dragonruby.org/#----attr_sprite-rb) that allow you to use any object as a sprite/solid/border, etc. (Note that the code still passes `player` into `outputs.solids` and DragonRuby treats it as a solid. If it were passed into `outputs.sprites` then it would be treated like a sprite instead!)

### Separate Files?

For the sake of a blog post, all the code is together. But there is no reason not to start splitting the code across separate files.

But! In DragonRuby there is one weirdness with `require`: you must include the file extension (usually `.rb`), while in regular Ruby that is usually omitted.

## Wrapping Up

Did our code become longer and less straight-forward? Yes, for this small example, definitely.

But as a game (or any project) grows, pulling bits out into modular pieces is going to be an advantage. Personally I fall back on this structure pretty quickly when I start a new DragonRuby project.

Hopefully this post is useful to Rubyists trying to get into DragonRuby!
