---
layout: post
title: "Rails 6.1 SQL Injection Updates"
date: 2021-07-21 12:14
comments: true
categories: 
---

Since early 2013, I have been maintaining [rails-sqli.org](https://rails-sqli.org), a collection of Rails ActiveRecord methods that can be vulnerable to [SQL injection](https://guides.rubyonrails.org/security.html#sql-injection).

Rails 6 has been out [since December 2019](https://guides.rubyonrails.org/6_0_release_notes.html), but sadly the site has been missing information about changes and new methods in Rails 6.

As that deficiency has recently been rectified, let's walk through what has changed since Rails 5!

## `delete_all`, `destroy_all`

In earlier versions of Rails, `delete_all` and `destroy_all` could be passed a string of raw SQL.

In Rails 6, these two methods no longer accept any arguments.

Instead, you can use...

## `delete_by`, `destroy_by`

New in Rails 6, [`delete_by`](https://api.rubyonrails.org/v6.1.4/classes/ActiveRecord/Relation.html#method-i-delete_by) and [`destroy_by`](https://api.rubyonrails.org/v6.1.4/classes/ActiveRecord/Relation.html#method-i-destroy_by) accept the same type of arguments as `where`: a Hash, an Array, or a raw SQL String.

This means they are vulnerable to the same kind of SQL injection.

For example:

```ruby
params[:id] = "1) OR 1=1--"
User.delete_by("id = #{params[:id]}")
```

Resulting query that deletes all users:
```sql
DELETE FROM "users" WHERE (id = 1) OR 1=1--)
```

## `order`, `reorder`

Prior to Rails 6, it was possible to pass arbitrary SQL to the [`order`](https://api.rubyonrails.org/v6.1.4/classes/ActiveRecord/QueryMethods.html#method-i-order) and [`reorder`](https://api.rubyonrails.org/v6.1.4/classes/ActiveRecord/QueryMethods.html#method-i-reorder) methods.

Since Rails did not offer an easy way of setting sort direction, this kind of code was common:

```ruby
User.order("name #{params[:direction]}")
```

In Rails 6.0, injection attempts would raise a deprecation warning:

```ruby
DEPRECATION WARNING: Dangerous query method (method whose arguments are used as raw SQL) called with non-attribute argument(s): "--". Non-attribute arguments will be disallowed in Rails 6.1. This method should not be called with user-provided values, such as request parameters or model attributes. Known-safe values can be passed by wrapping them in Arel.sql().
```

Starting with Rails 6.1, some logic to check the arguments to `order`. If the arguments do not appear to be column names or sort order, they will be rejected:

```ruby
> User.order("name ARGLBARGHL")
Traceback (most recent call last):
        1: from (irb):12
ActiveRecord::UnknownAttributeReference (Query method called with non-attribute argument(s): "name ARGLBARGHL")
```

It is still possible to inject additional columns to extract some information from the table, such as number of columns or names of the columns:

```ruby
params[:direction] = ", 8"
User.order("name #{params[:direction]}")
```

Resulting exception:
```
ActiveRecord::StatementInvalid (SQLite3::SQLException: 2nd ORDER BY term out of range - should be between 1 and 7)
```

## `pluck`

[`pluck`](https://api.rubyonrails.org/v6.1.4/classes/ActiveRecord/Associations/CollectionProxy.html#method-i-pluck) pulls out specified columns from a query, instead of loading whole records.

In previous versions of Rails, `pluck` (somewhat surprisingly!) accepted arbitrary SQL strings if they were passed in as an array.

Like `order`/`reorder`, Rails 6.0 started warning about this:

```ruby
 > User.pluck(["1"])
DEPRECATION WARNING: Dangerous query method (method whose arguments are used as raw SQL) called with non-attribute argument(s): ["1"]. Non-attribute arguments will be disallowed in Rails 6.1. This method should not be called with user-provided values, such as request parameters or model attributes. Known-safe values can be passed by wrapping them in Arel.sql().
```

In Rails 6.1, `pluck` now only accepts attribute names!

## `reselect`

Rails 6 introduced [`reselect`](https://api.rubyonrails.org/v6.1.4/classes/ActiveRecord/QueryMethods.html#method-i-reselect), which allows one to completely replace the `SELECT` clause of a query. Like `select`, it accepts any SQL string. Since `SELECT` is at the very beginning of the SQL query, it makes it a great target for SQL injection.

```ruby
params[:column] = "* FROM orders -- "
User.select(:name).reselect(params[:column])
```

Note this selects _all_ columns _from a different table_:
```sql
SELECT * FROM orders -- FROM "users"
```

## `rewhere`

[`rewhere`](https://api.rubyonrails.org/v6.1.4/classes/ActiveRecord/QueryMethods.html#method-i-rewhere) is analogous to `reselect` but it replaces the `WHERE` clause.

Like `where`, it is very easy to open up `rewhere` to SQL injection.

```ruby
params[:age] = "1=1) OR 1=1--"
User.where(name: "Bob").rewhere("age > #{params[:age]}")
```

Resulting query:
```
SELECT "users".* FROM "users" WHERE "users"."name" = ? AND (age > 1=1) OR 1=1--)
```

## Wrapping Up

Any other new methods that allow SQL injection? Let me know!

Want to find out more?

* [rails-sqli.org](https://rails-sqli.org) for the complete list.
* [Brakeman](https://brakemanscanner.org/) to help find vulnerable queries in your code.
* [OWASP SQL Injection Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html) to learn more about preventing SQL injection.
