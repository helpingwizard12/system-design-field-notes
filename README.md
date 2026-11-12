# System Design Field Notes

Learning system design from real engineering: books, foundational papers, and production case studies. Each folder is one reading, broken down section by section with diagrams, key lessons, and self-check questions.

## Contents

| # | Topic | Notes |
|---|---|---|
| 00 | The reading list ("the bible" and beyond) | [00-reading-list.md](00-reading-list.md) |
| 00 | How to read system design material | [00-how-to-study.md](00-how-to-study.md) |
| 01 | Google SRE Book, Ch. 24: Distributed Periodic Scheduling with Cron | [01-google-distributed-cron/](01-google-distributed-cron/README.md) |

## Pattern index

Patterns that keep coming back across readings. Add a row whenever a new reading uses one.

| Pattern | Seen in |
|---|---|
| Leader + replicated log (consensus) | Google cron (Paxos), Chubby, Raft, Kafka, etcd |
| Write intent before acting (write-ahead) | Google cron BeginLaunch, database WAL |
| Idempotency keys / unique attempt IDs | Google cron `job@time` names, Stripe `Idempotency-Key` |
| Snapshot + log compaction | Google cron, Raft, Kafka, databases |
| Majority quorum (2f+1 survives f) | Paxos, Raft, ZooKeeper |
| Prefer skip over duplicate (fail closed) | Google cron |
| Spread load with deterministic hashing | Google cron `?` syntax, Dynamo consistent hashing |
| Minimal dependencies for foundation services | Google cron, Chubby, ZooKeeper |
