---
layout: post
title: "The three questions I now ask before shipping anything."
date: 2026-08-15 18:00:00 +0300
description: >
  A quick framework for deciding whether a change is reliable, scalable,
  and maintainable before it is shipped.
image:
  path: /assets/img/posts/blog3.jpg
  alt: Shipping software, reliability, and maintainability
categories:
  - Backend
  - Architecture
  - Databases
tags:
  - reliability
  - scalability
  - maintainability
  - ddia
  - engineering
  - systems
  - software
  - architecture
toc: true
comments: true
math: false
mermaid: false
---

A few months ago I would have told you my job was to ship features. Write the code, pass the tests, open the PR, move on. If it worked on my machine and it worked in staging, I called it done.

Then I read the first chapter of *Designing Data-Intensive Applications*, and it ruined that definition of "done" for me.

Martin Kleppmann opens the book with a deceptively simple idea: most applications are not slow because of some exotic algorithm problem. They are slow, or broken, or unmaintainable, because of how data moves through them. And underneath almost every design decision he walks through, there are really only three questions worth asking. Not "does it work," but:

Is it reliable? Will it scale? Can we maintain it?

I've started asking myself these three, in that order, before I open a pull request. Here's what each one actually means, because I used to think I knew, and I didn't.

<!--more-->

## Is it reliable?

My first instinct used to be to treat reliability as a binary. The code either has a bug or it doesn't. Kleppmann's framing is more useful: a fault is one component deviating from spec, a failure is the system as a whole stopping. Reliable systems are not systems without faults. They are systems that keep working in spite of faults.

That reframing matters because it changes what you build for. You stop asking "can this fail" because everything can fail, and start asking "when this fails, what happens next." Does a bad row silently corrupt downstream data, or does it get rejected loudly? Does a slow dependency take the whole request down with it, or does it degrade gracefully? Does a human fat-fingering a config value get caught by a sanity check, or does it go straight to production?

That last one stuck with me the most. The book points out that human error is one of the biggest sources of outages, more than hardware failure, and the fix is not "be more careful." It is designing systems where the easy path is also the safe path. So now, before I ship, I ask: if I'm wrong about something here, how far does the damage spread before someone notices?

## Will it scale?

I used to think scalability meant "handles more traffic." Chapter 1 talks me out of that lazy definition too. Scalability is not a single number you either have or don't. It is a question you can only answer relative to a specific kind of load growth. Twitter's famous fan-out problem is the example that made this click for me: reads are cheap when celebrities do not tweet much, and the moment you flip from "fan out on write" to "fan out on read," you are not scaling the same system. You are solving a different problem entirely.

So "will it scale" is not really the question. The real question is: what is the load parameter that is going to grow here, and what happens to this design when it does? Is it request volume, is it the number of items in someone's timeline, is it the size of a single record? And just as importantly, am I measuring the right thing when I check? Average latency lies. A p50 that looks great can hide a p99 that is making your slowest 1% of users miserable, and depending on what you have built, that 1% might be your most valuable customers, the ones with the most data and the most complex accounts.

I now genuinely cannot look at a latency dashboard with just an average on it and trust it.

## Can we maintain it?

This is the one I used to skip entirely, because "maintainability" felt like a nice-to-have compared to "does it work." Kleppmann splits it into three things that actually matter day to day: operability, simplicity, and evolvability.

Operability is boring on purpose. Good logging, good monitoring, predictable behavior, so the person debugging this at 2am, possibly future me, is not reverse-engineering my intent from scratch. Simplicity is not about fewer lines of code. It is about reducing accidental complexity, the stuff that exists because of how the system was built rather than because of what the problem actually requires. And evolvability is the one that hit hardest: the requirements I am building for today are not the requirements this system will have in a year, and the real cost of a design is not how fast I can ship it now. It is how painful it is to change later.

So the last question I ask is not "is this good code," but "if someone who is not me has to change this in eight months, having forgotten all the context I currently have in my head, is this going to be a reasonable afternoon or a miserable week?"

## Three questions, not one

None of these questions have a clean yes or no answer, and that is kind of the point. Reliability, scalability, and maintainability all trade off against each other and against shipping speed, and pretending otherwise is how you end up with a system that is fast in benchmarks and unbearable to operate, or bulletproof against failure and impossible to change.

I'm four chapters into part one of this book, and I've already caught myself mid-PR going, "wait, what's actually going to break here, and who's going to be the one who has to fix it." That is not a bad trade for a few hundred pages of reading.

Two chapters in, storage engines and data models next. I have a feeling my mental model of "just use Postgres" is about to get complicated too.
