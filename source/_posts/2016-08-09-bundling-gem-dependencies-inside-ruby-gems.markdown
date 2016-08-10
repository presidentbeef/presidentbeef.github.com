---
layout: post
title: "Bundling Dependencies inside Ruby Gems"
date: 2016-08-09 08:53
comments: true
categories: 
---

### Backstory

I recently decided to distribute the [Brakeman](http://brakemanscanner.org/) gem with all its dependencies included.
This was the culmination of a lot of frustration with [Bundler](https://github.com/presidentbeef/brakeman/issues/659), [version conflicts](https://github.com/presidentbeef/brakeman/issues/709), [RubyGem bugs](https://github.com/presidentbeef/brakeman/issues/767#issuecomment-180964293), and trying to maintain compatibility with older versions of Ruby [while libraries did not](https://github.com/presidentbeef/brakeman/pull/602#issuecomment-69494355).

Brakeman is most often used as an *application*, not a library. Yet most Rubyists are used to including *all* dependencies in a `Gemfile` for use with Bundler.
Doing so causes Brakeman's dependencies to be mixed in with users' application dependencies, which doesn't make sense and causes a lot of anguish.

I liken it to having to worry about whether or not your Rails application's dependencies conflict with your browser's. It shouldn't matter.

However, Bundler does not have a way to isolate dependencies for applications like Brakeman, and Bundler is the best way to manage dependencies so we are stuck with it.

Since Brakeman is not normally loaded into users' applications (and I recommend against doing so), its dependencies are separate and should not really matter to the end user.
To this end, I wanted to distribute Brakeman with all its dependencies already inside the gem.

### Bundling Dependencies

Conveniently, Bundler already has a way to do this: `bundle install --standalone`.
This generates a `bundle` directory with two subdirectories `bundler` and `ruby`.

The `bundler` directory just has one file: `setup.rb`. This file adds the bundled gems to the load path. We'll come back to this file later. 

The `ruby` directory has everything you need to run Bundler, along with all of the bundled gems and their executables.
The path to the gems looks something like `ruby/2.3.0/gems/rake-10.1.1/`.
Note this includes the Ruby version and the gem's version.
When `setup.rb` sets up the library paths, it chooses dynamically based on the running Ruby implementation and version (which is not what we want, see below).

### Adding Dependencies

All the dependencies are now there in the `bundle/` directory, but it's still assumed you will be using Bundler.
I would prefer to just load the dependencies myself.

To do so, the Brakeman build script removes the `bundle/bundler/setup.rb` file and generates its own `load.rb` using similar logic.
However, it does not build paths dependent on the running Ruby version because we don't know what the end user will be using.
Instead, it just globs the paths as they are and loads those.

In Brakeman itself, it loads `bundle/load.rb` [lazily](https://github.com/presidentbeef/brakeman/blob/fb4f9de160fd97a2b72d5e01c16058718941ec3d/lib/brakeman.rb#L431-L439) if the file exists. I do not use it in normal testing or development.
In general, all that is needed is to `require` the `load.rb` file inside your code somewhere.

### Building the Gem

All that is left to do is add the bundled gems to the Brakeman gem itself.

Note that Brakeman's Gemfile relies on its gemspec, but the gemspec needs to rely on the bundled gems, leading to a circular dependency.

This simple code is all that is required in the gemspec:

    if File.exist? 'bundle/load.rb'
      s.files += Dir['bundle/ruby/*/gems/**/*'] + ['bundle/load.rb']
    else
      # add dependencies as normal
    end

### Pros

The main advantage of this approach is not polluting application dependencies!
No more version conflicts! No more worries that weird Bundler or gem bugs will break users' installs.

In theory it also makes it easier to distribute Brakeman as a standalone application, if someone were interested in that.

### Cons

The main problem, of course, is that this hides the dependencies.
If you add Brakeman as a dependency and then either load it programmatically or run it with Rake, you may get mysterious library conflicts.
To avoid this, use the "[brakeman-lib](https://rubygems.org/gems/brakeman-lib)" gem, which is the same as the main Brakeman gem but does not bundle dependencies.

It also locks dependencies to a specific version such that updating dependencies requires a new release.
This can be good (avoid breaking with new versions) but it can also be bad if a library has a bug or vulnerability.

### Code

The script I use to build the main Brakeman gem is [here](https://github.com/presidentbeef/brakeman/blob/fb4f9de160fd97a2b72d5e01c16058718941ec3d/build.rb).

Here's the annotated version:

    #!/usr/bin/env ruby
    puts 'Packaging Brakeman gem...'

    # Clean up any existing build artifacts
    system 'rm -rf bundle Gemfile.lock brakeman-*.gem' and

    # Generate gem bundle in ./bundle
    system 'BM_PACKAGE=true bundle install --standalone'

    abort "No bundle installed" unless Dir.exist? 'bundle'

    # Remove the setup.rb file we don't use
    File.delete "bundle/bundler/setup.rb"
    Dir.delete "bundle/bundler"

    # Generate new file to set load paths
    # Code below is a little confusing because it is generating code
    File.open "bundle/load.rb", "w" do |f|

      # Set path at runtime
      f.puts "path = File.expand_path('../..', __FILE__)"

      # Add each gem's lib/ directory to the load path (again at runtime)
      Dir["bundle/ruby/**/lib"].each do |dir|
        f.puts %Q[$:.unshift "\#{path}/#{dir}"]
      end
    end

    # Build the gem
    system "BM_PACKAGE=true gem build brakeman.gemspec"

When bundling gems and building the gem, the script sets the `BM_PACKAGE` variable so that development dependencies [are not included](https://github.com/presidentbeef/brakeman/blob/fb4f9de160fd97a2b72d5e01c16058718941ec3d/brakeman.gemspec#L22) in the bundled gems.
