---
layout: post
title: "Another Reason to Avoid constantize in Rails"
date: 2020-09-14 08:37
comments: true
categories: 
---

## Backstory

Recently, a friend asked me if _just_ calling `constantize` on user input was dangerous, even if subsequent code did not use the result:

```ruby
params[:class].classify.constantize
```

[Brakeman](https://brakemanscanner.org/) generates a "remote code execution" warning for this code:

<pre>
<font color="#00C300">Confidence</font>: <font color="#AA0000">High</font>
<font color="#00C300">Category</font>: Remote Code Execution
<font color="#00C300">Check</font>: UnsafeReflection
<font color="#00C300">Message</font>: Unsafe reflection method `constantize` called with parameter value
<font color="#00C300">Code</font>: <font color="#6C009B">params[:class].classify</font>.constantize
<font color="#00C300">File</font>: app/controllers/users_controller.rb
<font color="#00C300">Line</font>: 7
</pre>

But why? Surely just converting a string to a constant (if the constant even exists!) can't be dangerous, right?

Coincidentally, around that same time I was looking at Ruby deserialization gadgets - in particular [this one](https://www.elttam.com/blog/ruby-deserialization/) which mentions that Ruby's `Digest` module will load a file based on the module name. For example, `Digest::A` will try to `require 'digest/a'`:

<pre>2.7.0 :001 &gt; require <font color="#FF5555"><b>&apos;</b></font><font color="#AA0000">digest</font><font color="#FF5555"><b>&apos;</b></font>
 =&gt; <font color="#55FFFF"><b>true</b></font> 
2.7.0 :002 &gt; <font color="#5555FF"><u style="text-decoration-style:single"><b>Digest</b></u></font>::<font color="#5555FF"><u style="text-decoration-style:single"><b>Whatever</b></u></font>
<font color="#DEDEDE"><b>Traceback</b></font> (most recent call last):
        5: from /home/justin/.rvm/rubies/ruby-2.7.0/bin/irb:23:in `&lt;main&gt;&apos;
        4: from /home/justin/.rvm/rubies/ruby-2.7.0/bin/irb:23:in `load&apos;
        3: from /home/justin/.rvm/rubies/ruby-2.7.0/lib/ruby/gems/2.7.0/gems/irb-1.2.1/exe/irb:11:in `&lt;top (required)&gt;&apos;
        2: from (irb):2
        1: from /home/justin/.rvm/rubies/ruby-2.7.0/lib/ruby/2.7.0/digest.rb:16:in `const_missing&apos;
<font><b>LoadError (</b><u style="text-decoration-style:single"><b>library not found for class Digest::Whatever -- digest/whatever</b></u><b>)</b></font></pre>

The `Digest` library uses the [`const_missing`](https://rdoc.info/stdlib/core/Module:const_missing) hook to implement this functionality.

This made me wonder if `constantize` and `const_missing` could be connected, and what the consequences would be.

## Constantizing in Rails

The `constantize` method in Rails [turns a string into a constant](https://api.rubyonrails.org/classes/String.html#method-i-constantize). If the constant does not exist then a `NameError` will be raised.

However, it is possible to hook into the constant lookup process in Ruby by defining a `const_missing` method. If a constant cannot be found in a given module, and that module has `const_missing` defined, then `const_missing` will be invoked.

<pre>2.7.0 :001 &gt; <font color="#00C300">module</font> <font color="#5555FF"><u style="text-decoration-style:single"><b>X</b></u></font>
2.7.0 :002 &gt;   <font color="#00C300">def</font> <font color="#55FFFF"><b>self</b></font>.<font color="#5555FF"><b>const_missing</b></font>(name)
2.7.0 :003 &gt;     puts <font color="#FF5555"><b>&quot;</b></font><font color="#AA0000">You tried to load #{</font>name.inspect<font color="#AA0000">}</font><font color="#FF5555"><b>&quot;</b></font>
2.7.0 :004 &gt;   <font color="#00C300">end</font>
2.7.0 :005 &gt; <font color="#00C300">end</font>
 =&gt; <font color="#6C009B">:const_missing</font> 
2.7.0 :006 &gt; <font color="#5555FF"><u style="text-decoration-style:single"><b>X</b></u></font>::<font color="#5555FF"><u style="text-decoration-style:single"><b>Hello</b></u></font>
You tried to load :Hello
 =&gt; <font color="#55FFFF"><b>nil</b></font></pre>

If `const_missing` is implemented with behavior based on the constant name, such as loading a file or creating a new object, there is an opportunity for malicious behavior.

## Some Vulnerable Gems

Fortunately, `const_missing` is not used very often. When it is, the implementation is not usually exploitable.

Searching across ~1300 gems, I found only ~40 gems with a `const_missing` implementation.

Of those, the majority were not exploitable because they checked the constant name against expected values or called `const_get` which raises an exception if the constant does not exist.

One gem, [coderay](https://github.com/rubychan/coderay), [loads files based on constant names](https://github.com/rubychan/coderay/blob/d38502167541a1cd1b505a0e468e0098e3ae7538/lib/coderay/helpers/plugin_host.rb#L59-L67) like the Digest library. Also like the Digest library, this does not appear to be exploitable because the files are limited to a single coderay directory.

The next two gems below have memory leaks, which can enable denial of service attacks through memory exhaustion. 

### Temple

The [Temple](https://github.com/judofyr/temple) gem is a foundational gem used by Haml, Slim, and other templating libraries.

In Temple, there is a module called `Temple::Mixins::GrammarDSL` that implements `const_missing` like this:

```ruby
def const_missing(name)
  const_set(name, Root.new(self, name))
end
```

The method creates a new constant based on the given `name` and assigns a new object.

This is a memory leak since constants are never garbage collected. If an attacker can trigger it, they can create an unlimited number of permanent objects, using up as much memory as possible.

Unfortunately, it is easy to exploit this code.

`Temple::Grammar` extends `Template::Mixins::GrammarDSL` and is a core class for Temple. Let's see if it is loaded by Haml, a popular templating library often used with Rails:

<pre>2.7.0 :001 &gt; require <font color="#FF5555"><b>&apos;</b></font><font color="#AA0000">haml</font><font color="#FF5555"><b>&apos;</b></font>
 =&gt; <font color="#55FFFF"><b>true</b></font> 
2.7.0 :002 &gt; <font color="#5555FF"><u style="text-decoration-style:single"><b>Temple</b></u></font>::<font color="#5555FF"><u style="text-decoration-style:single"><b>Grammar</b></u></font>
 =&gt; <font color="#5555FF"><u style="text-decoration-style:single"><b>Temple</b></u></font>::<font color="#5555FF"><u style="text-decoration-style:single"><b>Grammar</b></u></font> </pre>

Great! What happens if we try to reference a module that definitely does not exist?

<pre>2.7.0 :003 &gt; <font color="#5555FF"><u style="text-decoration-style:single"><b>Temple</b></u></font>::<font color="#5555FF"><u style="text-decoration-style:single"><b>Grammar</b></u></font>::<font color="#5555FF"><u style="text-decoration-style:single"><b>DefinitelyDoesNotExist</b></u></font>
 =&gt; #&lt;Temple::Mixins::GrammarDSL::Root:0x000055a79b011060 @grammar=Temple::Grammar, @children=[], @name=:DefinitelyDoesNotExist&gt; </pre>

As can be seen above, the constant is created along with a new object.

To go one step further... does the use of constantize invoke this code?

We can test by loading a Rails console for an application using Haml:

<pre>Loading development environment (Rails 6.0.3.2)
2.7.0 :001 &gt; require <font color="#FF5555"><b>&apos;</b></font><font color="#AA0000">haml</font><font color="#FF5555"><b>&apos;</b></font>
 =&gt; <font color="#55FFFF"><b>false</b></font> 
2.7.0 :002 &gt; <font color="#FF5555"><b>&apos;</b></font><font color="#AA0000">Temple::Grammar::DefinitelyDoesNotExist</font><font color="#FF5555"><b>&apos;</b></font>.constantize
 =&gt; #&lt;Temple::Mixins::GrammarDSL::Root:0x000055ba28031a50 @grammar=Temple::Grammar, @children=[], @name=:DefinitelyDoesNotExist&gt; 
</pre>

It does!

Any Ruby on Rails application using Haml or Slim that calls `constantize` on user input (e.g. `params[:class].classify.constantize`) is vulnerable to a memory leak via this method.

### Restforce

A very similar code pattern is implemented in the [restforce](https://github.com/restforce/restforce) gem.

The [ErrorCode module](https://github.com/restforce/restforce/blob/b14175a9072fb6333c1c1b3dce2bb498c022a1b3/lib/restforce.rb#L65-L67) uses `const_missing` like this:

```ruby
module ErrorCode
  def self.const_missing(constant_name)
    const_set constant_name, Class.new(ResponseError)
  end
end
```

Nearly the same, except this actually creates new _classes_, not just regular objects.

We can verify again:

<pre>Loading development environment (Rails 6.0.3.2)
2.7.0 :001 &gt; require <font color="#FF5555"><b>&apos;</b></font><font color="#AA0000">restforce</font><font color="#FF5555"><b>&apos;</b></font>
 =&gt; <font color="#55FFFF"><b>false</b></font> 
2.7.0 :002 &gt; <font color="#5555FF"><u style="text-decoration-style:single"><b>Restforce</b></u></font>::<font color="#5555FF"><u style="text-decoration-style:single"><b>ErrorCode</b></u></font>::<font color="#5555FF"><u style="text-decoration-style:single"><b>WhateverWeWant</b></u></font>
 =&gt; <font color="#5555FF"><u style="text-decoration-style:single"><b>Restforce</b></u></font>::<font color="#5555FF"><u style="text-decoration-style:single"><b>ErrorCode</b></u></font>::<font color="#5555FF"><u style="text-decoration-style:single"><b>WhateverWeWant</b></u></font> 
</pre>

This time we get as many new classes as we want.

**This has been fixed in [Restforce 5.0.0](https://github.com/restforce/restforce/blob/master/CHANGELOG.md#500-jul-10-2020).**

## Finding and Exploiting Memory Leaks

_Finding_ vulnerable code like this in a production application would be difficult. You would need to guess which parameters might be `constantize`d.

Verifying that you've found a memory leak is a little tricky and the two memory leaks described above create very minimal objects.

From what I could estimate, a new `Rule` object in Temple uses about 300 bytes of memory, while a new class in Restforce was taking up almost 1,000 bytes.

Based on that and my testing, it would take 1 to 4 million requests to use just 1GB of memory.

Given that web applications are usually restarted on a regular basis and it's not usually a big deal to kill off a process and start a new one, this does not seem particularly impactful.

However, it would be annoying and possibly harmful for smaller sites. For example, the base Heroku instance only has 512MB of memory.

Another note here: Memory leaks are not the worst outcome of an unprotected call to `constantize`. More likely it can trigger remote code execution. The real issue I am trying to explore here is the unexpected behavior that may be hidden in dependencies.

## Conclusions

In short: Avoid using `constantize` in Rails applications. If you need to use it, check against an allowed set of class names _before_ calling `constantize`. (Calling `classify` before checking is okay, though.)

Likewise for `const_missing` in Ruby libraries. Doing anything dynamic with the constant name (loading files, creating new objects, evaluating code, etc.) should be avoided. Ideally, check against an expected list of names and reject anything else.

In the end, this comes down to the security basics of not trusting user input and strictly validating inputs.

_Edit:_ It seems some language I used above was a little ambiguous, so I tweaked it. Calling `classify` does not make the code safe - I meant calling `classify` is not dangerous by itself. It's the subsequent call to `constantize` that is dangerous. So you can safely call `classify`, check against a list of allowed classes, then take the appropriate action.
