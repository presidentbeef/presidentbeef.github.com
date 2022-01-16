---
layout: post
title: "DragonRuby: Static Outputs"
date: 2022-01-08 11:00
comments: true
categories: 
---

In a [previous post](https://dev.to/presidentbeef/api-levels-in-dragonruby-game-toolkit-4jb4) we looked at different ways to render outputs (sprites, rectangles, lines, etc.) in the [DragonRuby Game Toolkit](https://dragonruby.org/toolkit/game).

The post ended by hinting at a more efficient way to render outputs instead of adding them to e.g. `args.outputs.solids` or `args.outputs.sprites` each tick.

This post explores the world of "static outputs"!

## Static What?

First of all, we should address the most confusing part of all this.

**"Static" does not mean the images don't move or change.** Instead, it means that render "queue" is not cleared after every tick.

Normally, one would load up the queue each tick, like this:

```ruby
def tick args

  # Render a black rectangle
  args.outputs.solids << {
    x: 100,
    y: 200,
    w: 300,  # width
    h: 400   # height
  }
end
```

But this is kind of wasteful. We are creating a new hash table each tick (60 ticks/second) and then throwing it away. Also each tick we are filling up the `args.outputs.solids` queue and then emptying it.

Instead, why not create the hash table once, load up the queue once, and then re-use them?

That's the idea of static outputs!

There are static versions for each rendered type:

* `args.outputs.static_borders`
* `args.outputs.static_labels`
* `args.outputs.static_primitives`
* `args.outputs.static_solids`
* `args.outputs.static_sprites`

## Going Static

### Starting Out

Here's an example with comments explaining what the code is doing. This "game" simply moves a square back and forth across the screen. This is the entire program!

```ruby
def tick args
  # Initialize the x location of the square
  args.state.x ||= 0

  # Initialize the direction/velocity
  args.state.direction ||= 10

  # If we hit the sides, change direction
  if args.state.x > args.grid.right or args.state.x < args.grid.left
    args.state.direction = -args.state.direction
  end

  # Update the x location
  args.state.x += args.state.direction

  # Build the square
  square = {
    x: args.state.x,
    y: 400,
    w: 40,
    h: 40,
  }

  # Add the square to the render queue
  args.outputs.solids << square
end
```

The resulting output looks like:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o56m4w483wxr46y07zh0.gif)

This example introduces `args.state`. This is basically a persistent bag you can throw anything into. (For Rubyists - this is like [OpenStruct](https://ruby-doc.org/stdlib-3.1.0/libdoc/ostruct/rdoc/OpenStruct.html).)

`x` and `direction` are not special, they are just variables we are defining. We use `||=` to initialize them because we only want to set the values on the first tick.

This example illustrates the point from above - every tick it creates a new square and adds it to the queue. The queue is emptied out and then the code starts all over again.

Seems wasteful, right?

### Caching the Objects

First thing I think of is - "why not create the square once, then just update the object each tick? Does that work?" Yes! It does.

```ruby
def tick args
  args.state.direction ||= 10
  args.state.square ||= {
    x: 0,
    y: 400,
    w: 40,
    h: 40,
  }

  if args.state.square[:x] > args.grid.right or args.state.square[:x] < args.grid.left
    args.state.direction = -args.state.direction
  end

  args.state.square[:x] += args.state.direction

  args.outputs.solids << args.state.square
end
```

In this code, we create the `square` only once and then store it in `args.state.square`.

Instead of having a separate `x` variable, the code updates the `x` property on the square directly.

This is _better_, but we are still updating `args.outputs.solids` each tick.

### Full Static

```ruby
def tick args
  args.state.direction ||= 10
  args.state.square ||= {
    x: 0,
    y: 400,
    w: 40,
    h: 40,
  }

  # On the first tick, add the square to the render queue
  if args.tick_count == 0
    args.outputs.static_solids << args.state.square
  end

  if args.state.square[:x] > args.grid.right or args.state.square[:x] < args.grid.left
    args.state.direction = -args.state.direction
  end

  args.state.square[:x] += args.state.direction
end
```

In this code, we use the fact that the first `args.tick_count` is `0` to add the `square` to `args.outputs.static_solids` just _once_. It will continue to be rendered on each tick.

## Performance

Intuitively, since the code is doing less, it should be faster. But does it really make a difference?

It depends on your game, how much it's doing per tick, how many sprites you are rendering, and what platform/hardware it's running on.

The examples above? Not going to see any difference using `static_solids`.

But DragonRuby contains two examples that directly compare `args.outputs.sprites` vs. `args.outputs.static_sprites` ([here](http://docs.dragonruby.org/#----performance---sprites-as-classes---main-rb) and [here](http://docs.dragonruby.org/#----performance---static-sprites-as-classes---main-rb)).

In these examples, you can play with the number of "stars" rendered to see different performance. On my ancient laptop, I do not see a performance difference until around 3,000 stars.

Your mileage may vary, though!

## Should I Always Use the Static Versions?

It depends! Probably not?

If your code mainly manipulates the same objects around the screen _and_ always renders them in the same order, then using the `static_` approach might be simpler and faster.

But in many cases it might be easier to simply set up the render queues each tick, especially if the objects rendered or their ordering change regularly. Otherwise, managing the state of the rendering queues can become cumbersome. (We haven't even talked about clearing the static queues, for example.)

Some of this comes down to personal preference and how you would like to structure your code. But hopefully this post has helped explain how to use the `args.outputs.static_*` methods in your game!
