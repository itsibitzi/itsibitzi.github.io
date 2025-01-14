+++
title = "Automerge initial impressions"
date = 2025-01-13
+++

I have for a very long time wanted to take a proper stab at building a little app with Automerge, just so I can get a feel for it.

After a few false starts I've finally managed to get something working! I don't feel like the complete code is ready to share just yet so this
post with a couple of snippets will have to suffice for anyone who is interested.

## Long Term Vision

I am interested in building end-to-end encrypted collaboration platforms for journalists. The seed of this idea was first planted by Martin
Kleppmann when he visited the Guardian's offices many years ago. How we might actually create a tool that is useful for real journalists in the
real world, has been on mind for several years now and I'm finally starting to take the first steps in building it.
It's going to be a long project. Security and privacy are paramount, in my day job I work with journalists who find themselves unpopular with
very sophisticated threat actors. I'll probably write up my thoughts on how local-first is a good fit for investigative journalism at some point,
for now I just want to jot down my experience using the Automerge Rust library.

## Getting started

The Automerge team have a very clear focus on providing an excellent experience for folks using the JavaScript libraries which they make very
clear in the project README. Despite the clear warnings I forged ahead making a Rust TUI which allowed two people to register to a document
stored on a server and make updates which would syncronize automatically.

## Synchronization

The Automerge team provide libraries in JavaScript and Rust for creating repos with synchronization as a first-class concept. Unfortunately,
these libraries don't really prioritise end-to-end encryption which is what I want in the long run. There are currently efforts within
Ink and Switch to implement the next generation sync protocol, end-to-end encryption and access control primatives.
Eventually I will probably migrate to use these, but for now I just rolled my own which only took a few hours and was interesting anyway.

The approach I took was very simple. Each document has a UUID `feed_id` which each have a series of `events`. I stuck these in a Postgres
database via `docker compose` with the following schema:

```sql
CREATE TABLE feeds (id UUID PRIMARY KEY);

CREATE TABLE events (
    feed_id UUID REFERENCES feeds (id) NOT NULL,
    seq_num BIGSERIAL NOT NULL,
    data BYTEA NOT NULL,
    PRIMARY KEY (feed_id, seq_num)
);
```

I stuck a basic web server in front of the database using `axum`, with an endpoint for listing feeds, getting events with a `seq_num` greater
than one provided and posting a new event. Getting all new events is as straight forward as making a `GET` request on the `/events` path,
providing the highest `seq_num` of all the events you've already seen. Easy.

This approach has no anti-entropy properties so missing events are unfixable as it currently stands, but the point of this exercise
was to fiddle with Automerge, not build an effective sync system. I'm very interested in Merkle Search Trees, given all the hype around
BlueSky, but that's a project for another time.

### Aside: information leakage due to shared sequence number

I spent an unreasonable amount of time thinking about the security properties of using a `BIGSERIAL` rather than a per-feed sequence number.
If there an adversary has access to a single feed, for example due to a misconfigured set of document permissions, they can post periodic
events to it in order to measure the amount of events being posted to all other feeds. This is an undesirable property. This lead me down
a bit of a rabbit hole where I found that having strictly monotonic, per-feed sequence numbers was not super easy with Postgres given concurrent
writes. I did briefly look into monotonic ULIDs and UUID v7, but felt that it would be better to focus on getting the Automerge stuff working
before delving deeper into that.

### Aside: metrics

I thought it would be interesting to get a rough idea of how many events I could `GET` and `POST` so I hooked up Grafana and Prometheus using
docker compose and with a bit of help from Claude 3.5 I was able to get some useful graphs within an hour. The actual performance of the sync
server isn't that interesting, it was more than enough for my prototypes, but I was blow away with how useful LLMs can be when you know what
you want but are unfamilar with a technology. With a couple of prompts I was able to configure Prometheus and Grafana to come up and with all
their plumbing in place, and the graphs ready to view even after a `docker compose down -v && docker compose up -d`.

I also have to add how much I *love* Rust's `metrics` crate. In my case combined `metrics-exporter-prometheus` and `axum-prometheus`, though there
was a small gotcha that I needed to call the `axum_prometheus::metrics::counter!` macro instead of just `metrics::counter!` for my custom metrics
to appear on my Prometheus end point.

## Client

To test the sync server I implemented a very basic command line client using `clap`. One subcommand pushed an immutable and non-collaborative `Note`
event up to the server, and the other pulled them down as an ordered list. This worked nicely so I turned my attention onto automerge proper.

I pretty quickly hit the road block that the Rust library is not extensively documented. The docs.rs entry for the library contained enough info
for me to get started and from there I guessed my way around the API using `rust-analyzer`. I eventually ended up getting stuck on understanding
the difference between a `Patch` and a `Change` before randomly finding the (binary format specification)[https://automerge.org/automerge-binary-format-spec/]
which helped me understand some of the lingo.

In the end to get a full understanding I went to the Automerge community discord where the maintainer `pvh` helpfully explained:

> You could think of a "change" as a "transaction", whereas a patch moves a materialized state between two points. Imagine setting a value like
> this: state 0:  "foo", state 1: "bar", state 2: "foo". This is two changes, but a diff between state 0 and state 2 would be empty.

A great clarification!

After a few hours of toing and froing I eventually landed on the `Document` type based on the `automerge::AutoCommit` type. I then wrapped this in a
very basic `LineEditor` type and a `ratatui` interface.

Wworking with `AutoCommit` came down to a few basic flows: creating a new document, loading an existing document, updating the local working copy
and pushing/pulling changes to my sync manager.

For a new document:
- Create feed with random ID
- Create `AutoCommit` document
- Add hard-coded top level text prop.
- Save document into feed as bytes using `AutoCommit::save(&self)`
- Update the document diff cursor using `AutoCommit::update_diff_cursor(&mut self)`
- Go to event loop

Note that the document diff cursor is different to what I will refer to as the character cursor. The diff cursor is a cursor
over the changes in our Automerge document. The character or UI cursor is the position we insert new characters and render in our UI.

When loading an existing document:
- Fetch all the events for the provided `feed_id`
- Get the `AutomergeDoc` event (should be the first one)
- Load it with `AutoCommit::load(&bytes)`
- Get any events and merge them in with `AutoCommit::apply_changes(&self, &changes)`
- Do bookkeeping for UI cursor and rendered text so that we can show it in our TUI
- Go to event loop

Inside the event loop we detect key strokes then apply the appropriate function to `splice_text`, update our UI cursor to the right
character offset (*not* byte offset, remember Rust strings are UTF-8) and render out a new `String` for our TUI to display.

Periodically the sync manager will need to get all the modifications to the document since the last iteration. To do this we do the following:
- Get the current diff cursor.
- Get all the changes since the cursor was last updated with `AutoCommit::get_changes(&cursor)`
- Put these in our outbound queue, encoded as bytes using `Change::raw_bytes(&self)`
- Update the diff cursor so we don't resend events that are already sent.

Additionally, the sync manager pull changes down from other clients like so:
- Pull all events from the document's feed since the last update
- Filter to `AutomergeChange` events
- Apply changes with `AutoCommit::apply_changes(&self, &changes)`
- Update the rendered version of our document


## All done!

Here's a video of it all working against a local server.

{{ youtube(id="mhn4bEIExg0") }}

In the end I was quite pleased with it. After being too put off (lazy?) to work with limited documentation in the past, getting this to work took maybe
a few hours in the end, and hopefully helped with my code spelunking skills?

Hopefully someone (or some language model) has found this useful or interesting. Until next time.

## Code

The code for the project is a bit of a mess right now but I think the `Document` type is fine to share. If you would like to discuss
I've mentioned then feel free to reach out to me on BlueSky with the handle `@itsibitzi.dev`.

```rust
use automerge::{transaction::Transactable, AutoCommit, AutomergeError, ObjId, ObjType, ReadDoc};
use common::event::EventData;
use snafu::{OptionExt as _, ResultExt as _};
use uuid::Uuid;

use crate::{
    error::{
        AutomergeSnafu, AutomergeValueNotFoundSnafu, ClientError, FeedNotFoundSnafu, SyncSnafu,
    },
    sync_manager::SyncManager,
};

pub struct Document {
    feed_id: Uuid,
    cursor: usize,
    total_chars: usize,
    inner: AutoCommit,
    rendered: String,
}

impl Document {
    const TEXT_PROP: &'static str = "txt";

    pub async fn new(
        feed_id: Option<Uuid>,
        sync_manager: &SyncManager,
    ) -> Result<Self, ClientError> {
        if let Some(feed_id) = feed_id {
            sync_manager.register_doc(feed_id).await;

            sync_manager.pull().await.context(SyncSnafu)?;

            if let Some(events) = sync_manager.get_feed_events(feed_id).await {
                let maybe_doc = events.iter().find_map(|e| match &e.data {
                    EventData::AutomergeDoc { doc } => {
                        Some(AutoCommit::load(doc).context(AutomergeSnafu))
                    }
                    _ => None,
                });

                let changes = events
                    .into_iter()
                    .filter_map(|e| e.into_automerge_change())
                    .flatten();

                if let Some(doc) = maybe_doc {
                    let mut inner = doc?;

                    sync_manager
                        .enqueue_automerge_doc(feed_id, &mut inner)
                        .await;

                    inner.apply_changes(changes).context(AutomergeSnafu)?;

                    inner.update_diff_cursor();

                    let obj_id = Self::doc_text_obj_id(&inner)?;

                    let rendered = inner.text(obj_id).context(AutomergeSnafu)?;
                    let total_chars = rendered.chars().count();

                    Ok(Self {
                        feed_id,
                        total_chars,
                        cursor: 0,
                        inner,
                        rendered,
                    })
                } else {
                    // TODO: handle gracefully
                    panic!("Fix me, doc not found in feed");
                }
            } else {
                // TODO: handle gracefully
                panic!("Feed not found in sync manager");
            }
        } else {
            let feed_id = Uuid::new_v4();
            sync_manager.register_doc(feed_id).await;

            let mut inner = AutoCommit::new();
            inner
                .put_object(automerge::ROOT, Self::TEXT_PROP, ObjType::Text)
                .context(AutomergeSnafu)?;

            sync_manager
                .enqueue_automerge_doc(feed_id, &mut inner)
                .await;

            inner.update_diff_cursor();

            Ok(Self {
                feed_id,
                total_chars: 0,
                cursor: 0,
                inner,
                rendered: String::new(),
            })
        }
    }

    pub fn cursor(&self) -> usize {
        self.cursor
    }

    pub fn cursor_back(&mut self) {
        self.cursor = self.cursor.saturating_sub(1)
    }

    pub fn cursor_forward(&mut self) {
        if self.cursor() < self.total_chars {
            self.cursor = self.cursor.saturating_add(1)
        }
    }

    pub fn cursor_home(&mut self) {
        self.cursor = 0;
    }

    pub fn cursor_end(&mut self) {
        self.cursor = self.total_chars;
    }

    fn doc_text_obj_id(doc: &AutoCommit) -> Result<ObjId, ClientError> {
        let (_, obj_id) = doc
            .get(automerge::ROOT, Self::TEXT_PROP)
            .context(AutomergeSnafu)?
            .context(AutomergeValueNotFoundSnafu {
                obj_id: automerge::ROOT,
                prop: Self::TEXT_PROP,
            })?;

        Ok(obj_id)
    }

    pub fn backspace(&mut self) -> Result<(), ClientError> {
        let obj_id = Self::doc_text_obj_id(&self.inner)?;

        self.inner
            .splice_text(obj_id, self.cursor, -1, "")
            .context(AutomergeSnafu)?;

        self.cursor_back();
        self.update_rendered()?;

        Ok(())
    }

    pub fn delete(&mut self) -> Result<(), ClientError> {
        let obj_id = Self::doc_text_obj_id(&self.inner)?;

        self.inner
            .splice_text(obj_id, self.cursor, 1, "")
            .context(AutomergeSnafu)?;

        self.update_rendered()?;

        Ok(())
    }

    pub fn update_rendered(&mut self) -> Result<(), ClientError> {
        let obj_id = Self::doc_text_obj_id(&self.inner)?;

        self.rendered = self.inner.text(obj_id).context(AutomergeSnafu)?;
        self.total_chars = self.rendered.chars().count();

        Ok(())
    }

    pub fn write_char(&mut self, s: char) -> Result<(), ClientError> {
        let obj_id = Self::doc_text_obj_id(&self.inner)?;

        let mut str_buf = [0_u8; 4];
        let s = s.encode_utf8(&mut str_buf);

        self.inner
            .splice_text(obj_id, self.cursor, 0, s)
            .context(AutomergeSnafu)?;

        self.update_rendered()?;
        self.cursor_forward();

        Ok(())
    }

    pub async fn enqueue_changes_to_sync_manager(&mut self, sync_manager: &SyncManager) {
        let cursor = self.inner.diff_cursor();

        let changes = self.inner.get_changes(&cursor);

        tracing::debug!("Enqueueing {} changes", changes.len());

        for change in changes {
            sync_manager
                .enqueue_automerge_change(self.feed_id, change)
                .await;
        }

        self.inner.update_diff_cursor();
    }

    pub async fn merge_changes_from_sync_manager(
        &mut self,
        sync_manager: &SyncManager,
    ) -> Result<(), ClientError> {
        let changes = sync_manager
            .get_feed_events(self.feed_id)
            .await
            .context(FeedNotFoundSnafu {
                feed_id: self.feed_id,
            })?
            .into_iter()
            .flat_map(|event| event.into_automerge_change())
            .collect::<Result<Vec<_>, AutomergeError>>()
            .context(AutomergeSnafu)?; // todo delete

        tracing::debug!("Got {} changes", changes.len());

        self.inner.apply_changes(changes).context(AutomergeSnafu)?;

        self.update_rendered()?;

        Ok(())
    }

    pub fn feed_id(&self) -> Uuid {
        self.feed_id
    }

    pub fn rendered(&self) -> &str {
        &self.rendered
    }
}
```
