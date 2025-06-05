---
layout: default
title: May 18, 2025 Silent Breaking
nav_order: 3
---

# Silent Breaking
### Scheduled Build
Recently, I encountered an issue that made me rethink when you can actually trust the integrity of
your own codebase. The short version: only if you explicitly check it.

I hadn’t touched my *Dockerized-Microservices-Demo* project for a few months. Last time I built
it, everything worked fine. I have GitHub Actions configured to run builds after every push, so if I
push some changes, the build verifies that the project still builds correctly. So the logic is: if
there are no changes, everything should still be OK.
Turns out not really.

By complete accident, I discovered that during this inactivity part of my tests had started failing.
The reason was very simple - hardcoded test dates, verified directly against the absolute source of
truth - Java's `LocalDate.now()`. They were supposed to represent future dates — but time had
passed, and now they were representing the past. That meant certain validations failed, and
consequently, the tests.  
Since I hadn’t built the project for a while, I wasn’t aware.  
*(But I was still actively showcasing the project at the same time. What a shame.)*

That made me realize: *just because a project hasn’t changed, doesn’t mean it hasn’t broken*. Time
passes, dates expire, assumptions age. Once-working code can quietly slip into disrepair while
you’re off doing other things.

> *The only way to know a project still works is to actually check it - and regular builds can help
> with that.*

If you already have an automated workflow, adding a **scheduled build** (e.g. once a week) is easy. For
fast-moving projects it might not matter much - issues surface quickly anyway. But for less active
ones a scheduled build will give a confidence based on recent results, not just old assumptions.
```yaml
# GitHub Actions example: schedule weekly build
on:
  schedule:
    - cron: '0 2 * * 1' # Monday 02:00 UTC
```
> See GitHub Actions [official docs](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#schedule) for the full schedule documentation.

Another part of the story is whether the code should contain tests, that can unexpectedly expire.
This issue definitely made me rethink how my dates are handled in the code - and how much control I
might actually want over them. But that's a separate post.
