---
layout: post
title: "DragonRuby: Rotating Rectangles"
date: 2022-04-30 10:51
comments: true
categories: 
---

In [DragonRuby](https://dragonruby.itch.io/dragonruby-gtk), one of the drawing primitives is a "solid" - a rectangle, actually.

Rectangles are defined by the origin point of the bottom-right corner (`x`, `y`) and a size in height/width (`h`, `w`).

For example, this code paints a black rectangle in roughly the middle of the screen:

```ruby
def tick(args)
  args.outputs.solids << {
    x: 490,
    y: 310,
    w: 300,
    h: 100,
  }
end
```

![DragonRuby window with a black rectangle in the middle](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ejiyfwz8sz2t9eypvgs7.png)

Rectangles are simple and (presumably) fast. But what if we want to put the rectangle at an angle? Or spin it around?

For sprites, there is an `angle` attribute. Will that work for rectangles? Unfortunately, no.

We could make a rectangle image and use it in a sprite, but that seems wasteful. I don't know if resizing a sprite is very resource-intensive, but I'm sure it takes more cycles than changing the size of a simple rectangle.

Fortunately, there is a middle way.

From the 2.26 release notes (and yes, this is the only place I could find this functionality formally documented, though it is used in examples):

> ** [API] Pre-defined ~:pixel~ render target now available.
>    Before ~(boot|tick)~ are invoked, a white solid with a size of 1280x1280
>    is added as a render target. You can use this predefined render target to
>    create solids and get ~args.outputs.sprites~ related capabilities.

In other words, it is possible to create an equivalent rectangle to the above like this:

```ruby
def tick(args)
  args.outputs.sprites << {
    x: 490,
    y: 310,
    w: 300,
    h: 100,
    r: 0,
    g: 0,
    b: 0,
    path: :pixel,
  }
end
```

The key piece is `path: :pixel` and creating a sprite instead of a solid.

As is mentioned above, the `:pixel` [render target](https://dev.to/presidentbeef/dragonruby-render-targets-437k) is white, so to get exactly the same results as before the color is set to black. If no `r``g``b` values were specified, it would be a white rectangle.

Now, can we rotate this rectangle?

Sure!

```ruby
def tick(args)
  args.outputs.sprites << {
    x: 490,
    y: 310,
    w: 300,
    h: 100,
    r: 0,
    g: 0,
    b: 0,
    path: :pixel,
    angle: 45,
  }
end
```

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/iri546i42cs9waajgo91.png)

(By the way, angles in DragonRuby are in degrees.)

### Spinning Round and Round

Here is an example of spinning the rectangle around:

```ruby
def tick(args)
    args.outputs.sprites << {
      x: 490,
      y: 310,
      w: 300,
      h: 100,
      r: 0,
      g: 0,
      b: 0,
      path: :pixel,
      angle: args.tick_count % 360,
    }
end
```

![Black rectangle spinning around its center](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9ghaj7ac17cnpxwgs3yy.gif)

This is great and all, but what if you'd rather have it spin around like this?

![Black rectangle spinning around a corner](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jm0zj01nnjw22xrpv6mo.gif)

With some sleuthing, you might find the `angle_anchor_x` and `angle_anchor_y` attributes. And you would be forgiven for thinking those refer to points on the coordinate grid. But they do not!

Instead, `angle_anchor_x` and `angle_anchor_y` are _percentages relative to the sprite itself_.

In other words, if both anchors are set to `0.5` (the default), the center of the rotation will be the middle of the sprite (half of the width and half of the height).

`0`,`0` is the bottom-left corner of the sprite, as shown above:

```ruby
def tick(args)
    args.outputs.sprites << {
      x: 490,
      y: 310,
      w: 300,
      h: 100,
      r: 0,
      g: 0,
      b: 0,
      path: :pixel,
      angle: args.tick_count % 360,
      angle_anchor_x: 0,
      angle_anchor_y: 0,
    }
end
```

Anchor values between `0` and `1` will be inside the sprite. But values greater than `1` or less than `0` will be outside the sprite.

Here is a little reference:

![Angle anchors on a black rectangle](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fwlbn5o29zou5qvpf7ug.png)

Happy rectangle rotating!

![Five colorful rotating rectangles](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/l8rjfdi9tcvz26iuc9de.gif)
