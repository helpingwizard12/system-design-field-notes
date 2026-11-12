# The Reading List

## 1. Core books

| Book | Why |
|---|---|
| **Designing Data-Intensive Applications** (Martin Kleppmann) | "The bible." Replication, partitioning, consistency, transactions, streams, batch, explained with real systems. Check for the 2nd edition (with Chris Riccomini). |
| **System Design Interview Vol. 1 & 2** (Alex Xu) | Practical walkthroughs: URL shortener, chat, YouTube, etc. Read alongside DDIA. |
| **Database Internals** (Alex Petrov) | Storage engines and distributed consensus in depth. |
| **Site Reliability Engineering** (Google, free at sre.google/books) | How Google runs systems in production. |

## 2. Foundational papers (rough reading order)

1. Google File System (2003): distributed storage
2. MapReduce (2004): batch processing
3. Bigtable (2006): wide-column storage
4. Amazon Dynamo (2007): leaderless replication, eventual consistency
5. Chubby (2006) / ZooKeeper (2010): coordination and locks
6. Raft (2014): understandable consensus
7. Kafka (2011): the log as backbone
8. Spanner (2012): globally consistent database
9. Cassandra (2009), Facebook TAO (2013), Scaling Memcache at Facebook (2013)
10. Borg (2015): ancestor of Kubernetes

Paper summaries: Adrian Colyer's "the morning paper" archive, Murat Demirbas's blog.

## 3. Real engineering case studies

- Engineering blogs: Netflix, Uber, Discord ("How Discord Stores Trillions of Messages"), Cloudflare, Meta, Stripe, Figma, Slack, Shopify, Dropbox
- highscalability.com: "how X scaled" write-ups
- InfoQ / QCon talks
- **AWS Builders' Library**: retries, timeouts, load shedding (underrated)

## 4. Free structured material

- github.com/donnemartin/system-design-primer
- MIT 6.824 Distributed Systems (lectures + labs in Go)
- ByteByteGo (newsletter + YouTube)
- Jepsen.io: how real databases break under failure

## Suggested path

1. System Design Primer + ByteByteGo for vocabulary
2. DDIA, one chapter a week (the core)
3. After each DDIA chapter, read the matching paper
4. One engineering blog case study a week, sketch the architecture yourself
5. Optional: MIT 6.824 to build it
