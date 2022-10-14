---
title: "Technical writing like a pro"
date: 2022-11-01
tags: [Markdown, Documentation]
---

I've managed this blog since 2011.
I'm also a primary contributor to the documentation managed by my team.
For example: on-boarding guide for new colleagues or a [tourist](https://blog.allegro.tech/2019/10/team-tourism-case-studies-1.html),
technical guides, definition of done, [architecture decision records](https://adr.github.io) and more.
As you perhaps already know, software engineers don't like writing technical documentation.
And as you also know, high quality documentation has tremendous benefits for engineering team or organization.

Today, I would like to share my experiences how to choose appropriate tools for writing better documentation.

## Documentation as a code

Documentation as a code is a key enabler for all tools and practices presented in this blog post.
It enables the culture where developers take ownership of documentation.
With documentation as a code culture, developers:

* Write documentation with the same tools and processes they use for development.
* Keep the documentation in the source code repository, close to the code.
* Write and refactor code and documentation together.
* Apply best practices like code review or pair programming.
* Automate documentation tests and deployments.

I used to write documentation using [Atlassian Confluence](https://www.atlassian.com/software/confluence) Wiki platform. Regardless of efforts, the documentation was always rusty.
{: .notice--info}

## Markup language selection

Use whatever plain text markup you like: Markdown, reStructuredText, AsciiDoc, DocBook or LaTeX.
Check only, if the format meets the following criteria:

* Must be readable using plain text editor.
How to make a code review for binary formats?
* Must be portable without worrying about compatibility.
It's likely that you will change the publication platform several times during documentation life.
* Must separate the content and its formatting.
Writer should focus on content and semantic not on look and feel.
* Should be easy to learn, and forgive syntax mistakes.
* Should be widely supported by developers tools.

I used to write documentation using Latex and DocBook.
Although, for the last 10 years I've been using Markdown.
{: .notice--info}

## Markdown into static website conversion

Markdown is perfect for a single README, but how to build documentation consists of many pages?
Some parser must convert bunch of Markdown files into a beautiful website.

Again, use whatever converter you like but check if the following capabilities exist for generated website:

* Website navigation, you must be able to see website structure and navigate between pages.
* Many themes, you expect different layout for documentation website and blog post.
* Table of contents, it simplifies navigation for complex pages.
* Search capabilities, documentation without search isn't so useful.

Because you are writing technical documentation look also for:

* Code syntax highlighting for many popular programming languages like [Rouge](https://github.com/rouge-ruby/rouge) or 
[Chroma](https://github.com/alecthomas/chroma).
* Diagrams and visualizations like [Mermaid](https://mermaid-js.github.io/) or [yuml](https://yuml.me)
* Math expressions like [MathJax](https://www.mathjax.org)

At the time of writing my favorite Markdown converter is [Jekyll](https://jekyllrb.com) but I would also like to check [Hugo](https://gohugo.io).
{: .notice--info}

With Jekyll you can serve your website locally using the following command:

```shell
$ jekyll serve --livereload
Configuration file: /Users/marcin/tmp/site/my-new-site/_config.yml
            Source: /Users/marcin/tmp/site/my-new-site
       Destination: /Users/marcin/tmp/site/my-new-site/_site
 Incremental build: disabled. Enable with --incremental
      Generating...
       Jekyll Feed: Generating feed for posts
                    done in 0.404 seconds.
 Auto-regeneration: enabled for '/Users/marcin/tmp/site/my-new-site'
LiveReload address: http://127.0.0.1:35729
    Server address: http://127.0.0.1:4000/
  Server running... press ctrl-c to stop.
```

Every edit re-generates pages for modified files and reloads web browser.
Very convenient way of writing documentation, you continuously observe the results of editing.

### "Minimal Mistakes" theme

For my personal blog I use [minimal-mistakes](https://mademistakes.com/work/minimal-mistakes-jekyll-theme/) theme.
Advertised as "perfect for hosting your personal site, blog, or portfolio."
Look at <https://github.com/mkuthan/mkuthan.github.io> repository if you want to know,
how to get minimal but fully functional blog posts website like mine 😀

### "Just the Docs" theme

For technical documentation I prefer [just-the-docs](https://just-the-docs.github.io/just-the-docs/) theme.
Below you can see documentation site of my team (please note that we're during migration from Wiki):

![Just The Docs Theme](/assets/images/2022-11-01-technical-writing/just-the-docs.png)

I couldn't open source code of the repository but I will be glad to share with you essential configuration and customizations.

`Gemfile` with Ruby dependencies for local run:

```
source 'https://artifactory.allegrogroup.com/artifactory/rubygems.org'

gem 'github-pages', '~> 227', group: :jekyll_plugins
```

Jekyll configuration file `_config.yml`:

```yaml
title: "Foobar team documentation"
url: https://foobar-documentation.gh.allegrogroup.com
logo: /assets/images/foobar-logo.png
remote_theme: just-the-docs/just-the-docs

# color configuration for {: .note } and {: .important }
callouts:
  note:
    color: blue
  important:
    color: red

# enable mermaid graphs
mermaid:
  version: "9.1.7"

# enable footer link for easy navigation to the top of the page
back_to_top: true
back_to_top_text: "Back to top"

# enable footer link to the GitHub editor for the page
gh_edit_link: true
gh_edit_repository: https://github.com/allegro-internal/foobar-documentation
gh_edit_link_text: "Edit this page on GitHub"
gh_edit_branch: "master"
gh_edit_view_mode: "edit"

# header links
aux_links:
  "Quick link":
    - "https://c.qxlint/foobar"
  "GitHub repository":
    - "https://github.com/allegro-internal/foobar-documentation"
aux_links_new_tab: true
```

Customized `_layout/page.html` to automatically generate table of contents on every page.
See [jekyll-toc](https://github.com/allejo/jekyll-toc) repository for `_includes/toc.html` file.

```
---
layout: default
---
<!-- generate TOC on every page, see _includes/toc.html -->
{{ '{%' }} include toc.html html=content %}
{{ '{{' }} content }}
```

That's all folks, Jekyll and "Just the Docs" theme do all the hard work and you could focus on writing the documentation. 

## Publication automation

The ultimate goal of the automation is to deploy documentation on every commit or merge to the main branch of the repository.
GitHub provides [GitHub pages](https://docs.github.com/en/pages), hosting service that takes HTML, CSS and JavaScript files
from a repository, runs the files through a build process, and publishes a website.

If you use only allowed Jekyll [plugins](https://pages.github.com/versions/) and 
[remote theme](https://github.com/benbalter/jekyll-remote-theme), 
GitHub publishes the site automatically.
You only have to configure "Pages" section in the repository settings.

![GitHub Pages Configuration](/assets/images/2022-11-01-technical-writing/github-pages.png)

### GitHub actions

If you are using plugins not supported by GitHub Pages, you have to build the website using GitHub actions.
Below you can see the action I configured for my blog:

```yaml
name: Build

on:
  push:
    branches:
      - master

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - uses: actions/cache@v3
        with:
          path: vendor/bundle
          key: {{ '${{' }} runner.os }}-gems-${{ hashFiles('**/Gemfile') }}
          restore-keys: |
            {{ '${{' }} runner.os }}-gems-

      - uses: helaili/jekyll-action@v2
        with:
          token: {{ '${{' }} secrets.GITHUB_TOKEN }}
          target_branch: 'gh-pages'
```

* To speed up the build configure cache for Ruby gems (line 14). Without the cache build takes 6 minutes, with cache 40--50 seconds.
* Push generated website to `gh-pages` branch (line 24) and configure repository to watch for the documentation in that branch instead of master or main.

According to [KISS](https://en.wikipedia.org/wiki/KISS_principle) principle, 
I would prefer automated publication from GitHub Pages than custom action.
{: .notice--info}

### Dependant bot

To keep Ruby gems and GitHub actions up-to-date configure [Dependant Bot](https://github.com/dependabot/dependabot-core) in `.github/dependantbot.yml` file:

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
  - package-ecosystem: "bundler"
    directory: "/"
    schedule:
      interval: "weekly"
```

## Bring editorial style guide to life

You are a developer not a technical writer, so any automated writing assistance is more than welcome.
 

* TODO: vale (installation + configuration)
* TODO: PR comments

## Markdown editor

* TODO: Visual Studio Code (extensions: jekyll run, code spell checker, vale)
