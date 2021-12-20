# Design An Image Index

## What do i mean by image index?

- I want you to create a system to store all easily accessible new images posted on the web each day and build an application that lets a user search through all of those images.

### Questions and Answers

1. Will this be keyword search? (I query via keywords and the top images
   are returned, paginated?)

Yes: let's say the application returns 100 images per query and
have infinite scroll, so users can move forward and backwards from a
page. Let's imagine google images.

2. What is the amount of data that is queried by per, and what
   percentage of that is images?

Let's say 2.5 Quintillion Bytes of data to query a day, and ~40% of that
is images (1 Quintillion Bytes/ 1e18 Bytes)

3. What is the ratio of reads to writes?

Let's say that every day there are 6e21 (6 sextrillion Bytes) of data
are queried by our users. That should make a ratio of ~5000:1 for
reads.

4. Does the scraper need to tag malicious/inappropriate images and
   exclude them from search?

Let's say we won't have to worry about that for now.

5. Should the system follow robots.txt/not scrape certain websites?

Yes, if the robots.txt disallows our robot in the user agent, we won't
index it. This wont alter our numbers too much.

6. Will this system need to index more than once a day?

Let's assume this system refreshes its information once a day.

### Basic Numbers

First: Every day, we'll need to store the data: Let's say we need to
store 1e18 B a day. Let's say that we have an average cost of 10c/GB and
our average SSD size is 20TB (Western Digital's best commodity SSDs).

That means: 50 hard disks/PB or 50 \* 1e6 Hard Disks are required a day.
This means we need 50 million of these disks per day. These cost $350
each, so we'll need 17.5 B dollars to store one copy of this data on SSD
(or ~35B for 2 copies, and 50B for 3 copies for redundancy.)

(Just for reference, Western Digital's Annual Revenue is ~15B, so we
would buy their yearly stock of SSDs in one day).

In one year, about 1e-2 of these disks will fail, or 3e-3 a day. This
means that 1.5e5 of these disks will fail a day, costing about $50M/day.

This is probably too expensive, the cost to operate this system yearly
is > 17.5T, and the cost of dead drives > 175B/year.

As well, we'd need to actually write to these disks: to write 1e18 Bytes
to disk with 1GB/s write speed (SSDs) is 1e9 B/s. This would take 1e9
seconds, or 31 years to finish writing our write output for a day in
serial. With 1000 workers, this would take 1e6 seconds, and this could
feasibly be written to disk in a day.

As is, the question is impossible to do: I/O is too slow and this is
cost prohibitive. We could avoid SSDs, but then our write speed would
tank. On HDD, with write speeds of 50MB, this would take 600 years/day
of writes.

The absolute best network speeds right now are 1000 Gigabit, or 125
GB/s. The roughly average internet would be 1 Gigabit, or 125MB/s. ~t
125 GB/s internet, I/O ops would be the limiting factor at 1 GB/s.

Clearly this is not possible, so we'd go into another round of Q&A.

### Questions and Answers Round 2

1. How many images should we index a day?

5B images a day.

2. How big is the average image?

1MB.

3. On average, how much is each image viewed?

Let's say 99.99% of new images are viewed 0 times, and we read about
100:1 for the other images, so our reads are about 1/10 of our write
volume.

### Fixed Numbers

#### Writes

With 5B images a day at 1MB, this is 1e9 \* 1e6 bytes of data or 1e15 B.

To write this to disk daily would take 1e15 / 1e9 seconds, or just
barely doable (a little over 1 day).

If we're willing to use more compute, we can compress the images to be
5-50% their size based on compute: then our I/O could be done within 2 hours a day.

Compression can be done at about 50MB/s (5e7), so this would take 1e8
seconds or about 100 days. We would need about 100 workers in
parallel to make this step doable in a day's time as well.

As well, we're assuming we only have one writer: Since this work is
trivially parallelizable, we could have 1000 workers each writing to
their own disk. This would take us even less time, about 6 seconds to
issue all the I/O required.

It is possible to write all images either compressed or uncompressed
with parallelization.

#### Reads

For our reads, we assume that we will be reading the top 0.01% of the
images a lot of the time. Let's assume that since we're lazyloading,
each request will issue a read for a web-safe (compressed) version of
100 images per request.

At a good (95%) compression ratio, each image would be 50kb, and there
are 100 images, so this would be about 5MB over the wire to serve per
read. Assume there are 100k read requests per second, so we need to
serve 500GB of data a second over the wire.

To do so, we'll need to aggressively cache and also deduplicate blocks.

Let's assume that 1% of the searches result in 90% of the results. In
that case, we can cache 1% of our image set (10TB) in memory. We could
pretty much entirely hold this in memory with 3 copies of each image (a
1TB RAM instance is about 1k/month in the cloud), so we'd need 30
machines to act as CDNs.

The way our system would work would be like so: a query that the user
makes (e.g. "Brown dog") would go to a different service that would
return the location of the images (urls) for each searched image and a
copy of the image, for a query set. This would return the data + a
cursor to use for pagination. When the user hits the end of the page,
the frontend can issue another read with the cursor, and the database
can issue the location of the next 100 images, and so on and so forth.

In this case, since the app is designed to return the first page and
infinitely scroll, we only need to keep a primary key -> cursor relation
of the query that matches and the resulting cursor information.

This would be fast for data in the cache. We should do this for all hot
data for fast reads.

For infrequently accessed data, we could avoid one-hit wonders by using
something like a bloom filter (this is what akamai does). For any
infrequently accessed data, we could store it on disk, compressed, and
the first read can uncompress the data and serve it (so it will be
slow). We could then move the image to a "warm" storage (hot =
in-memory, warm = on disk, cold = compressed and on disk). Items can be
moved around at will, if required.

If the item is not queried again in the day, the worker will take the
image and place it in cold storage. Otherwise if it is queried for
twice, it could be moved to hot storage.

#### Tagging

Tagging is very compute extensive. We'll want to tag each image so it
shows up in a given query, and find a way to rank images. ML models
return a percentage of how much a given image matches, but we also need
to have some analytics to rank images based on how much people actually
use the image.

Assuming this ranking system works out, the read service will be able to
create a sorted order of images and return this to the client.

#### Deep Dive into Indexing

To index this data, we'll create an inverted index. The inverted index
would look like this, with a key-value pairing of image:url to termId.

```json
{
  "image:123": [1, 2, 3], // big brown dog
  "image:12": [1, 2] // big brown
}
```

The TermId table would look like this:

```json
["big", "brown", "dog"]
```

We do this because if we directly store the term dictionary in the
inverted index, if we decide to change any of the terms, we would need
to scan our entire inverted index to change all occurrences of the term.
Ouch. In this representation, we can do an extra lookup on reads to
ensure flexibility like this.

If we need to add steps to our workflow, we could use a DAG, where each
image is queued and asynchronously pulled off by a worker, and then
placed onto another queue for its next piece of work.

Web Crawler finds image -> AI tags -> Compressed -> Written to disk

Each step would be separated by a queue and could be persisted to disk
in between in order to keep its progress through the steps for easier
recovery.
