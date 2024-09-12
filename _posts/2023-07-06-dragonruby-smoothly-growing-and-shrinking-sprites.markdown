---
layout: post
title: "DragonRuby: Smoothly Growing and Shrinking Sprites"
date: 2023-07-06 14:17
comments: true
categories: 
---

This post is about [DragonRuby](https://dragonruby.org/toolkit/game), a Ruby implementation for writing games. Check it out!

## Starting Off

Let's start simply by rendering a sprite in the middle of the window. For convenience, the program also renders guidelines marking the center of the window (for brevity, not showing this in later code). Also for convenience, this code uses a sprite included with the DragonRuby distribution.

```ruby
def tick args
  # Double size of the original sprite, in pixels
  width = 80 * 2
  height = 80 * 2

  # Output sprite in middle of the window
  args.outputs.sprites << {
    x: args.grid.center_x, # Horizontal center
    y: args.grid.center_y, # Vertical center
    w: width,
    h: height,
    path: 'sprites/hexagon/red.png',
  }

  # Add grid lines to mark center of window
  args.outputs.lines << {
    x: 0,
    y: args.grid.center_y,
    x2: args.grid.w,
    y2: args.grid.center_y
  } <<
  {
    x: args.grid.center_x,
    y: 0,
    x2: args.grid.center_x,
    y2: args.grid.h
  }
end
```

![Off-center rendering of a red hexagon](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vvi2vdzdkylrg1em8j1m.png)

As you can see, we've perfectly rendered a sprite in the center of the - 

Wait, that's not quite right. What happened?

## Anchoring

In DragonRuby, the (0, 0) coordinate is the bottom-left of the window. Similarly, when setting `x` and `y` location for a sprite, those correspond to the bottom-left of the sprite.

This could be solved with some math like `x = x + (sprite.w / 2)` to adjust the sprite appropriately. But this will cause complications later when resizing the sprite.

Preview of the problem:

![Hexagon growing and shrinking off-center](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wqmm70ud4cgikzot23y5.gif)

This _might_ be what you want, but for this post we want the sprite to expand/shrink from the center.

A simpler approach that trying to move `x` and `y` around is to instead use the `anchor_x` and `anchor_y` attributes.

**Important note: `anchor_*` attributes were introduced in DragonRuby 4.8.**

These are very similar to the `angle_anchor_*` attributes [discussed previously](https://dev.to/presidentbeef/dragonruby-rotating-rectangles-2i9e/).

The values used for the anchors are a _percentage_ of the width or height of the sprite. This diagram from the earlier post might help:

![Angle anchors on a black rectangle](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fwlbn5o29zou5qvpf7ug.png)

The default "anchors" are essentially at (0, 0) (technically, they are `nil`, but never mind that).

The center of the sprite is at (0.5, 0.5).

In code:

```ruby
def tick args
  # Double size of the original sprite
  width = 80 * 2
  height = 80 * 2

  # Actually render sprite in middle of the window
  args.outputs.sprites << {
    x: args.grid.center_x,
    y: args.grid.center_y,
    w: width,
    h: height,
    anchor_x: 0.5,
    anchor_y: 0.5,
    path: 'sprites/hexagon/red.png',
  }
end
```

![Red hexagon centered in window](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/y228g59elua7p9d7fyxm.png)

There we go! Digression over, back to squishing and stretching this sprite.

### "Simple" Approach

For the first approach, let's do this:

* Set a rate of growth (e.g. 1 pixel per tick)
* Set a target size
* Grow (or shrink) the size on each tick
* When the target size is met, reverse direction

To avoid some duplication, this code uses the same size for width and height. Adjust as desired.

```ruby
def tick args
  # How big to make the sprite
  target_size = 80 * 2

  # Current size of the sprite
  args.state.size ||= 0

  # How fast to grow
  args.state.growth_rate ||= 1

  # If the target size is reached, reverse
  if args.state.size >= target_size
    args.state.growth_rate = -1
  elsif args.state.size <= 0
    args.state.growth_rate = 1
  end

  # Grow (or shrink) the size
  args.state.size += args.state.growth_rate

  args.outputs.sprites << {
    x: args.grid.center_x,
    y: args.grid.center_y,
    w: args.state.size,
    h: args.state.size,
    path: 'sprites/hexagon/red.png',
    anchor_x: 0.5,
    anchor_y: 0.5,
  }
end
```

Result:

![Red hexagon growing and shrinking in the center of the window](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/c233nxl7im2qxmi87pmx.gif)

Nailed it. Post over..?

### Easing In

Instead of calculating the growth rate "by hand," wouldn't it be nice if a function could do that for us? Maybe even have the ability to vary the growth rate over time?

`args.easing.ease` is here for that very reason.

([This video](https://www.youtube.com/watch?v=mr5xkf6zSzk), linked in the DragonRuby docs, is _really_ good. Plus it also will explain the names of the easing functions used below.)

`args.easing.ease` will return a "percentage" value (between 0 and 1) which can be multiplied against the target value to get the current value. For the example here, that means it will calculate the current percentage of the size of the the sprite.

```ruby
def tick args
  target_size = 80 * 2
  duration = 60

  args.state.start_time ||= 0
  args.state.easing_function = :identity

  # Calculate percentage (0 to 1) of progress based
  # on the start time, current time, duration, and easing function
  percentage = args.easing.ease args.state.start_time,
                                args.state.tick_count,
                                duration,                         
                                args.state.easing_function

  # Output the scaled image
  args.outputs.sprites << {
    x: args.grid.center_x,
    y: args.grid.center_y,
    w: target_size * percentage,
    h: target_size * percentage,
    path: 'sprites/hexagon/red.png',
    anchor_x: 0.5,
    anchor_y: 0.5,
  }
end
```

This code uses the `:identity` function which is linear - essentially the same as the earlier code that adds a constant "growth rate" on each tick.

Here is the result:

![Red hexagon growing, but not shrinking](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/eugztn22l0aq6j95olyt.gif)

Oops, forgot to shrink it back down!

### Easing Out

The code is going to get slightly more complicated now. Instead of passing in a single easing function, the new code uses an array of function names. The array is "splatted" into `args.ease.easing`.

When the set duration is up, the code adds `:flip` to the list of functions. Instead of going from 0 to 1, the percentage will now go from 1 to 0.

When that's over, `:flip` is removed from the list and it starts all over.

```ruby
def tick args
  target_size = 80 * 2
  duration = 60

  args.state.start_time ||= 0
                                                                               
  # List of easing functions                                                   
  args.state.easing_functions ||= [:identity]                                  
                                                                               
  # Calculate percentage (0 to 1) of progress based                            
  # on the start time, current time, duration, and easing function(s)          
  percentage = args.easing.ease args.state.start_time,                         
                                      args.state.tick_count,                   
                                      duration,                                
                                      *args.state.easing_functions             
                                                                               
  # When we reach the end of the duration, switch direction                    
  if args.state.tick_count == args.state.start_time + duration                 
    # Reset the start time for the easing function
    args.state.start_time = args.state.tick_count                              
                                                                               
    if args.state.easing_functions == [:identity]                              
      args.state.easing_functions = [:identity, :flip]                         
    else                                                                       
      args.state.easing_functions = [:identity]                                
    end                                                                        
  end                                                                          
                                                                               
  # Output the scaled image                                                    
  args.outputs.sprites << {                                                    
    x: args.grid.center_x,                                                     
    y: args.grid.center_y,                                                     
    w: target_size * percentage,                                               
    h: target_size * percentage,                                               
    path: 'sprites/hexagon/red.png',                                           
    anchor_x: 0.5,                                                             
    anchor_y: 0.5,                                                             
  }
end
```

![Red hexagon growing and shrinking](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uje6fhszquw966wcsl7o.gif)

One way to think of `[:identity, :flip]` is like this:

`:identity` is _f(x) = x_ and `:identity, :flip` is _g(x) = 1 - f(x)_. `:flip` can be used to "reverse" any function.

### More Easing

This may not be _very_ exciting, but keep in mind there are several pre-defined easing functions:

* `:identity` (f(x) = x)
* `:quad` (f(x) = x^2)
* `:cube` (f(x) = x^3)
* `:quint` (f(x) = x^4)
* `:smooth_start_quad` (same as `:quad`)
* `:smooth_start_cube` (same as `:cube`)
* `:smooth_start_quart` (same as `:quart`)
* `:smooth_start_quint` (same as `:quint`)
* `:smooth_stop_quad` (f(x) = 1 - (1 - x)^2)
* `:smooth_stop_cube` (f(x) = 1 - (1 - x)^3)
* `:smooth_stop_quart` (f(x) = 1 - (1 - x)^4)
* `:smooth_stop_quint` (f(x) = 1 - (1 - x)^5)

Mix and match as you'd like... for example, here's growing with `:cube` but shrinking with `:quint`:

```ruby
def tick args
  target_size = 80 * 2
  duration = 60

  args.state.start_time ||= 0
                                                                               
  # List of easing functions                                                   
  args.state.easing_functions ||= [:cube]                                  
                                                                               
  # Calculate percentage (0 to 1) of progress based                            
  # on the start time, current time, duration, and easing function(s)          
  percentage = args.easing.ease args.state.start_time,                         
                                args.state.tick_count,                   
                                duration,                                
                                *args.state.easing_functions             
                                                                               
  # When we reach the end of the duration, switch direction                    
  if args.state.tick_count == args.state.start_time + duration                 
    args.state.start_time = args.state.tick_count                              
                                                                               
    if args.state.easing_functions == [:cube]                              
      args.state.easing_functions = [:quint, :flip]                         
    else                                                                       
      args.state.easing_functions = [:cube]                                
    end                                                                        
  end                                                                          
                                                                               
  # Output the scaled image                                                    
  args.outputs.sprites << {                                                    
    x: args.grid.center_x,                                                     
    y: args.grid.center_y,                                                     
    w: target_size * percentage,                                               
    h: target_size * percentage,                                               
    path: 'sprites/hexagon/red.png',                                           
    anchor_x: 0.5,                                                             
    anchor_y: 0.5,                                                             
  }
end
```

![Hexagon growing, but then shrinking faster than it grew](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/l9qmcrd0kpg8lnj95nq8.gif)

### Closing Out

This post demonstrates two concepts: changing the anchors for a sprite, and using "easing" to set the size of a sprite.

Easing is a general-purpose concept that can be used for any applications, such as smooth movement.

Keep in mind, you can also:

* Change up the anchors
* Grow/shrink width and height separately
* Change up the target width/height as desired (for example, grow all the way but only shrink back a little)
* [Combine several easing functions together](https://docs.dragonruby.org/#/api/easing?id=combining-easing-definitions)
* [Write your own easing functions](https://docs.dragonruby.org/#/api/easing?id=custom-easing-functions)
* ???

Have fun!
