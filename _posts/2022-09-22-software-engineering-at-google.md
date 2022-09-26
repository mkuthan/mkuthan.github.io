---
title: "Software Engineering at Google -- book review"
date: 2022-09-22
tags: [Software Engineering]
categories: [Books]
toc: false
---

Recently I've read ["Software Engineering at Google"](https://www.oreilly.com/library/view/software-engineering-at/9781492082781/) 
curated by [Titus Winters](https://www.linkedin.com/in/tituswinters/), 
[Tom Manshreck](https://www.linkedin.com/in/thomas-manshreck-0111a11/) 
and [Hyrum Wright](https://www.linkedin.com/in/hyrum-wright-0905427/).

![Software Engineering at Google](/assets/images/2022-09-22-software-engineering-at-google/bookcover.jpg)

In short, the book greatly enhanced my software engineering comprehension
and gave me motivation for further experimentation.
The book wasn't easy to digest, many authors, different writing styles,
repetitions across chapters, and over 600 pages in paperback version.
But I don't regret even a minute of reading. 
I rated the book with 5 stars on [Goodreads](https://www.goodreads.com/user/show/6902906-marcin-kuthan).
Until now only 17 of 80 technical books I read have got top scores 😀.

Below you can find a few personal notes taken when I was reading the book.

**[Hyrum's law](https://www.hyrumslaw.com)**

Ubiquitous law, referenced many times in the book. 

> With a sufficient number of users of an API,
> it does not matter what you promise in the contract:
> all observable behaviors of your system will be depended on by somebody.

For example, Linux wraps process identifiers (PID) when they exceed 32768 (`2^15`).
Even if you can set the higher value, it would break because many libraries assume that the maximum is 32768.

**Sustainable software**

> You may choose to not change things, but you need to be capable.

* Think about product lifecycle, how do you keep your code for as long as it needs to keep working.
* Code must be easy to read, easy to understand and easy to maintain.
* Manual tasks should be scalable (linear or better) in terms of human input.
* When did you deploy a service last time?
* Are you ready to apply a security fix in a matter of hours?

**[Monorepo](https://research.google/pubs/pub45424/)**

* Monorepo is a key enabler for many other tools and practices, for example: large scale refactoring.
* It's much easier to extract a repository from the monorepo than to combine separated repositories into the monorepo.  

**Blaze**

* Artifact based, distributed build system.
* How to apply functional programming principles to the build system?
* What's wrong with tasks based build systems?
* Local workspaces, remote cache, pre-commit hooks.

See also the open source version of Blaze - [Bazel](https://bazel.build).

**[One version rule](https://opensource.google/documentation/reference/thirdparty/oneversion)**

* Dependency management is a magnitude harder than source management.
* Adding a dependency to a project isn't a free lunch, providing dependency isn't free as well.
* [Minimum Version Selection](https://research.swtch.com/vgo-mvs), interesting variation of [SemVer](https://semver.org). 

**[Style guides and rules](https://google.github.io/styleguide/)**

> Rules help us to manage complexity and build a maintainable codebase.

* Canonical source of styles for every approved programming language on the organization level.
* Focus on resilience to time and scaling.
* Automate enforcement when possible and fail the build if the code doesn't meet the rules.

See also: [google-java-format](https://github.com/google/google-java-format), [Yapf](https://github.com/google/yapf), 
[ClangFormat](https://clang.llvm.org/docs/ClangFormat.html) and [Gofmt](https://pkg.go.dev/cmd/gofmt).

**[Code review](https://google.github.io/eng-practices/review/)**

> The code review process is the primary developer workflow upon which almost all other processes must hang, 
> from testing to static analysis to CI.

* Many roles: readability engineers, hierarchical code owners, etc.
* Readability: standardised mentorship through code review.
* Trust your engineers, the first "Looks good to me" (LGTM) is the most important one.

**Testing**

> It has enabled us to build larger systems with larger teams, faster than we ever thought possible.

* Beyonce rule - *If you liked it you should put a test on it*.
* [Test size](https://testing.googleblog.com/2010/12/test-sizes.html): small, medium, large. 
  Easier to quantify than unit/integration/e2e (which is test scope).
* [Testing on the Toilets](https://testing.googleblog.com/search/label/TotT) - 
  how to improve software engineering practices in a funny way.
* [DAMP over DRY](https://enterprisecraftsmanship.com/posts/dry-damp-unit-tests/)  
* Test state not interactions.  
* Don't mock, use stubs, fakes or real implementation.
* Learn how to live with [flaky tests](https://testing.googleblog.com/2017/04/where-do-our-flaky-tests-come-from.html) 😂, 
  they're inevitable at some scale.

**Documentation**

> Software engineers need to realize that producing quality documentation is part of their job 
> and saves them time and effort in the long run.

* Documentation as a code (avoid wiki).
* Technical writers don't scale (but are useful if you don't know the readers).
* Use link shortener to create links to the canonical sources.

**Deprecation strategy**

> Scalably maintaining complex software systems over time is more than just building and running software:
> we must also be able to remove systems that are obsolete or otherwise unused.

* Deprecation is hard, but a without deprecation strategy you will be stuck.
* Code is a liability, not an asset.
* Deprecation during design like for a [power plant](https://www.iaea.org/publications/5716/design-and-construction-of-nuclear-power-plants-to-facilitate-decommissioning).
* Advisory deprecation (hope isn't a strategy) vs compulsory deprecation
* Prevent backsliding
 
**[Code search](https://developers.google.com/code-search)**

Global code search with full understanding of the structure and references, greatly simplifies development.

> How others have done something with this API?
> Where a specific piece of information exists in the codebase?
> What a specific part of the codebase is doing?
> Why a certain piece of code was added, or why it behaves in a certain way?

See also a primary building block of the code search: [kythe.io](https://kythe.io)

**Large Scale Changes (LSC)**

> No matter the size of your organization, it’s reasonable to think about how you would make these kinds of sweeping 
> changes across your collection of source code. 
> Whether by choice or by necessity, having this ability will allow greater flexibility as your organization scales 
> while keeping your source code malleable over time.

* Some design decisions might be fixed by LSC
* Relatively small team responsible for common infrastructure 
  is able to migrate all codebase from old systems, compilers, libraries, etc.

See [Sourcegraph](https://about.sourcegraph.com/batch-changes).

**Goal / Signals / Metrics (GSM)**

* Make data driven decisions, but in reality decisions are made based on a mix of data and assumptions.
* Start with goals and signals even if you can't measure them.
* Before measuring, ask if results are actionable.
  If you can't do anything with the results, don't measure at all.

**Engineering manager vs. Tech lead vs. Tech lead manager**

Engineering manager:

> Responsible for the performance, productivity, and happiness of every person on their team including their tech lead
> while still making sure that the needs of the business are met by the product for which they are responsible

Tech lead:

> Responsible for the technical aspects of the product, including technology decisions and choices, architecture, priorities, velocity,
> and general project management (although on larger teams they might have program managers helping out with this).
> Also an individual contributor.

Tech lead manager:

> A single person who can handle both the people and technical needs of their team.
> At Google, it’s customary for larger, well-established teams to have a pair of leaders—one TL and one engineering manager—working together as partners.
> The theory is that it’s really difficult to do both jobs at the same time (well) without completely burning out,
> so it’s better to have two specialists crushing each role with dedicated focus.

**Tips and tricks**

* Delegate but get your hands dirty.
* Always seek to replace yourself, always be leaving, avoid bus factor.
* Own *general problem* not a *particular tool*, well-chosen problem can be evergreen.
* Limiting engineers’ power and flexibility can improve their productivity.
* Losing the ego, trust the team.
* Be a Zen Master, ask questions and lead somebody to the answer.
* Don't ignore low performers.
* Fight with haunted graveyards.

**Tools and libraries**

* [Go](https://go.dev) -- The next programming language I'm going to learn
* [Abseil](https://abseil.io) -- C++ library drawn from the most fundamental pieces of Google’s internal codebase
* [xg2xg](https://github.com/jhuangtw/xg2xg) -- by ex-googlers for ex-googlers, a lookup table of similar tech & services
