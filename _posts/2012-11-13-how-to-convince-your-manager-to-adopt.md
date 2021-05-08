---
title: "How to convince your manager to adopt Git"
date: 2012-11-13
categories: [DVCS, git]
---

Distributed Concurrent Versions Systems (DCVSs) like Git or Mercurial has
changed software delivery processes significantly. I would not want to go back
to the mid ages of Subversion, and I'm able to convince almost any developer
to use DCVS. Convincing managers is much more tough task. Below I collected
some insights about DVCS which could help.
  
[Gartner - DVCS Begins to Come of
Age](http://blogs.gartner.com/tom_murphy/2012/05/10/dvcs-begins-to-come-of-
age/)  

> DVCS systems provide advantages in performance for branch and merge
operations thus enabling a smoother workflow during refactoring and with teams
that may be distributed. Over the last 6 months we have seen significant
investment into support for Git and Mercurial and other vendors who have been
delivering commercial DVCS systems This growing support will speed migration
for organizations that currently use Subversion but are struggling with
branch/merge overhead.

[ThoughtWorks Technology Radar (July
2011)](http://www.thoughtworks.com/articles/technology-radar-july-2011)  

> Starting from a challenge posed to the Linux community to stop using
commercial version control, Git has proved itself. Git embodies a well
architected, high performance implementation of distributed version control.
Git is powerful, so it should be used with respect, but that power enables
agile engineering workflows that simply cannot exist with other tools. Git’s
popularity is supported by the existence of GitHub. GitHub combines public and
private Git repositories, social networking, and a host of other innovative
tools and approaches.  
Some tools seek to enable and facilitate different ways of working.
Unfortunately other tools are created using a different premise, one of low
trust in users and the need to enforce a predefined process. ClearCase and TFS
do this. This makes version control systems with “implicit workflow”
unsuitable tools for modern agile software development. Project methodologies
and the best ways of working on a project need to emerge. Tools that enforce
high ceremony around things like check in just get in the way and kill
productivity.

![](https://lh4.googleusercontent.com/-RCjnnI3YBR8/U3usDDBQVTI/AAAAAAAAV9s/YbgJ-lD1D5M/s619/radar-tools-july-2011.jpg)

[Wikipedia](http://en.wikipedia.org/wiki/Distributed_Concurrent_Versions_System)  

> _String support for non-linear development_  
Git supports rapid branching and merging, and includes specific tools for
visualizing and navigating a non-linear development history. A core assumption
in Git is that a change will be merged more often than it is written, as it is
passed around various reviewers. Branches in git are very lightweight: A
branch in git is only a reference to a single commit. With its parental
commits, the full branch structure can be constructed.

> _Efficient handling of large projects_
Torvalds has described Git as being very fast and scalable, and performance
tests done by Mozilla showed it was an order of magnitude faster than some
revision control systems, and fetching revision history from a locally stored
repository can be one hundred times faster than fetching it from the remote
server. In particular, Git does not get slower as the project history grows
larger.

[From SVN to Git - Atlassian Case Study](http://blogs.atlassian.com/2013/01/svn-to-git-how-atlassian-made-the-switch-without-sacrificing-active-development/)  

> If Subversion has met my version control needs for many years, why should I
change? To me, that is the wrong question. The real question is, "How can DVCS
make what we do today even better"?
