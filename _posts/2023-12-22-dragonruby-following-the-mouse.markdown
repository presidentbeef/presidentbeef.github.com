---
layout: post
title: "DragonRuby: Following the Mouse"
date: 2023-12-22 14:18
comments: true
categories: 
---

Recently I discovered it is very easy to have objects move towards (or away from) any points in [DragonRuby](https://dragonruby.org/toolkit/game).

This post might be a little easier if you've already read my post on [moving in arbitrary directions](https://dev.to/presidentbeef/dragonruby-moving-in-arbitrary-directions-5eja), but actually the code here is even simpler.

If I skip any explanations here, the concepts should have been covered [earlier in the series](https://dev.to/presidentbeef/series/16166).

### Setup

To get started, let's just output a square (roughly) in the middle of the screen.

`args.grid.center_x` and `args.grid.center_y` are helpful for this instead of remembering/hardcoding the screen size.

In addition, the code uses `args.state.tick_count == 0` to do some setup on the first tick.

```ruby
def tick(args)
  # On the first tick...
  if args.state.tick_count == 0
    # Create a 50x50 pixel square in the middle of the screen
    args.state.player = { x: args.grid.center_x, y: args.grid.center_y, h: 50, w: 50}

    # Output that square on every tick
    args.outputs.static_solids << args.state.player
  end
end
```


![A square in the middle of a window](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/durocwoj9tj6uzzfdp5x.png)

Pretty basic!

(Note I'm skipping straight to [`static_solids`](https://dev.to/presidentbeef/dragonruby-static-outputs-p2c) because that's what I'd prefer in a "real" game.)

### Moving to a Point

Now we'll move the "player" to a given point - in this case where the mouse is. In typical DragonRuby fashion, `args.inputs.mouse` can be used to as a point, even though it has a bunch of other information attached to it.

To get the angle from the player to the mouse, there is a very convenient `angle_to` method! (Also `angle_from` depending on which way you'd like to go.)

Just like `args.inputs.mouse`, the `player` solid can be treated as if it is a point, too.

One the angle is calculated, `vector_x` and `vector_y` will provide the magnitude to move in the `x` and `y` directions.

```ruby
def tick(args)
  if args.state.tick_count == 0
    args.state.player = {
      x: args.grid.center_x,
      y: args.grid.center_y,
      h: 50,
      w: 50
    }

    args.outputs.static_solids << args.state.player
  end

  # Find angle from the square to the current location of the mouse
  angle = args.state.player.angle_to(args.inputs.mouse)
  
  # Move towards the mouse using the unit vector
  args.state.player.x += angle.vector_x
  args.state.player.y += angle.vector_y
end
```

![Square following the mouse around](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rh6qfnxypx8f744jrkww.gif)

And... that's it!! Less than 10 lines of code (without comments/spaces) and it just works.

_So now let's make it more complicated..._

### Centering

It is annoying that the player moves so the _bottom, right-hand_ corner meets the mouse, there is a simple fix: use `anchor_x` and `anchor_y`. DragonRuby will automatically use the anchor point for calculations.

To learn more about anchor points, see this [earlier post](https://dev.to/presidentbeef/smoothly-growing-and-shrinking-sprites-in-dragonruby-53d1). But typically the values are set to `0.5` which means "middle of the object".

```ruby
def tick(args)
  if args.state.tick_count == 0
    args.state.player = {
      x: args.grid.center_x,
      y: args.grid.center_y,
      h: 50,
      w: 50,
      anchor_x: 0.5,
      anchor_y: 0.5,
    }

    args.outputs.static_solids << args.state.player
  end

  angle = args.state.player.angle_to(args.inputs.mouse)
  args.state.player.x += angle.vector_x
  args.state.player.y += angle.vector_y
end
```


![Square following the mouse around, but centered](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0brhvw3f6clhe1jizpjf.gif)

### Moving to Classes

I continue to prefer taking an object/class-based approach in my own code. This makes it much easier to manage as the code grows larger, even if it seems silly for these small examples.

In a little departure from previous posts, I am not walking through the code in detail. (Please check out my other posts in this series to learn more!)

The main difference from the above is adding a `speed` to the movement calculation. Other than that, this is how I generally move from simple code like the above into a structure more suitable (in my opinion) as the code.

```ruby
class Game
  attr_gtk

  def initialize(args)
    # Separate setup method is easier if you need to
    # reset during the game
    setup(args)
  end

  def setup(args)
    # Since static_sprites persists between ticks,
    # need to clear it in case of reset
    args.outputs.static_sprites.clear

    @player = Player.new(x: args.grid.center_x, y: args.grid.center_y, h: 50, w: 50, speed: 2)

    args.outputs.static_sprites << @player
  end

  def tick(args)
    @player.tick(args)
  end
end

class Player
  attr_sprite

  def initialize(x:, y:, h:, w:, speed:)
    @x = x
    @y = y
    @h = h
    @w = w
    @anchor_x = 0.5
    @anchor_y = 0.5
    @speed = speed
  end

  def tick(args)
    angle = self.angle_to(args.inputs.mouse)

    @x += angle.vector_x * @speed
    @y += angle.vector_y * @speed
  end
end

def tick(args)
  $Game ||= Game.new(args)
  $Game.tick(args)
end
```

One minor note if you actually run this code - the square will be white (default for sprites) instead of black (default for solids).
