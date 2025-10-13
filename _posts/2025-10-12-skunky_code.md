---
layout: post
title:  "Skunkworks: Isolating experimental code in production"
date:   2025-10-12 12:00:00 -0700
categories:  
---

Sometimes we need to deal with substandard code in a release.
Using a `skunkworks` quarantine module to isolate it 
is a useful strategy when it is necessary.

*"How will I know the good [code] from the bad?"
In the skunkworks module it will be*


## Why would you allow *bad* code to be merged?

We Should™ only merge code which meets our quality and test coverage standards...

Sadly, this is not always practical or possible. 
There are times when a quick fix or a new feature needs to be merged ASAP.
Even after citing the [Big Ball of Mud](http://www.laputan.org/mud/) pattern
as a dire warning, it may be necessary to merge a big chunk of messy untested
jank and hope for the best.

I don't want to come across as too negative here,
because there are very good reasons to be flexible when it comes to code quality.
If there is an outage, getting back up and running is the top priority.
We do not want to block teammates or other teams by waiting on a perfect implementation.
There are many, many, real-world forces that legitimately drive us to cut corners.

## How we deal with *bad* code

There are many strategies for dealing with this scenario, but in a crunch
where minutes matter, a few simple and ineffective strategies tend to dominate.

- Mark it as TODO, or FIXME
    - It's Fast!
    - It's part of the code, so we see it when working on that part of the codebase.
    - But It's invisible outside of the code, not part of our planning or issue tracking.
    - And Easily ignored and put off till 'later'

- Create a cleanup ticket
    - It's tracked and can be included in planning
    - It can be included in reports on tech-debt
    - But It's invisible in the code.

These two quick and easy strategies work well together:
code comments can reference tickets, tickets support thorough explanations
and discussion about HOW TO "FIXME".


## Quarantine that code!
My preferred solution is to create a new submodule named `skunkworks`
to contain any code which doesn't meet quality and testing standards.

### How To:
1. Create the submodule inside of your project source code.
2. Exclude this module from code-quality requirements like
  - Test Coverage (instead of lowering your coverage requirements)
  - Static type checking
  - cyclic complexity checks
    - Formatting requirements like Ruff and Black should probably still apply.
3. Add the quarantined code to skunkworks
  - Open a ticket to track the tech-debt
  - Document the code to reference the ticket
4. Utilizing quarantined code now requires an import from skunkworks


### Advantages of this approach
#### Use of quarantined code is clearly marked
Every time a developer sees `from sunkworks import ...` they know to be cautious.

When that code breaks, the traceback will clearly show that the exception bubbled up from skunkworks.

#### Avoiding the blame game
No one enjoys, or benefits from, the finger-pointing accusations and defenses 
that can arise when code fails catastrophically. 
Using an explicit quarantine module can help.

When your choose to create a `skunkworks` module,
you're explicitly recognizing the need to take on tech-debt.
All of your stake-holders should be made aware, and know to expect problems.
It says "We need to use some substandard code. We acknowledge and accept the risks"

Everyone involved gets to share the responsibility, not just the author or approver of a specific pull request.
It is an architectural and managerial decision that the whole team makes together.

#### Cleanup!
Putting all the untrusted code in one place makes finding what to clean up easy.
Moving code out of skunkworks can take precedence over all other cleanup and refactoring work.

It's a great place for junior devs to start, tell them to "go clean out skunkworks".
They can `git grep skunkworks` to see where its used.
You're giving them working prototype-quality code, 
which they will need to write tests for,
refactor and clean up until it meets standards, and then merge into code base.

It's easier to prioritize for cleanup.
Instead of arguing that "the {some subsystem} code is a mess, and we need to clean it up."
Try "The code which {big important think people care about} relies on is still in skunkworks
and we need to get it out of there before something breaks"

