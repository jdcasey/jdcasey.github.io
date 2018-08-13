---
title: Through a Dashboard Darkly
categories: indy o11y software architecture
--- 

>The funny thing about reasoning with incomplete or irrelevant information is that it's hard to be objective; it's hard to tell from motivated reasoning and outright justification.


I'm a huge Philip K. Dick fan, so I hope you'll excuse the reference. I think it's relevant here, since we've been learning a lot more about Indy over the past few months, using a new metrics dashboard we deployed in May. In particular, we're seeing how things we thought we understood in the absence of good data were just a fiction we were writing for ourselves; and how, even with a metrics dashboard, we still struggle to understand this software we've written.

## Introduction

Recently, we released a new version of our entire build system, including a new version of Indy (our Maven repository manager, for which my team is responsible). Through rounds of testing, we ignored or explained away a growing feeling that something was wrong, partly because we couldn't find the issue. In the end, we wound up diagnosing and fixing the problem after shipping it to production.

This is a story of a flawed development process on top of a flawed infrastructure, without the necessary data to drive decision-making. It's also a story of waking up to these problems and charting a way out.

## Reasoning in the Dark

Before May 2018, when we ran into performance problems in Indy (and there were plenty in this system that serves 1000's of requests for each build) we had to use the information we could glean from short glimpses at logs that would sometimes roll in minutes; close reading of complex code; and an embarrassing amount of theorizing and intuition. We simply didn't have much insight into what was happening, which part of the system was actually our bottleneck. We theorized that we had a slow NFS mount...well, not slow, but maybe 5ms per file attribute read. We theorized that our content service process didn't fail fast enough, traversing too much code before returning a 404 result. We wrote unit tests and functional tests to verify these theories, but as with so many production problems, firm conclusions remained elusive outside of the production environment.

We wasted a lot of time and effort on this state of affairs. Not just production debugging time. Not just time poring over the content-service code path and trying to make it return negative results faster. We engaged in a lot of discussions, design work, and even some implementation that had negligible effect on performance...some even hurt Indy's performance!

All of this happened because we could not tell what problem needed to be fixed. As it turns out, pure reasoning cannot solve the kind of problems you see in the production environment of a complex application. These problems are almost always more difficult, since they have survived all of the testing you could throw at them.

We were fumbling in the dark.

## The Night Light - Our First Look at the Mess

When we turned on metrics visualization out of desperation during a weekend-long production outage in May, we immediately saw some things that were badly broken. They weren't even code; they were configurations that were completely un-optimized. We weren't using more than 1/16th of our available memory, and the server's back-end threadpools were a small fraction the size of the ones trying to serve users...which of course meant these user-facing threads were literally starving for content. These were our first glimpses at a pile of issues we could address to improve performance. The changes we made as a result transformed performance on Indy, and gave us a new vector to continue improving.

In the following weeks, we again saw production blocker issues crop up related to memory consumption. This time the problem wasn't configuration; it was an implementation detail in the code that amounted to a memory leak under the wrong conditions. Our configuration changes had made a big difference, but we were definitely not out of the woods. We began making plans to cap some of the worst memory users, and to do more metrics reporting for our internal caches to help us understand their memory usage in a live system. These changes were rushed into the 1.3 release effort for our build system, which had already entered acceptance testing before the big outage happened.

This felt a lot like turning on a night light in a dark house; we could see enough to avoid the furniture, but we still couldn't quite make out the Legos on the floor.

## The Flashlight - Seeing Clearly, One Circle at a Time

As testing for our new build system release progressed, we started to see the edges of a new performance problem. We had a dashboard that showed us statistics for Indy's internal caches, memory usage, and response times on some key components. We also had sluggish response times and what looked like an incredibly long warm-up time as we populated cold caches. We could see what the response times were like on certain remote repositories that we rely on constantly. We had an unprecedented view into Indy, but had a hard time seeing why it was responding so slowly. After following the limited evidence and doing a fair bit of experimentation, we eventually refactored Indy to make a net speed improvement over the old 1.2 version.

Part of the problem here was that we'd never had this kind of a look at the system, which meant we didn't know what "normal" looked like. The last time we did a release, we were still using a completely un-tuned production environment, which meant our expectations for the staging environment were still lagging the performance gains we made in production.

These facts led to a lot of reasoning about cache warm-up times, wrong-headed code to try to address that problem, and pressure to refactor our data architecture (which in point of fact, does need to happen...but this kind of change is not suitable for a hotfix). It also led us to point the finger at the stage environment itself, with its 6 CPUs (vs. 40 in production) and 16 GB RAM (vs 64 in production). We talked about how hard it was to try to transfer operations (and their attendant sense of "normal") from production into stage and use them to test performance. In the end, everyone had a vague sense that something wasn't right, but we couldn't put a finger on it. We convinced ourselves that the jump to such a large environment would result in non-linear performance gains. So, we jumped.

The funny thing about reasoning with incomplete or irrelevant information is that it's hard to be objective; it's hard to tell from motivated reasoning and outright justification.

When we deployed to production, we immediately saw our fears realized. We were (slowly) accumulating several million entries in our content index, and even more in our not-found cache (a negative result cache that prevents expensive upstream operations which are likely to fail). This was the payoff of the new metrics we deployed. We could see that things were wrong, but we had a hard time understanding why, or what to fix. One member of the build-system team had done a great analysis of access times in the run-up to release, studying how much our data architecture amplified latencies in different parts of the application. His work pointed out that our NFC was working way too hard and wasting our time. As a result, he cooked up a patch that exempted hosted repositories from the NFC (previously, while reasoning in the dark, we added NFC support for hosted repositories on the assumption that our NFS mount was slow). This patch put us back on par with performance before the 1.3 release.

However, the content index was still showing an extremely low hit rate, and contained tons of entries (consuming a LOT of memory). The effectiveness of the content index is roughly its size multiplied by its hit rate (as a percentage of retrieval attempts), so this was really telling us that the content index was not paying off. This clear signal contained echoes of intuition we'd had over the past couple years, where it seemed like the index should have yielded more of a performance gain. Finally, in a joint status meeting, we dug into the code with all of us trying to understand what was wrong.

It's fairly obscure, but because of a single line problem, we weren't using the content index appropriately at the repository group level. This meant that we would try to use it at the individual repository level...but owing to the ordered-list nature of repositories in groups, unless the first repository in the group membership contained the content that index wasn't doing much for us. Each cache miss would result in a filesystem verification or worse, a HTTP request to retrieve content from that repository. If we didn't do this, we risked serving up content that should have been obscured if the index was intact. We set group-level index entries to avoid iterating these repositories, and the group level is where the index should have paid large dividends. The problem here was hard to see, and hard to really understand...but very simple to fix.

While all of this was happening, it also became clear that we were keeping dozens of index entries for content that could be indexed together by a single directory-level index entry (since Maven content is mostly grouped by version directories). Another member of the Indy team took care of this problem on his own initiative, and submitted a patch late in the week of the 1.3 release.

After applying these three fixes and redeploying to the staging environment, we began seeing performance (with a cold cache, mind you) that left our production environment in the dust, sometimes sending content back to the user at 4 MB/s where previously it never exceeded 2 KB/s.  Not only that, but it has yet to exceed 3GB of RAM usage.

This result was good enough to trigger some disbelief in our build-system team, so we released Indy 1.3.1 with these fixes.

Since then, we've analyzed response times from Koji, which we started reporting metrics on in Indy 1.3. Koji is another build system from which we integrate repository content via an out-of-band mechanism. While response times were often in the hundreds of milliseconds, again we had no frame of reference of what was "normal" (Koji response stats were new in Indy 1.3). However, I noticed two methods that took much longer than expected: login, and logout. Each call we made into Koji at that time was authenticated, which meant that login/logout latency was incurred for each operation we executed. Setting the Koji session object to null in the code cut response times to Koji dramatically, driving them under 100ms. This doesn't sound like much, but one of the build system services submits requests to Indy that translate into 2000-3000 Koji requests each. Koji response times were amplified accordingly, sometimes leading to response times from Indy that ran into the minutes. Making matters worse, each Koji call happened in serial. These calls didn't depend on each other, so I threaded them off. This helped, though no threadpool can realistically handle 3000 simultaneous operations without iterating.

The upshot of all this was an Indy 1.3.1 release has already dramatically improved performance, which we pushed into production last week; and the ground work for a 1.3.1.1 release which will execute even faster.

## An Ugly Win

In the end, we had enough information in our metrics dashboard to see that something was wrong, even if we couldn't see hard evidence of response times in the not-found cache, or how far down into our content layers we had to drill before we hit content. In the end, we solved the performance problems in Indy 1.3, and even made some huge gains over Indy 1.2. In that sense, this was a win...but not an unambiguous win.

It took us almost two weeks to deploy the performance improvements gleaned from staring at our limited data. For two weeks, almost all of the resource time on the Indy team was consumed with finding these answers. This is not a story of effective production support.

While this wasn't a perfect outcome, and we don't have a highly efficient production support system in place, we do have enough information to start building that support system. We know that we need both timing and capacity information for all operations, with the ones touching the core content-service code path getting the highest priority. We know that we need to report a LOT more information to our time-series database, so we can have the data on hand to diagnose problems when they arise. If we had that this time around, we could have fixed this problem during acceptance testing (when we started gathering metrics), not in production.

Once we instrument all the operations, we will probably face difficulties in delivering it in a low-latency way without slowing down Indy or causing performance problems in the time-series database itself. And even when we've solved all of this, we will still struggle to put the right information on the screen. For this, I've been looking at systems like [Honeycomb](https://honeycomb.io/) and how they handle what they term "high cardinality" event data...and how they almost shun the notion of dashboards, since there's no way you can watch all of the metrics.

It's not entirely clear how we'll tackle all of these problems, and I'm sure we won't exactly stick the landing on the next release, but I think two things are clear from this exercise:

1. Metrics reporting and operational transparency (observability) is critical to running an application in production; **these are not mere luxuries**
2. We have a path forward to achieve sustainable production support in the application layer, even with our small team

It sort of makes me wonder though: how many other applications do we have that could be much, much faster if they implemented this kind of insight?
