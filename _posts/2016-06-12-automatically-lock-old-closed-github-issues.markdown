---
layout: post
title: "Automatically Lock Old Closed GitHub Issues"
date: 2016-06-12 22:27
comments: true
categories: 
---

I am not sure this is a problem everyone has, but I grew tired of people commenting on old, resolved GitHub issues.
Almost every time someone would comment "I have this problem, too" it would actually be a different issue. Then I'd
go through the routine of asking them to open a new issue with details about their specific problem.
Sometimes they would, and sometimes they'd never come back.

Fortunately, right around the time I decided I should do something about this annoyance, [GitHub released an API](https://developer.github.com/changes/2016-02-11-issue-locking-api/)
to lock issues. ([Locking issues or pull requests](https://github.com/blog/1847-locking-conversations) prevents any new comments except from repo collaborators.)

So I put together a little gem called [github-auto-locker](https://github.com/presidentbeef/github-auto-locker) to fetch and lock old, closed issues.

To install it (requires Ruby):

    gem install github-auto-locker

Then run:

    github-auto-locker USER REPO TOKEN [age in days]

For example, I run this to lock resolved issues over 60 days old:

    github-auto-locker presidentbeef brakeman N0TM1R34L70K3N 60

The default is 120 days.

I've been running it periodically myself since February without any complaints.
Perhaps it will be useful to you!
