---
layout: post
title: "Finding Ruby Performance Hotspots via Allocation Stats"
date: 2018-11-28 11:45
comments: true
categories: 
---

[RubyParser](https://github.com/seattlerb/ruby_parser/) is a library written by [Ryan Davis](http://www.zenspider.com/) for parsing Ruby code and producing an abstract syntax tree. It is used by [Brakeman](https://brakemanscanner.org/) and several other static analysis gems.

Recently I was poking around to see if there was any low-hanging fruit for performance improvements.
At first, I was interested in the generated parsers. Racc outputs some *crazy* arrays of state machine changes.
Instead of generating arrays of integers, it outputs arrays of strings, then splits those strings into integers which it loads into the final array.
I thought for sure skipping this and starting with the final array of integers would be faster, but...somehow it wasn't.

I moved on to thinking about [frozen string literals](https://wyeworks.com/blog/2015/12/1/immutable-strings-in-ruby-2-dot-3), which led me to checking String allocations.

### Measuring String Allocations

I found the [allocation_stats](https://github.com/srawlins/allocation_stats) gem very useful for this.

I set up a test like this to read in a file and parse it:

```ruby
require 'ruby_parser'
require 'allocation_stats'

f = File.read(ARGV[0])

rp = RubyParser.new

stats = AllocationStats.trace do
  rp.parse(f, ARGV[0], 40)
end

puts stats.allocations(alias_paths: true).where(class: String).group_by(:sourcefile, :sourceline).sort_by_count.to_text
```

This outputs a report like this (truncated here):

                        sourcefile                      sourceline  count
    --------------------------------------------------  ----------  -----
    <GEM:ruby_parser-3.11.0>/lib/ruby_parser.rb                 20  70686
    <GEM:ruby_parser-3.11.0>/lib/ruby_parser_extras.rb        1361  58154
    <GEM:ruby_parser-3.11.0>/lib/ruby_parser_extras.rb        1362  54672
    <GEM:ruby_parser-3.11.0>/lib/ruby_lexer.rb                 373  19019
    <GEM:ruby_parser-3.11.0>/lib/ruby_lexer.rb                 770  12005
    <GEM:ruby_parser-3.11.0>/lib/ruby_lexer.rex.rb             109   8252
    <GEM:ruby_parser-3.11.0>/lib/ruby_parser_extras.rb        1015   6818

Right away, these look like some juicy targets.

### Version Creation

Let's take a look at the first one:

```ruby
class Parser < Racc::Parser
  include RubyParserStuff

  def self.inherited x
    RubyParser::VERSIONS << x
  end

  def self.version
    Parser > self and self.name[/(?:V|Ruby)(\d+)/, 1].to_i
  end
end
```

On line 8 you can see the `Parser.version` method. RubyParser is actually not just one parser, but multiple parsers for different versions of Ruby.
So there is a `RubyParser` class but also `Ruby18Parser`, `Ruby19Parser`, etc. *and* `RubyParser::V18`, `RubyParser::V19`, etc.
To figure out the version of the current class, the code above grabs the version from the class name itself.

The problem is this code is called *a lot* (70k+ in the example above) to make version-specific decisions during the lexing phase.
This is [fairly easy to fix](https://github.com/presidentbeef/ruby_parser/commit/7274aa6df023981fc3c375a9d22bcde781f2cc3f).

In my testing, this **reduced string allocations by ~25% and parse time by 5-10%.**
One thing I have noticed - and you may also find if you go chasing object allocations in Ruby programs - is that reducing allocations doesn't necessarily help with peak memory use or run time.
It seems the Ruby VM has gotten pretty good at allocating and garbage collecting objects efficiently.

### Debug Code

Let's take a look at the next two large number of String allocations:

                        sourcefile                      sourceline  count
    --------------------------------------------------  ----------  -----
    <GEM:ruby_parser-3.11.0>/lib/ruby_parser_extras.rb        1361  58154
    <GEM:ruby_parser-3.11.0>/lib/ruby_parser_extras.rb        1362  54672

Interesting: just two lines apart, with over 100k allocations between them.

```ruby
def push val
  @stack.push val
  c = caller.first
  c = caller[1] if c =~ /expr_result/
  warn "#{name}_stack(push): #{val} at line #{c.clean_caller}" if debug
  nil
end
```

The two lines of interest are 3 and 4 - the assignments to the local variable `c`, which pull information from `caller`.
`caller` is a fairly expensive method, since it needs to generate a stack trace for the current method call.

Upon a closer look, it's clear the `c` variable is only used in the message on the following line, and that message is only used if the `debug` flag is set.
This means we can wrap all that code in a condition, like this:

```ruby
def push val
  @stack.push val
  
  if debug
    c = caller.first
    c = caller[1] if c =~ /expr_result/
    warn "#{name}_stack(push): #{val} at line #{c.clean_caller}"
  end
  
  nil
end
```

This change **saves 38-50% on string allocations and 20-26% on parse time.**

### Reading Lines

Skipping down a few unavoidable string allocations, there's this one:

                        sourcefile                      sourceline  count
    --------------------------------------------------  ----------  -----
    <GEM:ruby_parser-3.11.0>/lib/ruby_parser_extras.rb        1015   6818

Here's the code:

```ruby
header = str.lines.first(2)
```

RubyParser checks the first couple lines of a file for any comments setting the encoding for the file. The trouble is that calling `String#lines` will split the entire string up when we only need the first two lines.

Grabbing only the first two lines ends up being pretty trivial thanks to Ruby's standard approach of returning enumerators for enumeration methods if a block is not supplied:

```ruby
header = str.each_line.first(2)
```

`String#each_line` will lazily return the lines from the string, so it only does the work needed.

Sadly, this didn't do much for overall string allocations and parse time since this method is only called once, but I think it's a clear improvement to only grab the two lines needed.

### Freezing Strings

Finally, back to the original idea. By the time I made it back to freezing string literals, I was feeling pretty lazy, so I just threw the frozen string header on [`ruby_lexer.rb`](https://github.com/seattlerb/ruby_parser/blob/dd2adeca68471a2de7a8d541fb145972f3e3494f/lib/ruby_lexer.rb):

```ruby
# frozen_string_literal: true
```

Running the tests showed only one method where frozen string literals did not work, so these strings needed to be `dup`ed.

String allocations were reduced by 24-30%, but with almost no parse time change. Probably because these were tiny, tiny strings.

### Final Metrics

With these four changes, **string allocations were reduced by 75-83% and parse time was reduced by 30-37%.** The test suite for RubyParser ran 33% faster on my machine.

I did not see a huge decrease in peak memory use. Maybe 3%. My guess is this is because the String representation in Ruby is fairly well-optimized already (e.g. copy-on-write).

For Brakeman, parsing is a decent part of the run time (30-60% even), so a faster RubyParser definitely makes Brakeman scans faster. From a few test scans, I saw as much as a 30% improvement in total scan time.

### Final Changes

The final version of the changes applied by Ryan are in [this commit](https://github.com/seattlerb/ruby_parser/commit/358e5a058e1eca75c6d6ab075ae31c2cc44827a5).

I expect these improvements will be in the next RubyParser and Brakeman releases.
