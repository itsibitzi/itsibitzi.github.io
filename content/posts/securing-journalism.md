+++
title = "Securing journalism with local-first software"
date = 2024-12-01
+++

I've spent the better part of the last 9 years thinking about and building tools which assist in the process of
investigative journalism. As an original member of the Guardian's Digital Investigations team which specialises in
building tools to improve the process of newsgathering.

Over the years I've developed my thinking on what an idealized tool for investigative jouranlists could look like.

## Current state of the art

The Guardian's Digital Investigations team was started as a somewhat delayed reaction to the
[Snowden revelations](https://www.theguardian.com/world/2013/jun/06/us-tech-giants-nsa-data) which, despite it's impact,
was reported upon in a somewhat flatfooted manner from a technological perspective. Leaked documents were provided to journalists
at a time when there was simply no in-house tooling avaiable to process them, so they ended up using off-the-shelf eDiscovery software.
This worked well, and the story was a massive success, but internally people started to think about how we'd do it better next time.

> Around the same time, engineers at the Guardian became aware of how limited their interactions with the reporting part of
> journalism was. "We start when they finish", one senior engineer is quoted as saying, referring to the fact that there was all the
> engineering effort was put in as reporters completed their work and production staff took over.

A few years later the project to create a document search tool was green lit, and I was extremely fortunate to be on that original
crew.

The original set of requirements focused on a few key areas:

**Security**, the documents had to be confidential.

**Deployable on-premises and within the cloud**. This allows us to put most stuff in the cloud where resources are plentiful but gives
us the option to fall back to a more secure on-premises solution if the threat model required.

**High performance**, the tool should be fast and work for large amounts of data. The trend at the time was that datasets were growing
larger and larger (see Panama Papers, Paradise Papers, Pandora Papers, and other such double P alliteration).

**Journalists should be able to browse documents before they'd finished processing.** At the time the thinking was that it would be helpful
if an ingestion was to take several days (which was not uncommon) then it would be great if jouranlists could start to take a look before
that process was finished. In the end this turned out to be an _enormous_ anti-feature. Journalists hated the idea that they could do a
search one day and get no results, and then the same search the next day would give them hits. This would force them to have to memorize
what queries they'd done at different states of ingestion, and then rerun them once things were finished. It seems obvious now that this
wasn't the greatest idea, but I content that there's possibly ways to make it better with some clever UX. An obvious example might be something
like the system remembering what queries you've ran while ingestion is still in progress and giving you some feedback that there are some new
results.

**Journalism first** existing tools were primarily intended for use in computer forensics by police, legal teams, or corporate investigators.
By building our own tooling we could tailor the usecase towards journalists and journalism.

The tool we ended up building was a elasticaly scaling ingestion and indexing pipeline in AWS with a frontend web interface which
allows journalists to type in search queries and browse documents on big datasets (if memory serves the biggest we've worked with
measured in at around 1TB). It's [open-source](https://github.com/guardian/giant) so you can have a play around if you'd like.

We were [not](https://github.com/ICIJ/datashare) [the](https://github.com/alephdata/aleph) [only](https://github.com/liquidinvestigations) people
who managed to come up with the idea of having a load of computers performing ingestion and search. Several other organizations have their own
investigations platform with their own benefits and drawbacks. It's a bit of a shame that we've found it so difficult to collaborate on these
tools as an industry, but I digress.

## Local-first, an unexplored paradigm in journalism?

Before the advent of the internet, cloud computing, and software as a service, essentially all journalism was conducted in a way that could
reasonably be called "local-first". Journalists wrote their notes in books and their copy in desktop software (or typewriters before that).
As the web began to take over, aspects of the editorial process began to move online, and it was only natural that engineers born of that era
would build their new document search tooling online too.
