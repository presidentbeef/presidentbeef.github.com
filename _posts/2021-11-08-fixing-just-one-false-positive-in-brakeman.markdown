---
layout: post
title: "Fixing Just One False Positive in Brakeman"
date: 2021-11-08 11:30
comments: true
categories: 
---

A while ago, I came across a [Brakeman](https://brakemanscanner.org/) false positive that I wanted to fix.

For just one false positive, it became a bit of an epic journey.

The code looked something like this:

```ruby
class Task < ApplicationRecord
  enum status: {
    pending: 0,
    success: 1,
    failed: 3,
  }
  NOT_FAILURES = ['pending', 'success'].freeze
end

class TaskRunner
  def get_failures
    start_time = Date.beginning_of_quarter
    end_time = Date.end_of_quarter

    no_failure_enums = Task.statuses.values_at(*Task::NOT_FAILURES)

    query = <<~QUERY
      SELECT COUNT(*)
      FROM `tasks`
      WHERE `tasks`.`status` NOT IN (#{no_failure_enums.join(',')})
      AND `tasks`.`time_end` BETWEEN #{start_time} AND #{end_time}
    QUERY

    # Line below triggers a SQL injection warning
    Task.connection.select_all(query)
  end
end
```

(You can imagine in reality the query is a bit more complicated and justifies writing it this way.)

This code results in an SQL injection warning from Brakeman:

```
Confidence: High
Category: SQL Injection
Check: SQL
Message: Possible SQL injection
Code: Task.connection.select_all("SELECT COUNT(*)\nFROM `filings`\nWHERE `filings`.`status` NOT IN (#{Task.statuses.values_at(*["pending", "success"]).join(",")})\nAND `filings`.`time_end` BETWEEN #{Date.beginning_of_quarter} AND #{Date.end_of_quarter}\n")
File: app/models/task.rb
Line: 25
```

This is a little hard to read, so let's take a look at a better formatted version of the code Brakeman is complaining about:

```ruby
Task.connection.select_all(
  "SELECT COUNT(*) \
  FROM `filings` \
  WHERE `filings`.`status` NOT IN (#{Task.statuses.values_at(*["pending", "success"]).join(",")}) \
  AND `filings`.`time_end` BETWEEN #{Date.beginning_of_quarter} \
  AND #{Date.end_of_quarter}"
)
```

Brakeman is warning about this SQL query because it is using string interpolation to unsafely add in values to the query. If an attacker could control those values, they could modify the SQL run by the database.

In particular, it's warning about
```ruby
Task.statuses.values_at(*["pending", "success"]).join(",")
```
(the value in `no_failure_enums.join(',')`).

However, in this case, we know that `Task.statuses` is actually a constant - it's defined using [`enum`](https://api.rubyonrails.org/v5.2.4.4/classes/ActiveRecord/Enum.html) in the `Task` class. This code is just grabbing the integer values for the given enums and joining them back into a comma-separated string.

So how do we get Brakeman to understand that this value is actually safe?

## Splatted Arrays

Let's dive in!

The call we care about:

```ruby
Task.statuses.values_at(*["pending", "success"]).join(",")
```

First up is `*["pending", "success"]`. This code converts an array of strings to individual method arguments (i.e., `values_at("pending", "success")`.

This is pretty easy to handle. In the case where a splatted array is the only argument to a method, just use the elements of the array as the argument list. (Check out the [pull request here](https://github.com/presidentbeef/brakeman/pull/1564))

This gets us to:

```ruby
Task.statuses.values_at("pending", "success").join(",")
```

Better!

## Hash Values

In this case, `values_at` is [`Hash#values_at`](https://rdoc.info/stdlib/core/Hash:values_at) - it returns an array of values from the hash table for the given keys.

This is also not too difficult to implement. I went ahead and covered `Hash#values` at the same time. (Check out the [pull request here](https://github.com/presidentbeef/brakeman/pull/1595))

Brakeman will now do something like this:

```ruby
h = { a: 1, b: 2, c: 3 }
h.values_at(:a, :c)  #=> [1, 3]
```

Great! Back to our code sample, how does it look now?

```ruby
Task.statuses.values_at("pending", "success").join(",")
```

Oh... it looks exactly the same because Brakeman has no idea what `Task.statuses` is.

Okay, no problem. We just need to implement support for ActiveRecord's `enum`.

## Detour!

Here I took a little detour. `Task.statuses` is a method that returns a hash value. Instead of just implementing that, I thought this would be a good time to support methods with single, simple return values.

For example:

```ruby
class Dog
  def self.sound
    'bark'
  end
end
```

If Brakeman could know that `Dog.sound` returns `'bark'`, I could implement enums as method definitions that return simple arrays or hashes. (More on this later!)

To implement this functionality, I rewrote how Brakeman tracks methods (as real objects) and updated some method lookup code.

The details aren't particularly interesting, but the [code changes are here](https://github.com/presidentbeef/brakeman/pull/1596).

## Back to Enums

Calling `enum` essentially defines a bunch of methods.

For example this code:

```ruby
class Task < ApplicationRecord
  enum status: {
    pending: 0,
    success: 1,
    failed: 3,
  }
end
```

Will define methods like

```ruby
Task.statuses # the one we care about!
Task.status
Task#status
Task#pending?
Task#success?
# ..etc.
```

You can pass in an explicit hash mapping keys to values or just an array of keys and Rails will do the mapping.

To implement this in Brakeman, we simulate the creation of the `status` and `statuses` methods:

```ruby
class Task < ApplicationRecord
  def self.statuses
    {
      pending: 0,
      success: 1,
      failed: 3,
    }
  end
end
```

Then rely on the previous changes for `Task.statuses` now returning a hash.

## Another Detour

You might have noticed that the enum definition uses `status` but we need `statuses`.

This required [a tiny tweak](https://github.com/presidentbeef/brakeman/commit/c27f44ab8bf1d8dfad6c5e5908ba621ec067f9a7) to Brakeman's extremely over-simplified `pluralize`.

## Back to the False Positive

Where are we now?

With `enum` support and proper pluralization, this code:

```ruby
Task.statuses.values_at(*["pending", "success"]).join(",")
```

Gets reduced like this:

```ruby
{
  pending: 0,
  success: 1,
  failed: 3,
}.values_at(*["pending", "success"]).join(",")
```
to
```ruby
{
  pending: 0,
  success: 1,
  failed: 3,
}.values_at("pending", "success").join(",")
```
to
```ruby
[:BRAKEMAN_SAFE_LITERAL, :BRAKEMAN_SAFE_LITERAL].join(",")
```
to
```ruby
"BRAKEMAN_SAFE_LITERAL,BRAKEMAN_SAFE_LITERAL"
```

Well... it's not perfect. Note that the array values are strings, but our enum uses symbol keys. But Brakeman knows the enum is all literal values which are safe, so we end up with this.

The query now looks like:

```ruby
Task.connection.select_all(
  "SELECT COUNT(*) \
  FROM `filings` \
  WHERE `filings`.`status` NOT IN (#{"BRAKEMAN_SAFE_LITERAL,BRAKEMAN_SAFE_LITERAL"}) \
  AND `filings`.`time_end` BETWEEN #{Date.beginning_of_quarter} \
  AND #{Date.end_of_quarter}"
)
```

Again, not perfect but at least Brakeman isn't going to warn about interpolating a string literal into the query.

## Not Done Yet

Ah, but wait. The warning is not gone!

```
Confidence: Medium
Category: SQL Injection
Check: SQL
Message: Possible SQL injection
Code: Task.connection.select_all("SELECT COUNT(*)\nFROM `filings`\nWHERE `filings`.`status` NOT IN (#{"BRAKEMAN_SAFE_LITERAL,BRAKEMAN_SAFE_LITERAL"})\nAND `filings`.`time_end` BETWEEN #{Date.beginning_of_quarter} AND #{Date.end_of_quarter}\n")
File: app/models/task.rb
Line: 25
```

What's wrong now?

Brakeman doesn't know what `Date.beginning_of_quarter` or `Date.end_of_quarter` are, so it generates a lower confidence warning about it. For SQL injection, Brakeman is pretty paranoid about any string interpolation, even if it's not sure the values are "dangerous".

But anything coming from `Date` is likely to be safe, so now [Brakeman ignores `Date` calls in SQL](https://github.com/presidentbeef/brakeman/pull/1615).

## Whew. Done?

Yep - now that code will no longer warn.

Except... in the months it took me to address this false positive, the code has changed in such a way that Brakeman now has a false _negative_ problem (should be warning about some things, but isn't). But that's a different problem for a different day.

At least that one false positive is fixed!
