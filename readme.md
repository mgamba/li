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

### @events

Like the different subfolders of _src/_, these are independent of each other.

#### crawler event

Crawls data sources and writes the data to s3.

#### scraper event

Pulls latest data out of s3, parses the data as requested by the source, and passes it to a scraper function.  If the scraper function is successful, the resulting data is saved to a database.

#### regenerator event

This is a less used function to regenerate a source from cache.  Done with timeseries sources at least once a day.

Can also be invoked if a source goes offline and we missed some data.  Given that we have the needed cache, we can re-run the scrape function and then run the regenerator to bring the data back online.

# Using li

## getting started

Start by forking and then cloning:
```
git clone git@github.com:<YOUR-USERNAME>/li.git
```

Install dependencies
```
npm install
```

Start the sandbox
```
npm run
```

## Sandbox

The sandbox is necessary to operate the codebase.  It takes the events that get fired and run them through as though it was on AWS.  

Events can be run without the sandbox, but sandbox will give you the closest approximation to running events in production.

Once sandbox is running
- it will listen for events (crawler, regenerator, scraper)
- it has the http endpoints (headless, normal)
- regen-timeseries can be manually run

If you look at the _.arc_ file structure and the _src_ folder structure, you can see a resemblence to sandbox.

## local cache

In the root of the project is a _crawler-cache/_ directory that stores a local cache of downloaded sources.  Like _node_modules/_ there's a line in _.gitignore_ to ignore _crawler-cache/_ so that your local cache isn't commited to the repo.

Every data source has a unique key, which is derived from the file path of the source.  For example, the source in _src/shared/sources/us/ca/san-fransisco-county.js_ has the key `us-ca-san-fransisco-county`, and so all the downloads for that source are saved to _crawler-cache/us-ca-san-fransisco-county/_.  Downloads are further organized into timestamped folders within thier key directory, and saved as [Gzip](https://www.gnu.org/software/gzip/) archives.


# Credit

[Li_Wenliang](https://en.wikipedia.org/wiki/Li_Wenliang)

