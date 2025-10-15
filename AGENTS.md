# AGENTS.md

Instructions for AI coding agents working on this Jekyll-based personal blog.

## Project Overview

This is a personal blog built with Jekyll and hosted on GitHub Pages. Content is written in Markdown and follows specific style and spelling guidelines.

## Build Commands

1. Install dependencies: `bundle install`
2. Update dependencies: `bundle update`
3. Run development server: `bundle exec jekyll serve --livereload --future`

## Content Quality Checks

Check markdown syntax and style:

```bash
markdownlint file_to_check.md
```

## General Style

- Write for a personal blog voice, not corporate/formal tone
- Write in first person (use "I" freely)
- Keep sentences short and easy to understand

## Technical Writing

- Be descriptive with headings (use full names like "BigQuery Storage Write API" in headings)
- Use adverbs when they add clarity (unfortunately, perfectly, etc.)
- Explain accessibility concepts using common terms (e.g., "healthy" for health checks)
- Format ranges and links naturally

## Terminology and Vocabulary

Always use the correct capitalization and spelling for:

Cloud & Infrastructure: BigQuery, Bigtable, Dataflow, Dataproc, GCS, Firestore, Pubsub

Technologies & Tools: Spark, Kafka, Beam, Flink, Akka, Gradle, Maven, Docker

Programming: Scala, Python, Go (the language), Ruby, SQL, WebAssembly, Monoid, Semigroup, Functor, Applicative, Contravariant, Kleisli

Testing: TDD, Scalacheck, Testcontainers, JUnit

Other: JVM, DNS, VPN, VLAN, API, SDK, CLI, CPU, SSD, NVMe

## Code Style

- Use Markdown for all blog posts in `_posts/` directory
- Follow markdownlint rules for consistent Markdown formatting
- Format code blocks with appropriate language identifiers

## File Organization

- `_posts/`: Blog post content (format: `YYYY-MM-DD-title.md`)
- `_pages/`: Static pages
- `_data/`: Site data files
- `_includes/`: Reusable HTML components
- `_site/`: Generated site (don't edit directly)
- `assets/`: Static assets (images, CSS, JS)
