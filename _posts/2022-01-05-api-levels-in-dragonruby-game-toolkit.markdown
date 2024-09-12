---
layout: post
title: "API Levels in DragonRuby Game Toolkit"
date: 2022-01-05 11:00
comments: true
categories: 
---

[DragonRuby Game Toolkit (DRGTK)](https://dragonruby.org/toolkit/game) is a 2D game engine built with [mRuby](https://mruby.org/), [SDL](https://www.libsdl.org/), and [LLVM](https://llvm.org/). It's meant to be tiny, fast, and allow you to turn out games quickly using [Ruby](https://www.ruby-lang.org/).

Unfortunately, since the documentation is focused on making games _quickly_, I sometimes get lost when trying to figure out how to do things that should be simple. DragonRuby seems to have borrowed Perl's "[There's more than one way to do it](https://en.wikipedia.org/wiki/There%27s_more_than_one_way_to_do_it)" philosophy because for anything you want to do with the API there are _several_ ways to do it.

The documentation and examples tend to focus on the simplest forms (which is fine) but then require digging and experimentation to figure out the rest.

To help explain/document the different API options, this post will go through different methods of rendering images (well, rectangles mostly). 

## API Levels

### Level 0 - Getting Started

The main object one interacts with in DragonRuby is canonically called `args` (always accessible with `$gtk.args`... because there's more than one way!)

To output _things_, like shapes, sprites, or sounds, you can use the "shovel" operator `<<` on `args.outputs` - like `args.outputs.sprites` or `args.outputs.sounds`.

For these "basic" objects, you'll need to shovel _things_ in on every "tick" of the game engine.

This is a complete DragonRuby example to output a rectangle:

```ruby
def tick args
  args.outputs.solids << [100, 200, 300, 400]
end
```

So far, so good.

(You can assume the rest of the examples below are inside a `tick` method if it's not explicitly defined.)

### Level 1 - Arrays

The documentation usually starts off by passing _things_ to `args.outputs` as arrays - essentially positional arguments.

For example:

```ruby
args.outputs.solids << [100, 200, 300, 400]
```

What does that do? I'm not quite sure!

![A black rectangle on a gray background](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p9aobr3f41cbih3kpa95.png)

Okay - it shows a black rectangle on the screen. Not that exciting, but useful enough for our examples.

The problem with using arrays though is remembering which index in the array is which attribute. On top of that, it's not even recommended to pass in arrays because they are slow (for some reason).

### Level 2 - Hashes

What is better than arrays? Hashes! (Hash tables/associative arrays for anyone not familiar with Ruby.)

```ruby
args.outputs.solids << {
  x: 100,
  y: 200,
  w: 300,  # width
  h: 400   # height
}
```

Okay, that's way easier to understand!

And there are more options, too:

```ruby
args.outputs.solids << {
  x: 100,
  y: 200,
  w: 300,
  h: 400,
  r: 255,  # red
  g: 200,  # green
  b: 255,  # blue
  a: 100,  # alpha
  blendmode_enum: 0  # blend mode
}
```

### Level 3 - Primitives

But there is yet another way... instead of using `args.outputs.solids`, `args.outputs.labels`, `args.outputs.sprites`, etc., we can output a hash to `args.outputs.primitives` but mark it as the right primitive "type":

```ruby
args.outputs.primitives << {
  x: 100,
  y: 200,
  w: 300,
  h: 400,
  r: 255,  # red
  g: 200,  # green
  b: 255,  # blue
  a: 100,  # alpha
  blendmode_enum: 0  # blend mode
}.solid!
```

Weird, but okay. Why might one want to do this? See the "Layers" section down below!

### Level 4 - Classes

Finally, probably the most natural for a Rubyist: just use a class!

To do this, you _must_ define all the methods expected for the type of primitive, plus define a method called `primitive_marker` that returns the type of primitive.

```ruby
class ACoolSolid
  attr_reader :x, :y, :w, :h, :r, :g, :b, :a, :blendmode_enum

  def initialize x, y, w, h
    @x = x
    @y = y
    @w = w
    @h = h
  end

  def primitive_marker
    :solid
  end
end

def tick args
  args.outputs.primitives << ACoolSolid.new(100, 200, 300, 400)
end
```

Instead of defining a bunch of methods with `attr_reader`, you can use `attr_sprite` instead which is a DragonRuby shortcut method to do the same thing.

## Layers

DragonRuby renders outputs in this order (from back to front):

* Solids
* Sprites
* Primitives
* Labels
* Lines
* Borders

For each "layer" the objects are rendered in FIFO order - the first things in the queue are rendered first.

But wait... one of these things is not like the others. Doesn't `primitives` just hold things like solids, sprites, labels...?

Yes!

But using `primitives` enables better control over render order.

For example, what if we want to render a rectangle on top of a sprite? With the fixed rendering order above, it's impossible! But by using `args.outputs.primitives` we can do it:

```ruby
def tick args
  a_solid = {
    x: 100,
    y: 200,
    w: 300,
    h: 400
  }.solid!

  a_sprite = {
    x: 100,
    y: 200,
    w: 500,
    h: 500,
    path: 'metadata/icon.png'
  }.sprite!

  args.outputs.primitives << a_sprite << a_solid
end
```

And here's the proof:

![Screenshot showing a black rectangle on top of the DragonRuby logo](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/z1596wyp7n71lwya3f2c.png)

## Every Tick?

`args.outputs.sprites`, etc. get cleared after each call to `tick`. So every tick we have to recreate all the objects and pass them in to `args.outputs`. Seems wasteful, right? Yes, it is!

It's somewhat odd that most DragonRuby examples show creating arrays or hashes for primitives each tick. It made me think somehow the rendering process was destructive - were the things added into `args.outputs` destroyed or modified in some way?

Turns out, no. It is fine to create e.g. a sprite representation once and render the same object each time.

Here we'll use a global for demonstration purposes:

```ruby
class ACoolSolid
  attr_reader :x, :y, :w, :h, :r, :g, :b, :a, :blendmode_enum

  def initialize x, y, w, h
    @x = x
    @y = y
    @w = w
    @h = h
  end

  def primitive_marker
    :solid
  end
end

$a_solid = ACoolSolid.new(10, 20, 30, 40)

def tick args
  args.outputs.primitives << $a_solid
end
```

### But Wait...

We are still shoveling an object into `args.outputs.primitives` each time. Surely that is unnecessary?

Correct! There are [_static_ versions](http://docs.dragonruby.org/#------static_solids-) for each `args.outputs` (e.g. `args.outputs.static_solids`) that do _not_ get cleared every tick.

Naturally, this is more efficient than creating objects and updating the outputs 60 times per second.

We'll explore these options in a future post, but be aware they are available!

