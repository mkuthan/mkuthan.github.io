---
title: "Software Engineering at Google - book review"
date: 2022-12-31
tags: [Software Engineering]
categories: [Books]
---

Recently I've read ["Software Engineering at Google"](https://www.oreilly.com/library/view/software-engineering-at/9781492082781/) 
written by Titus Winters, Tom Manshreck, Hyrum Wright and others. 

![Software Engineering at Google](/assets/images/software-engineering-at-google.jpg)

In short, the book greatly enhanced my software engineering comprehension
and gave me a motivation for further experimentation.
The book was not easy to digest, different writing styles,
many repetitions across chapters, and over 600 pages in paper back version.
But I don't regret even a minute of reading. I rated the book with 5 stars on [Goodreads](https://www.goodreads.com/user/show/6902906-marcin-kuthan).
Until now only 17 of 80 technical books I read, have got top score 😀

Below you can find a few personal notes about the book.

**[Monorepo](https://research.google/pubs/pub45424/)**

I finally understood the advantages of monorepo and ... challenges of distributed build.
Monorepo is a key enabler for many other tools and practices.

**Blaze**

Artifact based, distributed build system.
How to apply functional programming principles to the build system?
What is wrong with tasks based build system?

See also opensource version of Blaze - [Bazel](https://bazel.build).

**[One version rule](https://opensource.google/documentation/reference/thirdparty/oneversion)**

Dependency management is a magnitude harder than source management.

**[Hyrum's law](https://www.hyrumslaw.com)**

> With a sufficient number of users of an API, 
> it does not matter what you promise in the contract: 
> all observable behaviors of your system will be depended on by somebody.

**Beyonce rule**

> If you liked it you should put a test on it.

**[Code search](https://developers.google.com/code-search)**

Global code search with full understanding of the structure and references greatly simplifies development.
 
> How others have done something with this API?
> Where a specific piece of information exists in the codebase?
> What a specific part of the codebase is doing?
> why a certain piece of code was added, or why it behaves in a certain way?

See also a primary building block of the code search: [kythe.io](https://kythe.io)

**[Testing on the Toilets](https://testing.googleblog.com/search/label/TotT)**

How to improve software engineering practices (for example testing) in a funny way.

**[Code review](https://google.github.io/eng-practices/review/)**

Very strict rules of the code review. 
Many roles: readability engineers, code owners, etc.
Readability: standardised mentorship through code review

**Sustainable software**

* Code must be easy to read, easy to understand and easy to maintain.
* Think about product lifecycle - years not days or months.
* When did you deploy a service last time?
* Are you ready to apply security fix in matter of hours?

> You may choose to not change things, but you need to be capable.

**Mocks**

Do not mock, use stubs or fakes.

**Flaky tests**

There are many flaky tests, they are inevitable at some scale. 
Learn how to live with flaky tests 😂. 

**Deprecation strategy**

Deprecation is hard, but without deprecation strategy you will be stuck.

**Scalability**

Manual tasks should be scalable (linear or better) in terms of human input.

**Goal / Signals / Metrics (GSM)**

* Start with goal and signals even if you can't measure them
* Make data driven decision, but in reality decisions are made based on mix of data and assumptions
* Before measure sth, ask if results are actionable.
  If you can't do anything with the results, do not measure at all.

**Documentation**

* Documentation is a software engineer responsibility
* Canonical source of knowledge, for example single Java style guide for the whole organization!
* Technical writers do not scale (but are useful if you don't "feel" the readers)
* Documentation as a code (avoid wiki)
* Use link shortener to create link to the canonical source

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

**Manager tips**

* Delegate but get your hands dirty
* Always seek to replace yourself, always be leaving

**Technical lead tips**

* Own "general problem" not a "particular tool", well-chosen problem can be evergreen

**[Style guides and rules](https://google.github.io/styleguide/)**

* Canonical source of style for the whole organization
* Focus on resilience to time and scaling

Tools:
* errors checkers (static code analysis)
* code formatters (https://github.com/google/google-java-format, https://github.com/google/yapf, https://clang.llvm.org/docs/ClangFormat.html, https://pkg.go.dev/cmd/gofmt)

**Test size**

* Easier to quantify than unit/integration/e2e (which is test scope)
