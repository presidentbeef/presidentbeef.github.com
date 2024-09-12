---
layout: post
title: "DragonRuby: Moving in Arbitrary Directions"
date: 2022-11-15 14:09
comments: true
categories: 
---

In a [previous post](https://dev.to/presidentbeef/dragonruby-rotating-rectangles-2i9e) we looked at rotating rectangles in [DragonRuby](https://dragonruby.org/toolkit/game).

Now let's take that one step further to try turning _and moving_!

In this post, we'll look at very simple movement of a spaceship.

## Setup

First, let's get the state all ready.

The code below puts the sprite of a ship in the middle of the screen.

```ruby
def tick args
  # Setting up initial state
  args.state.ship.w ||= 50
  args.state.ship.h ||= 50
  args.state.ship.x ||= args.grid.center_x - (args.state.ship.w / 2)
  args.state.ship.y ||= args.grid.center_y - (args.state.ship.h / 2)
  args.state.ship.angle ||= 0
  args.state.ship.speed ||= 0

  # Show the ship
  args.outputs.sprites << {
    x: args.state.ship.x,
    y: args.state.ship.y,
    w: args.state.ship.w,
    h: args.state.ship.h,
    path: 'sprites/ship.png',
    angle: args.state.ship.angle,
  }
end
```

![Simple ship in the middle of the window](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g7kchjbljua9wcgss3sm.png)

Not too exciting thus far, but we'll make it better.

## Rotation

For rotation, we'll turn the ship `2.5` degrees when the left or right arrow keys are pressed or held down.

```ruby
def tick args
  # Setting up initial state
  args.state.ship.w ||= 50
  args.state.ship.h ||= 50
  args.state.ship.x ||= args.grid.center_x - (args.state.ship.w / 2)
  args.state.ship.y ||= args.grid.center_y - (args.state.ship.h / 2)
  args.state.ship.angle ||= 0 
  args.state.ship.speed ||= 0 

  # Turn left and right
  if args.inputs.keyboard.right
    args.state.ship.angle += 2.5
  elsif args.inputs.keyboard.left
    args.state.ship.angle -= 2.5
  end

  # Keep angle between 0 and 360
  args.state.ship.angle %= 360

  # Show the ship
  args.outputs.sprites << {
    x: args.state.ship.x,
    y: args.state.ship.y,
    w: args.state.ship.w,
    h: args.state.ship.h,
    path: 'sprites/ship.png',
    angle: args.state.ship.angle,
  }
end
```

![Rotating spaceship](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/e4fsmjga7et4merixxwr.gif)

Important to note DragonRuby puts `0` straight to the right or "due east", with increasing angles rotating counter-clockwise.

![Angles in DragonRuby](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/alqymowrzfxmqp3trt0q.png)

## Acceleration

Still keeping things simple, let's accelerate the ship when the up arrow is held down and decelerate otherwise.

For now we won't worry about what direction the ship is heading.

```ruby
def tick args
  # Setting up initial state
  args.state.ship.w ||= 50
  args.state.ship.h ||= 50
  args.state.ship.x ||= args.grid.center_x - (args.state.ship.w / 2)
  args.state.ship.y ||= args.grid.center_y - (args.state.ship.h / 2)
  args.state.ship.angle ||= 0
  args.state.ship.speed ||= 0

  # Accelerate with up arrow, otherwise decelerate
  if args.inputs.keyboard.up
    args.state.ship.speed += 0.2
  else
    args.state.ship.speed -= 0.1
  end

  # Keep speed between 0 and 10
  args.state.ship.speed = args.state.ship.speed.clamp(0, 10)

  # Turn left and right
  if args.inputs.keyboard.right
    args.state.ship.angle -= 2.5
  elsif args.inputs.keyboard.left
    args.state.ship.angle += 2.5
  end

  # Keep angle between 0 and 360
  args.state.ship.angle %= 360

  # Go?
  args.state.ship.x += args.state.ship.speed

  # Show the ship
  args.outputs.sprites << {
    x: args.state.ship.x,
    y: args.state.ship.y,
    w: args.state.ship.w,
    h: args.state.ship.h,
    path: 'sprites/ship.png',
    angle: args.state.ship.angle,
  }
end
```


![Ship moving to the right](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tt45a5n5mfgkxvr6oa62.gif)

## Moving in Arbitrary Directions

We are almost there!

In the code above, we only update the ship's `x` position. This is just to make sure we have the acceleration/deceleration working how we'd like.

But what we really want is to go in the direction the ship is pointing!

To do so, we need to update both the `x` and `y` position, proportional to the angle and speed... ugh that sounds like we might need some math! Trigonometry even??

Actually, DragonRuby comes to the rescue here! `Integer#vector` will return a unit vector (`[x, y]` where `x` and `y` are between `-1` and `1`) corresponding to the angle.

For example:

```ruby
0.vector # => [1.0, 0.0]
```

So at an angle of 0 degrees, only the `x` direction is affected.

But all we really need to know is that `angle.vector_x` and `angle.vector_y` will give us the "magnitude" we need to convert speed and angle to x and y distance:

```ruby
  args.state.ship.x += args.state.ship.speed * args.state.ship.angle.vector_x
  args.state.ship.y += args.state.ship.speed * args.state.ship.angle.vector_y
```

Putting that into context:

```ruby
def tick args
  # Setting up initial state
  args.state.ship.w ||= 50
  args.state.ship.h ||= 50
  args.state.ship.x ||= args.grid.center_x - (args.state.ship.w / 2)
  args.state.ship.y ||= args.grid.center_y - (args.state.ship.h / 2)
  args.state.ship.angle ||= 0
  args.state.ship.speed ||= 0

  # Accelerate with up arrow, otherwise decelerate
  if args.inputs.keyboard.up
    args.state.ship.speed += 0.2
  else
    args.state.ship.speed -= 0.1
  end

  # Keep speed between 0 and 10
  args.state.ship.speed = args.state.ship.speed.clamp(0, 10)

  # Turn left and right
  if args.inputs.keyboard.right
    args.state.ship.angle -= 2.5
  elsif args.inputs.keyboard.left
    args.state.ship.angle += 2.5
  end

  # Keep angle between 0 and 360
  args.state.ship.angle %= 360

  # Go in the correct direction!
  args.state.ship.x += args.state.ship.speed * args.state.ship.angle.vector_x
  args.state.ship.y += args.state.ship.speed * args.state.ship.angle.vector_y

  # Show the ship
  args.outputs.sprites << {
    x: args.state.ship.x,
    y: args.state.ship.y,
    w: args.state.ship.w,
    h: args.state.ship.h,
    path: 'sprites/ship.png',
    angle: args.state.ship.angle,
  }
end
```

![Spaceship flying around](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/egud8xk5if4ecyj3najt.gif)

## Staying Within Bounds

Just to round this out, let's do Astroids-style wrap-around to keep the ship on the screen.

`args.grid.right` is the width of the screen and `args.grid.top` can be used for the height.

```ruby
def tick args
  # Setting up initial state
  args.state.ship.w ||= 50
  args.state.ship.h ||= 50
  args.state.ship.x ||= args.grid.center_x - (args.state.ship.w / 2)
  args.state.ship.y ||= args.grid.center_y - (args.state.ship.h / 2)
  args.state.ship.angle ||= 0
  args.state.ship.speed ||= 0

  # Accelerate with up arrow, otherwise decelerate
  if args.inputs.keyboard.up
    args.state.ship.speed += 0.2
  else
    args.state.ship.speed -= 0.1
  end

  # Keep speed between 0 and 10
  args.state.ship.speed = args.state.ship.speed.clamp(0, 10)

  # Turn left and right
  if args.inputs.keyboard.right
    args.state.ship.angle -= 2.5
  elsif args.inputs.keyboard.left
    args.state.ship.angle += 2.5
  end

  # Keep angle between 0 and 360
  args.state.ship.angle %= 360

  # Go in the right direction!
  args.state.ship.x += args.state.ship.speed * args.state.ship.angle.vector_x
  args.state.ship.y += args.state.ship.speed * args.state.ship.angle.vector_y

  # Wrap around to keep the ship on the screen
  args.state.ship.x %= args.grid.right
  args.state.ship.y %= args.grid.top

  # Show the ship
  args.outputs.sprites << {
    x: args.state.ship.x,
    y: args.state.ship.y,
    w: args.state.ship.w,
    h: args.state.ship.h,
    path: 'sprites/ship.png',
    angle: args.state.ship.angle,
  }
end
```

![Spaceship flying around but warping between edges](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/n89ri46u96ug7y2e1f40.gif)

## Conclusion

And there we go. In less than 50 lines of code and with no complicated math, we can move an object around in arbitrary directions!
