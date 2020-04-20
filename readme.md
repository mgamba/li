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

### Events

Like the different subfolders of _src/_, these are independent of each other.

#### Crawler

Crawls data sources and writes the data to s3.

#### Scraper

Pulls latest data out of s3, parses the data as requested by the source, and passes it to a scraper function.  If the scraper function is successful, the resulting data is saved to a database.

#### Regenerator

This is a less used function to regenerate a source from cache.  Done with timeseries sources at least once a day.

Can also be invoked if a source goes offline and we missed some data.  Given that we have the needed cache, we can re-run the scrape function and then run the regenerator to bring the data back online.

# Using li

## Getting Started

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

## Running a Crawl

Let's use NYT as an example since it's a big data set.
```
./start --crawl nyt
```
This is calling the executable _start_ file in the root of the project with `--crawl nyt` option.  Inside the _start_ script,
1. we're letting bash know to interpret the file as javascript
  ```
  #! /usr/bin/env node
  ```
2. `crawl` is set to `nyt`
  ```
  let { crawl, scrape, regenerate, regenTimeseries, runner } = makeNice(args)
  ```
3. and a nyt event is dispatched to the crawler, resulting in a crawl/cache of the resulting data
  ```
  if (crawl) {
    await arc.events.publish({
      name: 'crawler',
      payload: {
        source: crawl
```

There should now be `.gz` archive(s) of the source in _crawler-cache/nyt/<YYYY-MM-DD>/_ with `<YYYY-MM-DD>` being today's date.

## Running a Scrape
Assuming we have some data that's been crawled (and saved to cache), we can scrape it.  In this case, we'll assume that we've already run the crawler for "nyt" and have it cached locally in _crawler-cache/nyt/_:
```
./start --scrape nyt
```
This will scrape/format today's data and write it to the database.

## Regenerating
Sometimes data needs to be re-scraped.  In this case, we can run a single command to iterate through each day and re-scrape the data.
```
./start --regenerate nyt
```

This is automatically run for timeseries sources, which tend to correct themselves over time.

# Understanding li

## Sandbox

The sandbox is necessary to operate the codebase.  It takes the events that get fired and run them through as though it was on AWS.  

Events can be run without the sandbox, but sandbox will give you the closest approximation to running events in production.

Once sandbox is running
- it will listen for events (crawler, regenerator, scraper)
- it has the http endpoints (headless, normal)
- regen-timeseries can be manually run

If you look at the _.arc_ file structure and the _src_ folder structure, you can see a resemblence to sandbox.

## Local Cache

In the root of the project is a _crawler-cache/_ directory that stores a local cache of downloaded sources.  Like _node_modules/_ there's a line in _.gitignore_ to ignore _crawler-cache/_ so that your local cache isn't commited to the repo.

Every data source has a unique key, which is derived from the file path of the source.  For example, the source in _src/shared/sources/us/ca/san-fransisco-county.js_ has the key `us-ca-san-fransisco-county`, and so all the downloads for that source are saved to _crawler-cache/us-ca-san-fransisco-county/_.  Downloads are further organized into timestamped folders within thier key directory, and saved as [Gzip](https://www.gnu.org/software/gzip/) archives.  This folder structure is similar in S3.

## Scrapers

Opening the source _src/shared/sources/nyt/index.js_, we see an array of scrapers:
```
...
  scrapers: [
    {
      startDate: '2020-01-21',
      crawl: [
        {
          type: 'csv',
          url: 'https://raw.githubusercontent.com/nytimes/covid-19-data/master/us-counties.csv'
        }
      ],
      scrape (data, date) {
...
```

This list of `scrapers` contains everything related to crawling and scraping.  There are 3 main parts of any scraper:
1. `startDate`
  - in [iso 8601 Date format](https://xkcd.com/1179/)
  - tells us when the source came online (not necessarily when it became part of the master branch)
  - helps to prevent extra regenerations for dates that don't exist
2. `crawl`
  - defines one or more sources to crawl
  - becomes first argument to `scrape`
  - `url` can be either a string or async function
  - if `url`'s a function,
    - generic http client gets passed in
    - should return final crawl url or object with cookie
  - `type` indicates what type of parsing should be done on the source data (ex. csv vs json)
  - `data` is optional and can be used to help rank a source (ex. geojson vs json)
  - `name` helps to differentiate between multiple crawlers for a source (ex. county vs city)
3. `scrape(data, date)`
  - `data` argument comes from an object in the `crawl` array
  - `date` argument is the date to scrape for
    - locale is assumed to be same as source data that's being crawled/scraped
    - formatted as an iso 8601 Date
    - use _shared/datetime_ for conversions
  - if there are multiple crawlers
    - multiple data sets will be returned and passed to the `scrape` function
    - these can be destructured in the first argument (ex. `scrape({county, city}, date)`)
    - and then accessed as individual variables (ex. `county.length`)

## Timeseries Data

Timeseries are anything that includes data over time.

### Example 1
Data is multiple days worth of data returned in a single request, often sorted by date.  Consider these two consecutive days of scraped data:
4/14
```
date,num_cases
2020-04-14,2187
2020-04-13,2213
2020-04-12,2343
...
```
4/15
```
date,num_cases
2020-04-15,1946
2020-04-14,2216
2020-04-13,2220
2020-04-12,2343
...
```

Notice that on 4/15, new data was added for that day.  But also the data for 4/14 and 4/13 was updated.  It's common for data to be updated over time in this manner to correct inaccuracies, especially for more recent data.

### Example 2



Time series are denoted by the `timeseries: true` on a data source definition.  For example in _src/shared/sources/nyt/index.js_:
``` 
...
module.exports = {
  country: 'iso1:US',
  timeseries: true,
...
```

# Credit

[Li_Wenliang](https://en.wikipedia.org/wiki/Li_Wenliang)
