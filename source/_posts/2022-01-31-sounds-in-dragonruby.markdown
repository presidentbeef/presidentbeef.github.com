---
layout: post
title: "Sounds in DragonRuby"
date: 2022-01-31 11:00
comments: true
categories: 
---

The sound API in [DragonRuby](https://dragonruby.itch.io/dragonruby-gtk) is a little tricky because at first it seems similar to the sprite API (e.g. using `args.outputs`) but it diverges sharply when you want to have some more control.

So let's take a quick look at audio in DragonRuby!

### Simple Music

As usual with DragonRuby, the simple stuff is simple.

To play a looping sound, for example as background music, get a sound file in `.ogg` format and then add it on to `args.outputs.sounds`:

```ruby
args.outputs.sounds << 'sounds/my_music.ogg'
```

The audio will loop forever.

### Simple Sound Effects

Want to fire off a single sound effect?

Get a sound file in `.wav` format and then add it on to `args.outputs.sounds`:

```ruby
args.outputs.sounds << 'sounds/my_effect.wav'
```

The sound will play once.

_Wait so the API behaves differently based on the file format?_

Yep. A little odd but it works.

### Adjusting the Volume Knob

Playing a sound effect in DragonRuby is pretty easy. But what if we need to adjust the volume or some other attribute of the audio? Or maybe loop a `.wav` or not loop a `.ogg` file?

You might think, based on how sprites work in DragonRuby, you could do something like this:

```ruby
# This does not work!
args.outputs.sounds << {
  input: 'sounds/my_music.ogg',
  gain: 0.5,      # Half volume
  looping: false  # Don't loop
}
```

But this does not work.

Instead, you need to use an entirely different interface: `args.audio`.

`args.audio` is a hash table of sounds.

To add a new sound, assign it with a name:

```ruby
args.audio[:music] = {
  input: 'sounds/my_music.ogg',
  gain: 0.5,
  looping: false
}
```

To adjust an attribute of the audio, access it by the same name:

```ruby
# Turn up the volume
args.audio[:music].gain = 0.75
```

To pause a sound:

```ruby
args.audio[:music].paused = true
```

To completely remove a sound:

```ruby
args.audio.delete :music
```

### More Music Manipulation

This is the full set of options for audio from [the documentation](http://docs.dragonruby.org/#--docs---gtk--args#audio-):

```ruby
args.audio[:my_music] = {
  input: 'sounds/my_music.ogg',
  x: 0.0, y: 0.0, z: 0.0,   # Relative position to the listener, x, y, z from -1.0 to 1.0
  gain: 1.0,                # Volume (0.0 to 1.0)
  pitch: 1.0,               # Pitch of the sound (1.0 = original pitch)
  paused: false,            # Set to true to pause
  looping: false,           # Set to true to loop
}
```

That's it!
