---
layout: post
title: "Why 'Escaping' JavaScript is Dangerous"
date: 2020-05-12 11:40
comments: true
categories: 
---

A recent [vulnerability report](https://github.com/rails/rails/security/advisories/GHSA-65cv-r6x7-79hv) and [the blog post behind it](https://chefsecure.com/blog/i-found-xss-security-flaws-in-rails-heres-what-happened) brought my attention back to the [`escape_javascript`](https://api.rubyonrails.org/v6.0.0/classes/ActionView/Helpers/JavaScriptHelper.html#method-i-escape_javascript) Ruby on Rails helper method.

<blockquote class="twitter-tweet" data-dnt="true"><p lang="en" dir="ltr">Let me say it again... if you are calling `escape_javascript` or `j` in your Rails code, please don&#39;t! <a href="https://t.co/60KLEjHX3T">https://t.co/60KLEjHX3T</a></p>&mdash; Justin Collins (@presidentbeef) <a href="https://twitter.com/presidentbeef/status/1258917835977318400?ref_src=twsrc%5Etfw">May 9, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

It's bad form to drop blanket statements without explanation or evidence, so here it is:

### Escaping HTML

Part of the danger of `escape_javascript` is the **name** and apparent relationship to [`html_escape`](https://api.rubyonrails.org/v6.0.0/classes/ERB/Util.html#method-c-html_escape).

HTML is a markup language for writing documents. Therefore, it must have a method for representing itself in text.
In other words, there must be a way to encode `<b>` such that the browser *displays* `<b>` and does not interpret it as HTML.

As a result, HTML has a well-defined HTML encoding strategy.
In the context of security and [cross-site scripting](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html), if a value output in an HTML context is *HTML escaped*, it is safe - the value will not be interpreted as HTML.

_([See my post all about escaping!](https://blog.presidentbeef.com/blog/2020/01/14/injection-prevention-sanitizing-vs-escaping/))_

### Escaping Javascript

On the other hand, JavaScript has no such escaping requirements or capabilities.

Therefore, the "escaping" performed by `escape_javascript` is limited.
The [vulnerability report](https://github.com/rails/rails/security/advisories/GHSA-65cv-r6x7-79hv) states the method is for "escaping JavaScript string literals".

In particular, `escape_javascript` is only useful in one, single context: inside JavaScript strings!

For example:

```erb
# ERb Template
<script>
  var x = '<%= escape_javascript some_value %>';
</script>
```

**Use of `escape_javascript` in any other context is incorrect and dangerous!**

This is and always has been dangerous (note the missing quotes):

```erb
# ERb Template
<script>
  var x = <%= escape_javascript some_value %>;
</script>
```

`some_value` could be a payload like `1; do_something_shady(); //` which would result in the following HTML:

```html
<script>
  var x = 1; do_something_shady(); //; 
</script>
```

The `escape_javascript` helper **does not** and **cannot** make arbitrary values inserted into JavaScript "safe" in the same way `html_escape` makes values safe for HTML.

### CVE-2020-5267

[Jesse's post](https://chefsecure.com/blog/i-found-xss-security-flaws-in-rails-heres-what-happened) has more details, but here's the gist: JavaScript added a new string literal. Instead of just single and double-quotes, now there are also backticks `` ` `` which support string interpolation (like Ruby!).

This meant it was simple to bypass `escape_javascript` and execute arbitrary JavaScript by using a backtick to break out of the string or just `#{...}` to execute code during interpolation.

For example, if this were our code: 

```erb
# ERb Template
<script>
  var x = `<%= escape_javascript some_value %>`;
</script>
```

Then if `some_value` had a payload of ```; do_something_shady(); //``, the resulting HTML would be:

```html
<script>
  var x = ``; do_something_shady(); //`
</script>
```

This is because `escape_javascript` was not aware of backticks for strings.

### Dangers of Dynamic Code Generation

<blockquote class="twitter-tweet" data-dnt="true"><p lang="en" dir="ltr">Let me say it again… using dynamic javascript under practically any circumstance is inviting trouble. It might be ok. I’d rather not have to worry about it. <a href="https://t.co/wnPy3OnkKI">https://t.co/wnPy3OnkKI</a></p>&mdash; Shake, Oreo (@ndm) <a href="https://twitter.com/ndm/status/1258982525172510720?ref_src=twsrc%5Etfw">May 9, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

As I have [talked about before](https://youtu.be/g_24036NDhM), web applications are essentially poorly-defined compilers generating code with untrusted inputs.
In the end, the server is just returning a mishmash of code for the browser to interpret.

However, directly trying to generate safe code in a Turing-complete language like JavaScript or Ruby via string manipulation is a risky game.
Methods like `escape_javascript` make it tempting to do so because the name sounds like it will make the code safe.

If at all possible, avoid dynamic code generation!
