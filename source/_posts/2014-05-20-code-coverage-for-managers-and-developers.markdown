---
layout: post
title: "Code coverage for managers and developers"
date: 2014-05-20 16:13:08 +0200
comments: true
categories: [craftsmanship]
---

From time to time, people ask me what code coverage by tests should be. Does 60% mean that project is healthy? Or maybe the goal should be 70% or 80%?

_I don't know, it depends on your role in the project_

If you are a manager, Excel is your friend. Manager likes numbers, columns and charts. So the following rules are perfect for them:

* Below 20% - red, can not be accepted
* Between 20% - 50% - yellow, might be accepted conditionally
* Above 50% - green, accepted without doubts

But as a developer I know that the numbers are meaningless. I can easily achieve high code coverage by:

* Measure coverage for integration or even acceptance tests.
* Write a lot of unmaintainable tests.
* Testing supporting or generic domain instead of core domain, because it's easy.
* Testing setters and getters because it's easy.
* Testing without assertions.

Why would I do that?

* To meet acceptance criteria (sometimes enforced by the build/deployment tool).
* Christmas are coming, and my bonus depends on the code coverage level.

Instead focusing on the numbers, manager should promote TDD approach in the team.
Writing the unit tests should be a natural part of development process, not an exception.
But writing unit test (or tests in general) is not easy, especially for badly written code.

If you are a developer you need instant feedback what are you developing.
Waiting one day for Sonar code coverage report is not acceptable.
Write the tests, write the code, run the tests and check code coverage.
Verify that your tests cover all important lines and branches.
Generate report in isolation, only for test under development, check only coverage for tested part of the code.
Everything else is covered by accident.

Figure below illustrates coverage report in my IDE inlined with the tested source code.
Personally I configured Eclipse (STS) with JaCoCo, works like a charm.

{% img https://lh4.googleusercontent.com/-aQXF5ck4hcg/U3uHsdejeqI/AAAAAAAAV8c/-6Kmq7OK5hU/s878/sts-code-coverage.png %}

Code coverage reports and tools are not for managers, there are for developers to help them during development.
If you are a manager, focus on the people, their skills, and build TDD approach into the team.

_We are all professionals, aren't we?_
