---
layout: post
title: "Testing Brakeman Against 253 Rails Apps"
date: 2013-11-01 07:54
comments: true
categories: 
---

Here is some information about how [Brakeman](http://brakemanscanner.org/) is tested!

### Basic Testing and Continuous Integration

Brakeman does have a few unit tests...pitifully few. In fact, Brakeman had no tests at all until version [0.5.2](https://github.com/presidentbeef/brakeman/blob/master/CHANGES#L490), nearly a year after Brakeman's initial public release. Unit testing Brakeman remains difficult, since much of the code relies on data built up from scanning an entire Rails application.

As such, the majority of tests in Brakeman rely on scanning [sample applications](https://github.com/presidentbeef/brakeman/tree/master/test/apps) and checking the resulting reports for an expected set of warnings. There are tests for the presence and absence of specific warnings, as well as checking for the specific number of warnings and an absence of reported errors. Since writing tests is pretty tedious, there is [a script](https://github.com/presidentbeef/brakeman/blob/master/test/to_test.rb) which generates the Ruby code to asserts the presence of reported warnings. This script takes the same arguments as Brakeman, so it's simple to generate a set of tests for a specific scenario.

```ruby
def test_information_disclosure_local_request_config
  assert_warning :type => :warning,
    :warning_code => 61,
    :fingerprint => "081f5d87a244b41d3cf1d5994cb792d2cec639cd70e4e306ffe1eb8abf0f32f7",
    :warning_type => "Information Disclosure",
    :message => /^Detailed\ exceptions\ are\ enabled\ in\ produ/,
    :confidence => 0,
    :relative_path => "config/environments/production.rb"
end
```

The tests run on [Travis CI](https://travis-ci.org/presidentbeef/brakeman) which is integrated with GitHub. This is especially helpful for testing compatibility with Ruby 1.8.7, which many Rails applications still run on and Brakeman will probably continue supporting for a long time.

### Regression Testing with a Wide Net

Unfortunately, the sample applications Brakeman uses for tests are quite limited, not real, and generally just test very specific warnings or previous bugs. To gain higher confidence that Brakeman is not too broken, Brakeman is run against a set of 253 open source Rails applications I have managed to scrape together. (If you have an open source application to add to this test set, please let me know!)

The scans are run on my personal machine - six jobs in parallel, which takes about nine minutes total. After puttering around with a few different approaches, I ended up simply using the [Queue](http://rdoc.info/stdlib/thread/Queue) class from Ruby's standard library as the job queue. In a Frankenstein combination, a shell script starts up a JRuby process, which builds the Brakeman gem and then runs six threads for scan jobs. Each job launches Brakeman as an external process running under MRI 1.9.3 and, if successful, produces a JSON report. The JSON report is then augmented with some information about the Brakeman commit and the app that was scanned.

When all the apps have been scanned, the JSON reports are tarred up and sent to a server. I use [DigitalOcean](https://www.digitalocean.com/?refcode=35d9e7aec070) (referral link!) because I needed an Ubuntu setup and their API lets me use some handy [scripts](https://github.com/presidentbeef/my_ocean) to spin the server up and down whenever I need it (and only pay for when it's up).

On the server, the reports are unpacked and imported into a [RethinkDB](http://rethinkdb.com/) database. Since RethinkDB stores JSON documents, it's simple to dump the JSON reports from Brakeman in there. I just have two tables: one just contains commit SHAs and their timestamps, and the other contains the actual reports. I have secondary indexes on the reports to efficiently look them up by the name of the Rails app or the Brakeman SHA. 

A small [Sinatra](http://www.sinatrarb.com/) app serves up some basic graphs and allows two commits to be compared:

![Brakeman Graphs](/images/blog/brakeman-graphs.png "Ugly, I know")

This "system" is not open source at the moment, but probably will be in the future when I've removed hard-coded stuff.

Anyhow, since I have all these reports, I can share some data...but just be forewarned you can't really draw any conclusions from it!

### Numbers!

This is the RethinkDB query for warnings per category, in JavaScript since I ran it in the web UI:
```javascript
r.db("brakeman").
  table("reports").
  getAll("25a41dfcd9171695e731533c50de573c71c63deb", {index: "brakeman_sha"}).
  concatMap(function(rep) { return rep("brakeman_report")("warnings") }).
  groupBy("warning_type", r.count).
  orderBy(r.desc("reduction"))
```

<table>
  <tr>
    <th>
      <strong>Warning Category</strong>
    </th>
    <th>
      <strong>Count</strong>
    </th>
  </tr>
  <tr>
    <td>
      Cross Site Scripting
    </td>
    <td>
      6669
    </td>
  </tr>
  <tr>
    <td>
      Mass Assignment
    </td>
    <td>
      3385
    </td>
  </tr>
  <tr>
    <td>
      SQL Injection
    </td>
    <td>
      1353
    </td>
  </tr>
  <tr>
    <td>
      Remote Code Execution
    </td>
    <td>
      458
    </td>
  </tr>
  <tr>
    <td>
      Denial of Service
    </td>
    <td>
      440
    </td>
  </tr>
  <tr>
    <td>
      Redirect
    </td>
    <td>
      232
    </td>
  </tr>
  <tr>
    <td>
      Format Validation
    </td>
    <td>
      230
    </td>
  </tr>
  <tr>
    <td>
      Attribute Restriction
    </td>
    <td>
      205
    </td>
  </tr>
  <tr>
    <td>
      File Access
    </td>
    <td>
      200
    </td>
  </tr>
  <tr>
    <td>
      Session Setting
    </td>
    <td>
      169
    </td>
  </tr>
  <tr>
    <td>
      Dynamic Render Path
    </td>
    <td>
      140
    </td>
  </tr>
  <tr>
    <td>
      Command Injection
    </td>
    <td>
      116
    </td>
  </tr>
  <tr>
    <td>
      Cross-Site Request Forgery
    </td>
    <td>
      96
    </td>
  </tr>
  <tr>
    <td>
      Default Routes
    </td>
    <td>
      67
    </td>
  </tr>
  <tr>
    <td>
      Response Splitting
    </td>
    <td>
      44
    </td>
  </tr>
  <tr>
    <td>
      Dangerous Eval
    </td>
    <td>
      43
    </td>
  </tr>
  <tr>
    <td>
      Dangerous Send
    </td>
    <td>
      33
    </td>
  </tr>
  <tr>
    <td>
      Nested Attributes
    </td>
    <td>
      5
    </td>
  </tr>
  <tr>
    <td>
      Information Disclosure
    </td>
    <td>
      2
    </td>
  </tr>
  <tr>
    <td>
      Authentication
    </td>
    <td>
      2
    </td>
  </tr>
</table>
<br>

Some educated guesses about these numbers:

* Mass assignment numbers are likely high because they include warnings about dangerous attributes that are whitelisted.
* Remote code injection is mostly uses of `constantize` and similar methods.
* Most denial of service warnings are calls to `to_sym` on parameters
* Response splitting is interesting because it is only reported in regards to [CVE-2011-3186](https://groups.google.com/d/msg/rubyonrails-security/b_yTveAph2g/jKe6OuRC47sJ) which was fixed in Rails 2.3.13. 

This last point made me curious about the Rails versions in use by the applications. Keeping in mind these apps are not necessarily up-to-date, they represent at least 37 different versions! Some were reported as unknown versions.

Here are the top ten:
<table>
  <tr>
    <th>
      <strong>Rails Version</strong>
    </th>
    <th>
      <strong>Count</strong>
    </th>
  </tr>
  <tr>
    <td>
      3.2.13
    </td>
    <td>
      26
    </td>
  </tr>
  <tr>
    <td>
      2.3.5
    </td>
    <td>
      19
    </td>
  </tr>
  <tr>
    <td>
      3.0.3
    </td>
    <td>
      18
    </td>
  </tr>
  <tr>
    <td>
      3.2.14
    </td>
    <td>
      14
    </td>
  </tr>
  <tr>
    <td>
      4.0.0
    </td>
    <td>
      11
    </td>
  </tr>
  <tr>
    <td>
      3.2.12
    </td>
    <td>
      9
    </td>
  </tr>
  <tr>
    <td>
      2.3.8
    </td>
    <td>
      8
    </td>
  </tr>
  <tr>
    <td>
      3.2.11
    </td>
    <td>
      8
    </td>
  </tr>
  <tr>
    <td>
      3.0.0
    </td>
    <td>
      7
    </td>
  </tr>
  <tr>
    <td>
      3.1.0
    </td>
    <td>
      6
    </td>
  </tr>
</table>
<br>

With so many applications and nearly 14,000 warnings, there is a lot more information to go through here.

For now this process is used to help test new Brakeman code and avoid regressions. It's stopped quite a few bugs from going out!
