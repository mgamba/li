# About

li is written with the [architect](https://arc.codes/) serverless framework.

# Motivation

Gather learnings from underlying CDS codebase to allow robust data scraping and processing.
- retain sources from CDS (formerly "scrapers")
- failure tolerant
- little human intervention
- focus developer efforts on business logic instead of manual intervention

# High-level Overview

Looking at the main li repo, there are different subfolders of _src/_ that can be thought of as separate projects that know nothing about each other.  This seperation of projects is important because each project is deployed, run, and scaled independently.  For example, if a file in _src/events/crawler/_ requires `@architect/shared/cache/_hash.js`, then that shared dependency is copied from _src/shared/_ to _src/events/crawler/node_modules/_, like any other external dependency.

## Architecture

You can get a good idea of the architecture of the project by looking at the _.arc_ file at the root of the project.
