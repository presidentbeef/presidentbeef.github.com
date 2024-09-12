---
layout: post
title: "DragonRuby: Basic Sprite Animation"
date: 2022-06-26 14:11
comments: true
categories: 
---

Animating sprites in [DragonRuby](https://dragonruby.org/) is fairly simple, but it does require putting a couple ideas together.

First, it's best to have a single image with all frames of the animation together, equally spaced apart. I prefer the frames are arranged horizontally from left-to-right, so that is what we will use here.

Here is an example, [borrowed from here](https://kenney.nl/assets/toon-characters-1):

![Animation frames of a walking adventurer](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jv9eg653bi1mdwk3ea8m.png)

The first frame can be displayed like this:

```ruby
def tick args
  height = 195
  width = 192

  args.outputs.sprites << {
    x: args.grid.center_x - (width / 2),
    y: args.grid.center_y - (height / 2),
    h: height,
    w: width,
    source_x: 0,
    source_y: 0,
    source_w: width,
    source_h: height,
    path: 'sprites/walking.png',
  }
end
```

`source_x` and `source_y` set the _bottom left_ corner of a "tile" or basically a slice of the image. (To use the _top_ left instead, set `tile_x` and `tile_y`). `source_w` and `source_h` set the width and height of the tile. The sprite can be scaled when displayed with `w` and `h`.

![Single frame of adventurer](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/a4t2zh97adawgjgo5em8.png)

If the frames are laid out horizontally, then all one needs to do is update the `source_x` value (typically by the width of the tile) in order to change the frame.

Here is an illustration for a few frames:

![Frame index illustration](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9w8p5wzlp482b8c6gg8k.png)

We could accomplish this by using the multiplying the width of the tile by the current tick (modulo the number of frames, so it loops):

```ruby
def tick args
  height = 195
  width = 192
  num_frames = 8

  source_x = width * (args.tick_count % num_frames)

  args.outputs.sprites << {
    x: args.grid.center_x - (width / 2),
    y: args.grid.center_y - (height / 2),
    h: height,
    w: width,
    source_x: source_x,
    source_y: 0,
    source_w: width,
    source_h: height,
    path: 'sprites/walking.png',
  }
end
```

This works... but it's a bit fast for a walk!

![Very fast walk](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/aeeeimw4tx4a9gjg9egh.gif)

This is where DragonRuby helps out. The [`frame_index` method](https://docs.dragonruby.org/#/api/numeric?id=frame_index) will do the calculation of the current frame for us.

`frame_index` accepts these arguments:
* `count`: total number of frames in the animation
* `hold_for`: how many ticks to wait between frames
* `repeat`: whether or not to loop

`frame_index` can be called on any integer, but typically uses the tick number on which the animation started. Below, the code sets this to `0` (the first tick). This could instead be when an event happens, based on input, or anything else.

Multiplying the `width` of the tile by the frame index results in the `source_x` value for the current frame of the animation:

```ruby
def tick args
  height = 195
  width = 192
  num_frames = 8
  start_tick = 0
  delay = 4

  source_x = width * start_tick.frame_index(count: num_frames, hold_for: delay, repeat: true)

  args.outputs.sprites << {
    x: args.grid.center_x - (width / 2),
    y: args.grid.center_y - (height / 2),
    h: height,
    w: width,
    source_x: source_x,
    source_y: 0,
    source_w: width,
    source_h: height,
    path: 'sprites/walking.png',
  }
end
```

![Slower walk](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7wnj69rllj4561g02hvg.gif)

And that's it!

## But With Ruby Classes

Once a game starts to get moderately complex, I like to arrange behavior into classes. It's also convenient to use `attr_gtk` to avoid passing `args` around and to save on some typing (e.g. `args.outputs` becomes just `outputs`).

```ruby
class MyGame
  attr_gtk

  def initialize(args)
    @my_sprite = MySprite.new(args.grid.center_x, args.grid.center_y)
    args.outputs.static_sprites << @my_sprite
  end

  def tick
    if inputs.mouse.click
      if @my_sprite.running?
        @my_sprite.stop
      else
        @my_sprite.start(args.state.tick_count)
      end
    end

    @my_sprite.update
  end
end

class MySprite
  attr_sprite

  def initialize x, y
    @x = x
    @y = y

    @w = 192
    @h = 195
    @source_x = 0
    @source_y = 0
    @source_w = @w
    @source_h = @h
    @path = 'sprites/walking.png'

    @running = false
  end

  # Set @running to the current tick number
  # this is so the frame_index can use that as the
  # start of the animation timing.
  def start(tick_count)
    @running = tick_count
  end

  def stop
    @running = false
  end

  def running?
    @running
  end

  # Update source_x based on frame_index
  # if currently running
  def update
    if @running
      @source_x = @source_w * @running.frame_index(count: 8, hold_for: 4, repeat: true)
    end
  end
end

def tick args
  $my_game ||= MyGame.new(args)
  $my_game.args = args
  $my_game.tick
end
```

This example essentially follows my [Object-Oriented Starter](https://dev.to/presidentbeef/dragonruby-object-oriented-starter-3plg) approach and moves the logic into a game class and a sprite class.

When the mouse is clicked, the sprite starts moving (using the current `tick_count` as the starting tick). When the mouse is clicked again, the sprite stops.

## Source vs. Tile

To use just a piece of an image (for animations or otherwise), there are two options: `source_(x|y|h|w)` or `tile_(x|y|h|w)`.

These options are nearly identical, except `source_y` is bottom left and `tile_y` is top left.

The `source_` options were added in DragonRuby 1.6 and are more consistent with the rest of DragonRuby where the origin is the bottom left. On the other hand, the `tile_` options align easier with image editors.

Either option works, depending on what is important to you.

## Go!

Now that's really it! Get moving!
