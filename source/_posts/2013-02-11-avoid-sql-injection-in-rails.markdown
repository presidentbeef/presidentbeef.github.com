---
layout: post
title: "Avoiding SQL Injection in Rails"
date: 2013-02-08 09:45
comments: true
categories: 
---

SQL injection (SQLi) is any situation in which a user can manipulate a database query in an unintended manner. Consequences of SQL injection vulnerabilites range from data leaks, to authentication bypass, to root access on a database server. In short, it is a very big deal.

Most Rails applications interact with a database through ActiveRecord, the default and convenient Object Relational Mapping (ORM) layer which comes with Rails.
Generally, use of ORMs is safer than not. They can provide abstraction and safety and allow developers to avoid manually building SQL queries. They can embody best practices and prevent careless handling of external input.

Instead of unsafe code like

    query = "SELECT * FROM users WHERE name = '#{name}' AND password = '#{password'} LIMIT 1"
    results = DB.execute(query)

You can have safer, simpler code like

    User.where(:name => name, :password => :password).first

My impression is many people assume the Rails framework will protect them as long as they avoid the "obviously dangerous" methods, like `find_by_sql`.

Unfortunately, ActiveRecord is unsafe more often than it is safe. It does provide parameterization of queries (the API documentation for which can be [found here](http://api.rubyonrails.org/classes/ActiveRecord/Base.html)) for some methods, there are many methods for which it does not. While these methods are not intended to be used with user input, the truth is that has never stopped anyone.

To make it clear how dangerous it can be to use ActiveRecord, consider [ActiveRecord::FinderMethods#exists?](http://api.rubyonrails.org/classes/ActiveRecord/FinderMethods.html#method-i-exists-3F) which queries the database and returns `true` if a matching record exists. The argument can be a primary key (either integer or string, if a string it will be sanitized), an array consisting of a template string and values to safely interpolate, or a hash of column-value pairs (which will be sanitized).

Here is an example of using `exists?` to determine if a given user exists:

    User.exists? params[:user_id]

This looks harmless, since `params[:user_id]` is a string, and strings will be sanitized. In fact, the documentation clearly points out not to pass in conditions as strings, because they will be escaped.

However, there is no gaurantee `params[:user_id]` is a string. An attacker could send a request with `?user_id[]=some_attack_string`, which Rails will turn into an array `["some_attack_string"]`. Now the argument is an array, the first element of which is not escaped.

To avoid this problem, the user input should be converted to the expected type:

    User.exists? params[:user_id].to_i

Or use a hash:

    User.exists? :id => params[:user_id]

This should be the approach for all uses of user input. Do not assume *anything* about values from external sources or what safety mechanisms a method might have.

While working on [Brakeman](http://brakemanscanner.org/), I thought it would be useful to put together a list of all the unsafe ways one can use ActiveRecord.

To make it easier on myself, I built the list into a Rails application so I could easily test, verify, and record any findings. The source is [available here](https://github.com/presidentbeef/inject-some-sql) for those who would like try out the examples. The application is a single page of all the queries and example injections. From there one can submit queries and see the results:

![Query Example](/images/blog/inject-some-sql.png)

The resulting information is available at [rails-sqli.org](http://rails-sqli.org), including examples of how SQL injection can occur and the resulting queries. This is basically a big list of what *not* to do when using ActiveRecord. Again, please feel free to [contribute](https://github.com/presidentbeef/inject-some-sql) so that the list can be as authoritative as possible and help everyone avoid SQL injection in Rails.
