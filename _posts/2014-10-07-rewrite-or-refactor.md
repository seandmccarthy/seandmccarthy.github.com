---
layout: post
title: "Rewrite or Refactor"
date: 2014-10-07
comments: false
categories: 
---

You've inherited a mess of a codebase. Much of it seems impenetrable. Adding features is onerous, and you can't tell what else you might break in the process. It's brittle.

The appealing solution is to rewrite the application from scratch.

Rewrites _are_ appealing. A chance to start clean, use the latest technology or at least up-to-date versions of what you're familiar with.

Rewrites are for code with great tests and test coverage. Rewrites are for well understood business rules. Rewrites are for legacy codebases that are still easy to understand. Rewrites are for an unavoidable platform change. Rewrites are a last resort for when building it and missing some important details is less costly than gradual measured improvements.

The problem with the codebase you have is that there's some arcane business rule you don't know about and can't easily see. There's some forgotten piece of hard earned workaround, crucial to interoperating with another system. You won't know about these until the system(s) it integrates with start failing. Murphy's Law says you'll only find out in production.

The unappealing, but likely right choice, is that you should refactor. Call it a rebuild, renovation, or overhaul if you like.

Start by building some confidence around making changes by adding or refining tests. When the codebase you're refactoring doesn't have a good test suite, then start by testing what you can. From the outside in.

Start with integration tests if you can't do unit tests. As you break down large classes or functions later you can add unit tests to those bits.

Pull out the crudiest bits into their own isolated class or method. Make it testable. Then make it good.

You're going to fatigue, so gamify the process. Take some measure of something, like number of tests, code quality score, anything, and try to improve that score. Celebrate some milestones. Give before and after demonstrations. Make cosmetic changes to freshen the UI if there is one.

In the end the outcome will likely be better, and you can pride yourself on leaving something better than you found it.

