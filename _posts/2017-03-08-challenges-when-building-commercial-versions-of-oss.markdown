---
layout: post
title: "Challenges When Building Commercial Versions of Open Source"
date: 2017-03-08 21:53
comments: true
categories: brakeman 
---

It has been roughly three years since the ball began rolling on [Brakeman Pro](https://brakemanpro.com/) (the commercial version of the [Brakeman](http://brakemanscanner.org/) security tool for Ruby on Rails), and it has been a little over a year since Brakeman Pro actually [went on sale](https://brakemanpro.com/blog/announcement/2015/11/18/brakeman-pro-is-available). I have learned a ton in that time (and I am still having lessons beaten into me). There have been a ton of challenges going from an OSS project that was never meant to be a paid product to one people are actually buying. Here are just a couple I have much time thinking about:

## The "Free" Version

Clearly, what makes building a commercial product on top of an OSS project different from just selling some software is the existing OSS project itself.

With an OSS project, there is an opportunity to acquire a large number of people testing the software in many different environments. For a static analysis tool like Brakeman, testing on a wide variety of codebases is incredibly valuable. OSS with easy bug reporting and contributing (e.g. a GitHub repo) is not only very likely to receive bug reports and patches, but also suggestions for features and improvements. Brakeman does not receive a large number of code contributions, but bug reports and suggestions for new rules have driven a large chunk of Brakeman’s development.

Being free and open source also makes it easier to advertise your project. People are more willing to promote free software and you can share it around social media with little fear of backlash. Not to mention it is vastly easier to give  conference talks about open source tools!

Personally, I will forever be grateful to the OSS community. Being the main author of widely-used (within a small niche) software has propelled most of my career, led to me speaking all over the world, and has brought me acquaintances and friends I would not have otherwise. I am very glad Brakeman is open source and I would never want to change that.

However, the existence of a "free" version, especially a successful one, has a serious drawback for a business. In particular, **the "paid" version must now not only justify its utility, but also the *incremental* advantage over the free version.** It has become abundantly clear the biggest competitor to Brakeman Pro is Brakeman OSS!

The number one question I receive regarding Brakeman Pro is "What is the difference between Pro and the open source version?" Among other things, one very simple, easy-to-explain difference is the existence of a [GUI](https://brakemanpro.com/features#dt). However, most people want to know if it will "find more things." This leads to considerable hedging from me because Pro probably will find *different* vulnerabilities while also reducing *some* false positives - the net outcome of which may be more or fewer overall warnings. Explaining why Pro may report different vulnerabilities quickly gets me lost in fine details of how the two tools work - at which point people's eyes tend to glaze over.

Trying to quantify the differences between the OSS and Pro versions is a losing battle for me. Potential customers try to add up all these little details and see if it comes out to enough of a difference to begin paying for software they are used to having for free. But, as a technical person, papering over the differences with hyperbolic qualitative statements can seem dishonest. I have yet to arrive at a good solution for this problem.

## Existing User Base

With a well-established OSS project comes another big advantage: the existing user base. These users already like the project and have found it useful! In a way, they have validated a market exists for the product. In the case of Brakeman, I have also felt a tremendous amount of goodwill from the community (for which, again, I am incredibly thankful).

These users are going to be the *very first* people in line to try the commercial product.\* They will already be familiar with the OSS version - therefore communicating and justifying the *additional* value of the commercial version will be critical. The good news is they already know what the product does and have found it valuable. In some cases (but not very many, I’ve found) they may even purchase the product just to support the OSS version. In most cases, though, people need to justify why they are spending budget on this particular software instead of using the free version.

**If you are like me, you may also find this existing user base to be a source of stress.** Marketing to OSS users often feels scummy, but it also makes no sense not to promote the commercial tool to the people already using the free version! For quite a while I did not want to take advantage of the existing audience at all. I have only made very small steps in that direction, preceded by a lot of thought. The last thing I want to do is alienate the community or burn any of the goodwill Brakeman has.

One way to push customers towards the commercial version is to make the OSS version obviously *worse*. But while it would make selling the commercial version easier, not working on or supporting the OSS version is unthinkable. Even the appearance of doing so could turn a community against you. When your potential customers are mostly developers the support of the developer community has extreme value. Besides the business aspect, I personally would have a hard time dealing with loss of the community when the community has done so much for me.

That leaves making the commercial version *so much better* than the OSS version the additional value is ridiculously obvious and people happily pay for it. Sadly (gladly?), many people have let me know "the free version of Brakeman is really good and already does all I need." Making the Pro version *extra awesome* without hurting the OSS version is an ongoing struggle which I continue to hope will resolve itself over time as we continue to improve Pro.

Like many things, the existing user base for an OSS project has both advantages and disadvantages which need to be considered and kept in mind if one is going to turn the project into a commercial product.

## As a Security Tool...

This probably does not apply to very many projects, but as a security tool Brakeman has additional issues related to those above. With every feature that might be exclusive to Pro, I must consider - **"Am I making the world *less safe* by not adding this feature to the OSS version?"** The answers to this question likely lead me to make terrible business decisions. In the end I can live without Brakeman Pro being a successful business, but consciously compromising my integrity and potentially the security of applications is not something I could personally handle.

As a result, the features that tend to go into Pro but not the OSS version are noisier, slower, or focused on ease of use and not actual vulnerability discovery. I believe more false positives (but potentially more true positives) are acceptable in the Pro version because we make it easy to triage and ignore them. Slower features are also much more acceptable in the Pro version - the OSS version needs to be fast and lean. (Sometimes these features also end up in OSS, just off by default. "Off by default" means they might as well not exist for most users.)

## Conclusions

If you are considering taking an open source project and building a commercial tool on top of it, I hope this little post has given you some (perhaps less obvious?) issues to ponder. For Brakeman users, I hope this explains a little bit of the thinking I have done while trying to balance between OSS and Pro.

Note that this blog post is actually an example of the first two issues above: I had to tell you about the "free" version to talk about the Pro version and at the same time you probably feel like this is a bit of an advertisement for the Pro version!

(I think I have to plug my product here now? [Brakeman Pro is a static analysis security tool for Ruby on Rails applications](https://brakemanpro.com/). [Try it out for free](https://brakemanpro.com/purchase/pricing).)

---


\* _One of the early mistakes I made with Brakeman Pro was not realizing who the first customers would be. I thought the people most willing to *buy* a tool would be security auditors, and so the tool and pricing were targeted at *security professionals*. Unfortunately, the much larger market and initial user base for Brakeman are developers. Brakeman Pro should have made developers our top priority from the beginning just like Brakeman OSS does._

