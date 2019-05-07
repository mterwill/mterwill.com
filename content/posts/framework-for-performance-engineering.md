---
title: "What I've learned about performance engineering [teams]"
date: 2019-04-30
draft: false

---

# What I've learned about performance engineering [teams]

Over the past year or so, I've had the opportunity to work on performance
engineering at Mailchimp. Performance is an important feature for any web
application. It's been [shown][why-perf-matters] to impact both customer
retention and conversion.

I find performance engineering to be a fascinatingly vast problem space, both
from a human and a technical perspective. Mailchimp is (for the most part) a
monolithic application, incepted a decade ago. The company and the features we
offer have grown a lot since then: today we have several hundred engineers
across dozens of teams contributing daily to the same shared codebase. Ensuring
our application's performance across this large surface area continues to prove
challenging. My work has mainly centered around improving the backend
performance of our monolith. This post outlines a few methodologies I've found
effective for doing so; hopefully some of the ideas are transferable to other
environments.

---

If you're starting a performance engineering team from scratch like we were, one
of the first (and most daunting) questions is simply where to start.  Maybe you
have some direction from leadership, some data from research and support, and
telemetry around your application itself. It's easy to get caught up in this
stage – don't.

**You will never establish a total ordering of performance issues in your
application.** The best thing you can do for your customers and company is to
establish a framework that enables you to make locally optimal decisions based
on the information available to you and the constraints at a given time. I
recommend starting with something like this:

## A framework for pragmatic performance engineering

When I first wrote this, I omitted an important step zero: observability. This
framework presumes some level of telemetry in your stack. The ability to measure
the impact of an issue and the impact of your fix is critical; you need some
means to extract and aggregate arbitrary timeseries data from your application.
I also recommend that you invest [some time][1205] trivializing the process to
profile (or trace) in production. Make sure you can find _what_ is slow at a
high level, for _whom_, _how_ slow it is, _when_ it's slow (always? 50% of the
time?), and _why_. With these tools in your tool box, let's get started!

---

The first step is to examine the data available to you, both qualitative and
quantitative. Everything from load balancer logs, to Google Analytics, to
support data, and bug reports provide valuable context.

Next, collate the raw data into a story for each performance issue. Some
questions to consider:

* How does a given issue impact the customer?
* How many customers does it impact?
* How much do the impacted customers pay?
* Can you capture a video of the customer experience in this degraded
  performance state?
* Does the issue block their workflow?
* Does the issue impact anyone else (e.g. your customer's customer)?
* Does the issue cause toil for support?
* Has the issue caused customers to leave your product?

Start building this story in the ticketing system of your choice.

Taking the time to frame each performance opportunity into a cohesive and
compelling narrative is worth it. These tickets aren't for just you, they're for
stakeholders across the organization. When it's time to give an update to
leadership, or talk about the impact you've made at an all-hands, you'll be
thankful that the details of "why did we start looking at this darn thing in the
first place?" are captured for posterity. When a support agent hops into channel
with an issue, it's nice to say "yes, this is on our radar, _x_ other customers
are experiencing the same pain, and you can follow along here."

---

At this point, you have an unprioritized backlog of stories. Super high-level
stuff with limited information as to what technical factors contribute to the
performance issue. Perhaps some tickets already jump out: high-latency,
high-traffic pages, slow, mission-critical jobs, or web requests consistently
timing out. Great. In order to decide what to work on first, next, you need to
pop open the hood: it's time to break out the profiler and leverage your
telemetry.

Your goal at this stage is not to solve the issue, but to gauge what effort may
be required in a fix. [Timeboxing][timebox] is your best friend here. For each
issue, spend an hour or two digging. You may need to take a staged approach:
first, attempt to answer "what makes _x_ slow in this particular instance?"
Reproducing a performance issue can sometimes be tricky. Perhaps there's caching
involved. Perhaps the operation that's slow has side effects so you cannot
re-run it at will. Regardless, before you fix a problem, you need to pinpoint
it.

Once you've reproduced a performance issue once, you're not done. Don't run and
make the fix yet! You cannot extrapolate based on a single observation of
slowness. I.e., if one database query made one web request slow for one user,
that does **not** imply this same database query is responsible for the poor
performance of an endpoint in aggregate.

What you have now is a hypothesis. "I believe this database query causes the
poor performance of this web endpoint for these _x_ users." We should now
attempt to [dis]prove the hypothesis. Add instrumentation around the database
query. Switch gears for a bit to allow your metrics to stream in. If this new
data confirms your hypothesis, congratulations — you've taken a vague report of
slowness and broken it down into an isolated component whose impact you can now
quantify. You're in a very, very good place. If the new data disproves your
hypothesis, don't despair! You've validated why this step is important. I'd
rather invest a couple hours of investigation up front than a week of
development work optimizing something that will have a negligible impact.

Continue to fill out the story in your ticket with new information you gather at
this stage. Depending on the scope of the problem, you may also find it helpful
to break out isolated fixes into their own tickets.

---

Let's step back for a minute. First, we transformed raw performance data into a
story. Next, we spent a couple hours investigating each story. We've now
navigated our complex system and isolated a particular component that causes a
measurable slowdown in a known context. Y'all, that was the hard part. Now
comes the fun stuff.

How you sort your issues is up to you; at this stage you have all the
information necessary to inform prioritization. You understand the who, what,
when, where, why, and how for each issue. Go forth and write code! Don't forget
to tie your story in a bow by measuring and documenting the impact of your
improvements along the way.

---

This is a fluid cycle. Over time, you'll fix issues and reports of new ones will
surface. As you establish a process that works for your organization, and
improve tooling, etc., you can begin to share the wonderful world of performance
engineering across your company. I find this, establishing a culture of
performance, to be an ongoing challenge. I'm excited by the strides we've
already made and for what lies ahead. If you found this interesting and have
thoughts or questions, I'd [love to chat][me]. Thanks!

## Some additional considerations
* A dedicated performance engineering team cannot be _responsible_ for the
  performance of your application, but they should be _accountable_ and
  _consulted_. (See [RACI][]) Performance is the responsibility of everyone
  across engineering.
* While the practice of documentation may seem tedious at first, it's a muscle
  you build over time. In performance engineering, you will need to cross
  organizational boundaries. A well-written story allows you to share your work
  effectively and efficiently, say, if you need to convince another team to
  prioritize a performance improvement to their code.
* If you have a performance engineering team, I recommend setting up office
  hours: dedicated time where anyone from across the organization can sit down
  with you. This sets aside time for you to triage one-off reports into stories
  without interrupting your normal prioritization flow. Office hours allow you
  to _look_ at new issues without obligating yourself to _fix_ them.

[why-perf-matters]: https://developers.google.com/web/fundamentals/performance/why-performance-matters/
[1205]: https://xkcd.com/1205/
[timebox]: https://en.wikipedia.org/wiki/Timeboxing
[RACI]: https://en.wikipedia.org/wiki/Responsibility_assignment_matrix#Role_distinction
[me]: mailto:matt@terwilligers.com
