---
layout: post
title: "Sanitizing, Escaping, and Encoding"
date: 2020-01-14 14:24
comments: true
categories: 
---

_"We need to sanitize this data"_ is a phrase I have heard too many times in the context of web security.
It always makes me a little nervous.

The implication of the term "sanitize" is somehow cleaning the data or rendering it "safe".
But the details of how that safety is achieved are a little vague.

Often it means simply searching for a function containing `sanitize` and blindly using that function.

That is usually the wrong thing!

### Injection Vulnerabilities

Injection vulnerabilities, including [cross-site scripting](https://www.owasp.org/index.php/Top_10-2017_A7-Cross-Site_Scripting_\(XSS\)), are a [top category of web vulnerabilities](https://www.owasp.org/index.php/Top_10-2017_Top_10).

The root cause of injection vulnerabilities is the mixing of code and data which is then handed to a parser (the browser, database driver, shell, etc).
Injection is possible when the data is treated as code. 

([See my talk about injection for a deeper dive!](https://youtu.be/g_24036NDhM))

Since proper escaping or sanitization is the mitigation for injection vulnerabilities, it is important to have a clear understanding of what those terms mean.

### Escaping

The term "escaping" originates from situations where text is being interpreted in some mode
and we want to "escape" from that mode into a different mode.

For example, there are [ANSI "escape codes"](https://en.wikipedia.org/wiki/ANSI_escape_code)
to tell your terminal to switch from a text mode to interpreting a sequence of control characters.

The more common situation is when a developer needs to tell a parser to _not_ interpret a value as code.
For example, when one is writing a string and wants to include a double-quote inside the string:

    "blah\"blah"

The backslash `\` is an escape character that tells the parser to treat the following character as just a value,
not the end of the string literal.

However, especially in web security, when we say "escaping" we typically mean "encoding":

### Encoding

**Encoding** involves replacing special characters with a different representation.

HTML encoding uses [HTML entities](https://developer.mozilla.org/en-US/docs/Glossary/Entity).

For example, `<` would normally be interpreted as the start of an HTML tag.
To display a `<` character without it being interpreted as a tag, use `&lt;`.

In HTML, `&` is the escape character. So now you can see how encoding and escaping are intertwined.

In URLs, encoding involves replacing characters with `%` followed by a hexadecimal number that corresponds
to the ASCII code for that character.

For example, `/` in a URL would normally be interpreted as a path separator.
To pass in `/` without it being interpreted that way, use `%2F`.

This is called "URL encoding" or "[percent encoding](https://en.wikipedia.org/wiki/Percent-encoding)" and the `%` character is the escape character.
The value after `%` is the hex representation of the ASCII code for the desired display character.

Encoding special characters is typically a very simple and straightforward process.
Characters are simply replaced with their encoded value in a linear fashion.

The encoding scheme used depends on context.
For any type of interpretation (HTML, JavaScript, URLs, CSS, SQL, JSON, ...) there will be a different
encoding scheme. It is important to use the correct encoding for the context.

**Also note that encoding is a completely reversible process!**
Given an encoded string, we can easily decode it back to the original value.

### Sanitizing

Unlike encoding and escaping, **sanitization** involves _removing_ characters entirely in order to make the value "safe".

This is a complicated, error-prone process.

Here is a classic example of bad sanitization:

```ruby
# Remove script tags!
def sanitize_js(input)
  input.gsub(/<\/?script>/, "")
end

sanitize_js("<script>alert(1)</script>") # => "alert(1)"

sanitize_js("<scri<script>pt>alert(1)</scr</script>ipt>") # => "<script>alert(1)</script>"
```

This is not just an amusing theoretical example - I have seen this exact approach used in production applications.

Since sanitization is so difficult - nearly impossible - to do correctly, most sanitization implementations have seen
[a number of bypasses](https://www.google.com/search?q=html+sanitize+bypass).

Also, unlike encoding, sanitization is not reversible!
Information is lost when the data is sanitized.
You cannot retrieve the original input once it has gone through a sanitization process.
This is rarely a desirable side-effect.

_Sanitization can also mean removal or replacement of sensitive data.
That is a different usage not being discussed here._

### Using the Right Approach

From a security perspective, contextually encoding untrusted values _at time of use_ is the preferred approach.

The tricky part is understanding the output context of the data and which encoding to use.
HTML can easily have more than four different contexts in a single document!
Also, it makes no sense to use HTML encoding in SQL.

When possible, use encoding routines provided by libraries or frameworks.

Sanitization should be reserved for cases when encoding is simply not possible.
For example, if an application must accept and display HTML from users.
There is no way to use encoding in that scenario.

Again, when possible, do not write your own sanitization! Use existing libraries.

### Summary

When discussing handling potentially dangerous data, be precise with terms!

The security industry seems to have settled on "escaping" to actually mean "encoding".
In other words, a reversible transformation that encodes special characters so
they will not be interpreted as code.

Sanitization, in this context, means an irreversible stripping of special characters.

When possible, prefer encoding/escaping to sanitization!

### See Also

[OWASP Cross-Site Scripting Prevention Cheatsheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet)

[OWASP Injection Prevention Cheatsheet](https://cheatsheetseries.owasp.org/cheatsheets/Injection_Prevention_Cheat_Sheet.html)

[Taking on the King: Killing Injection Vulnerabilities](https://youtu.be/g_24036NDhM)
