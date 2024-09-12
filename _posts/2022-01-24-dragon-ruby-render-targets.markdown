---
layout: post
title: "DragonRuby: Render Targets"
date: 2022-01-24 11:00
comments: true
categories: 
---

Next up in my notes on [DragonRuby](https://dragonruby.itch.io/dragonruby-gtk): render targets!

Weirdly, the [documentation on DragonRuby's render targets](http://docs.dragonruby.org/#----advanced-rendering---simple-render-targets---main-rb) is limited to example code. Personally, I prefer prose when I am trying to learn... so here we are!

In DragonRuby, a render target is like an infinite canvas you can render as many regular sprites onto as you want, then manipulate the whole thing as if it is one sprite.

This is especially good for things like tiled backgrounds that are built once and do not change.

Let's take an example.

## Clouds!

Let's start off very simple and build up.

First, here's all the code to render a single 250x250 pixel image to the screen:

```ruby
def tick args
  clouds = {
    x: 0,
    y: 0,
    h: 250,
    w: 250,
    path: 'sprites/cloud.png'
  }

  args.outputs.sprites << clouds
end
```

![Single cloud tile](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o4bpadbxwln2zp3gsuai.png)

### More Clouds

Cool, but I'd like to fill the whole window with clouds, so I'm going to tile them.

The code below makes a 6x3 grid of the cloud image.

(In DragonRuby, the screen is always 1280x720. Our grid is 1500x750 but I'm not trying to be too precise with the numbers here.)

```ruby
def tick args
  clouds = []

  6.times do |x|
    3.times do |y|
      clouds << {
        x: x * 250,
        y: y * 250,
        h: 250,
        w: 250,
        path: 'sprites/cloud.png'
      }
    end
  end

  args.outputs.sprites << clouds
end
```

![Screen full of blue clouds](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o99o7jjmnfxupewak1n9.png)

Ah... blue clouds. Nice.

On every tick, the code builds up an array of 18 sprites (images) and renders it out to the screen.

(There are a number of ways to make this more efficient - check out the [previous posts](https://dev.to/presidentbeef/series/16166) in this series for different ways to "cache" the sprite information.)

### Render Targets

But in this post we are talking about render targets - which is a way of rendering a bunch of sprites (or any other renderable thing) just once, and then treating the whole group of sprites as a single sprite. This is _faster_, simpler, and enables some neat effects.

The code only needs minor changes to switch the cloud grid to using a render target instead:

```ruby
# Move cloud grid creation into a helper method
def make_clouds(args)
  clouds = []

  6.times do |x|
    3.times do |y|
      clouds << {
        x: x * 250,
        y: y * 250,
        h: 250,
        w: 250,
        path: 'sprites/cloud.png'
      }
    end
  end

  # Similar to `args.outputs`,
  # render targets have `.sprites`, `.solids`, etc.
  # The argument will be used as the path below
  args.render_target(:clouds).sprites << clouds
end

def tick(args)
  # Set up the render target on the first tick
  if args.tick_count == 0
    make_clouds(args)
  end

  # Output a single sprite
  # located at 0,0 and the size of the whole grid
  # created in `make_clouds`
  args.outputs.sprites << {
    x: 0,
    y: 0,
    w: 250 * 6,
    h: 250 * 3,
    path: :clouds # Name of the render target!
  }
end
```

For convenience, the code above moves the creation of the cloud grid and the render target into a helper method which gets called on the first tick of the game.

`args.render_target(:clouds)` automatically creates a new render target named `:clouds` if it does not already exist. Then we can render things to it just as if it were `args.outputs`.

Interestingly, render targets do not seem to have an innate width or height. In order to avoid unintentional scaling, you will need to "know" how big the render target is. In this case, we know it is a 6x3 grid of 250x250 images, so the size is fairly straightforward. I left the math in to make it clearer.

Finally, we reference the render target similarly to an image file, but pass in the name of the render target as the `:path` instead of an actual file path.

## Static Sprites, Too!

[As explored in a different post](https://blog.presidentbeef.com/blog/2022/01/08/dragon-ruby-static-outputs/), we can use `static_sprites` to "render" the sprite _once_.

```ruby
# No changes here
def make_clouds(args)
  clouds = []

  6.times do |x|
    3.times do |y|
      clouds << {
        x: x * 250,
        y: y * 250,
        h: 250,
        w: 250,
        path: 'sprites/cloud.png'
      }
    end
  end

  args.render_target(:clouds).sprites << clouds
end

def tick(args)
  if args.tick_count == 0
    make_clouds(args)

    # Create the clouds sprite once
    # and keep it in `args.state`.
    args.state.clouds = {
      x: 0,
      y: 0,
      w: 250 * 6,
      h: 250 * 3,
      path: :clouds
    }

    # Add the clouds sprite just once
    # as a "static" sprite
    args.outputs.static_sprites << args.state.clouds
  end
end
```

And now we can move the clouds around just by changing the attributes on the render target.

Adding a little bit of code at the end of `tick`:

```ruby
  args.state.clouds.x = (Math.sin(args.tick_count / 20) * 100) - 100
```

(The calculation and numbers aren't really important here, I just fiddled around until something looked decent.)

![Clouds moving back and forth](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hw1qpqotbcg6pfpqo2bq.gif)

Oh hey! Those extra pixels on the sides of the cloud grid actually came in handy.

### What Else?

Remember, the entire render target is like one sprite now. That means all the regular sprite attributes (e.g. color, size, blending, flipping, rotation) can be applied to the entire thing at once.

_Wait, did you say rotation?_

Sure, let's make ourselves dizzy.

```ruby
  args.state.clouds.angle = (Math.sin(args.tick_count / 120) * 180)
```

![Spinning clouds](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g5n2ug9ehwbn8yzh2orq.gif)

Okay, that's as deep as we'll go on render targets in this post!
