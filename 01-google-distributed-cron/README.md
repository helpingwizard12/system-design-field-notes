# Google SRE Book, Ch. 24: Distributed Periodic Scheduling with Cron

Source: https://sre.google/sre-book/distributed-periodic-scheduling/

> **The whole chapter in one line:** A safe distributed system knows what it was doing when it crashed. It records intent before acting, uses names that make actions checkable, and prefers skipping over running twice.

---

## 1. Cron on a single machine

- **Failure domain = one machine.** If the machine dies, the scheduler and its jobs die together.
- Moving jobs to a second machine (e.g. over SSH) gives you **two** failure domains. Adding machines doesn't automatically add reliability.
- **Cron remembers nothing** except the crontab. Launches are fire-and-forget, so a job missed during a reboot is **silently skipped**.
- **anacron** saves a last-run timestamp per job, so it can catch up on missed daily jobs.

**Lesson:** to recover from failure, you need state.

---

## 2. Cron jobs and idempotency

Two questions decide what's safe for a job:

| | Skipping is OK | Skipping is NOT OK |
|---|---|---|
| **Running twice is OK** | Garbage collection | Cache rebuild |
| **Running twice is NOT OK** | Email newsletter | Payroll (hardest: exactly once) |

- **Idempotent** = doing it twice has the same effect as doing it once ("set balance = 100" yes, "add 100" no).
- **Google's choice: skip rather than double-launch ("fail closed").** A skipped run can be noticed and fixed. A double run (1M emails, double payroll) may be impossible to undo.
- The platform guarantees *at most once*. **Job owners must monitor** their jobs.

| Guarantee | Meaning | Example |
|---|---|---|
| At-most-once | may skip, never duplicates | Google cron |
| At-least-once | never skips, may duplicate | most queues (SQS, Kafka default) |
| Exactly-once | neither | usually at-least-once + idempotency |

---

## 3. Cron at large scale

```
Old:  crond --fork--> job (same machine)
New:  cron service --RPC--> Datacenter scheduler (Borg) --> job on any machine
```

1. **Decouple processes from machines.** Ask the datacenter scheduler for resources, not for a specific machine.
2. **Rescheduling costs time and loses local data.** GFS solves durability but is slow to recover. **Hot spares** solve speed. Durability and availability are separate problems.
3. **Containers need resources declared up front**, so a launch can be delayed. Cron must track the **full lifecycle** of each launch.
4. **Partial failure:** a launch can take several RPCs, and a crash midway leaves it half-done. On one machine things work or crash. Distributed systems can half-work.
5. **Spread replicas across failure domains** (power units, racks).
6. **Stay within one datacenter:** low latency, and cron shares fate with the scheduler it depends on. Don't make a component more reliable than its dependencies.

---

## 4. Building cron at Google

### 4.1 Tracking the state of cron jobs
Two options: an external distributed filesystem (GFS/HDFS), or a small amount of state kept inside the cron service. **Google chose the latter:**
- GFS is built for huge files. Cron's state is tiny, and small writes on GFS are slow and expensive.
- **Foundation services need minimal dependencies.** If cron depended on GFS, a GFS outage would stop thousands of jobs.

### 4.2 The use of Paxos
- A change is committed once a **majority** of replicas have it. Any two majorities overlap, so committed state can't be lost.
- **2f + 1 replicas survive f failures** (3 survive 1, 5 survive 2).
- The Fast Paxos leader **is** the cron leader, reusing what Paxos provides instead of building a second election.
- On leader death: detected within seconds, a hot follower takes over, then runs **cron-specific takeover logic**. Total time is under 1 minute.
- Two layers: Paxos answers "who is leader?", cron answers "what was the old leader doing?"

```
          +---------------+        +----------------+
          |  Leader (A)   |<------>|   Datacenter   |
          | launches jobs |        |   scheduler    |
          +---------------+        +----------------+
             ^          ^
       Paxos |          | Paxos
             v          v
    +--------------+  +--------------+
    | Follower (B) |<>| Follower (C) |
    +--------------+  +--------------+
```

### 4.3 Leader and follower roles

**Leader, for each launch:**
1. Sleep until the first job in its time-sorted list is due.
2. Write **BeginLaunch {job, scheduled time, config}** to Paxos and **wait for a majority to confirm**.
3. Send the RPCs to the scheduler.
4. Write **EndLaunch** to Paxos, *whether the launch succeeded or failed*. Cron tracks attempts, not job success.
5. **Lose leadership → stop talking to the scheduler immediately** (prevents split brain).

Why wait in step 2? Otherwise:
```
Leader launches foo@12:34 -> crashes before followers hear about it
New leader: "no record of foo@12:34" -> launches again -> DOUBLE LAUNCH
```
**Write intent before acting** (write-ahead logging).

Why job + time? For a job that runs every minute, "foo" alone is ambiguous. **job@time uniquely identifies one run.**

**Follower:** on BeginLaunch, update the next run time and add the launch to **open launches**. On EndLaunch, remove it. After election, the new leader must resolve every open launch.

**Network split:** a leader cut off from the majority must step down. The majority side elects a new leader.

### 4.4 Resolving partial failures (the core)

When the sender crashes or times out, **you cannot know whether a request was executed**. All of these look the same:
crashed before sending, message lost, or executed but the reply was lost.

To resolve an open launch you need **at least one** of:
- **A) Idempotent operations:** just resend everything.
- **B) The ability to look up state:** ask the external system what happened.

**Google's solution: precomputed, deterministic job names** (e.g. `foo-12:34`), shared with all replicas *before* launching. The new leader looks each name up and launches only what's missing.

**Why the time must be in the name:**
```
12:34:00 launch "foo"   12:34:01 leader crashes   (slow failover)
12:34:30 foo finishes and disappears
12:34:45 new leader: does "foo" exist? no -> relaunch -> DOUBLE RUN
```
With `foo-12:34` the scheduler can say "this exact run already completed."
**An ID must identify the attempt, not just the action.**

| System | Unique attempt ID |
|---|---|
| Google cron | job name + scheduled time |
| Stripe | `Idempotency-Key` header |
| Kafka producer | producer ID + sequence number |
| Databases | transaction ID |

**Recovery by crash point:**

| Leader crashed after | New leader sees | Action |
|---|---|---|
| Sleep (before Begin) | nothing | launch normally |
| BeginLaunch | open launch | lookup `foo-12:34`, not found, launch |
| RPC #1 / #2 | open launch | lookup, partly created, finish the missing parts |
| RPC #3 | open launch | lookup, fully launched (maybe done), don't relaunch |
| EndLaunch | closed launch | skip and schedule the next run |

If neither idempotency nor lookup is possible, you're back to choosing between skip and double-launch.

### 4.5 Storing the state
Paxos is an append-only log, so it **grows forever** and **must be stored somewhere**.

- **Snapshots** compact the log ("+1" × 1000 becomes "counter = 1000"). Recovery = latest snapshot + replay of the later log.
- Losing logs: small, bounded loss. **Losing snapshots: start from zero.**

| Data | Local disk ×3 | GFS backup | Why |
|---|---|---|---|
| Logs | yes | no | small frequent writes are slow on GFS; 3 copies are enough |
| Snapshots | yes | yes | critical; survives all 3 machines dying; written rarely |

- Worst case (all 3 machines die at once): lose the changes since the last snapshot. The snapshot interval controls the trade-off.
- A new replica copies the snapshot and logs from peers over the network, so it doesn't depend on any one machine's disk.

**Lesson:** protect data in proportion to how bad its loss would be, and store it where its write pattern is cheap.

---

## 5. Running large cron: the thundering herd

- Everyone schedules daily jobs at midnight (`0 0 * * *`). 30 teams × MapReduces with thousands of workers all start at once.
- Google added **`?`**, meaning "any value, the system chooses." The value comes from **hashing the job config** over the range (0-59, 0-23).
- Hash vs random: both spread load, but a hash is **stable day to day and identical on every replica** (deterministic randomness).
- Figure 24-3 is still spiky: some jobs really need exact times. You can reduce the herd, not eliminate it.

| Situation | Fix |
|---|---|
| Cache keys expire together | random TTL jitter |
| Clients retry after an outage | exponential backoff + jitter |
| Hot key rebuild stampede | request coalescing / locking |
| Millions of devices sync at once | spread by user-ID hash |

---

## Chapter summary

| Section | Core idea |
|---|---|
| Single-machine cron | No state means no recovery |
| Idempotency | Prefer skip over double run (fail closed) |
| Large scale | Decouple from machines; partial failures appear |
| Tracking state | Small critical state, minimal dependencies |
| Paxos | Majority agreement, one leader |
| Leader/follower | Write intent before acting; step down when cut off |
| Partial failures | Idempotency or lookup; unique per-attempt IDs |
| Storing state | Snapshot + log; protect by severity of loss |
| Large cron | Thundering herd, spread with hashing |

---

## Self-check questions

1. Why can anacron recover missed jobs but regular cron can't?
2. A job charges customers' cards monthly. Which idempotency quadrant is it in, and how do you make it safe?
3. Why isn't saving state to GFS enough by itself? What do hot spares add?
4. Give an example of a partial failure in an app you use (food order, bank transfer).
5. Why is GFS a poor fit for cron's state?
6. Should a company-wide login service depend on a shared analytics database? Why?
7. With 5 replicas, how many can crash while cron still works? Why not all but one?
8. Why reuse the Paxos leader as the cron leader?
9. Network split: A (old leader) alone, B and C together. What should A do?
10. Why must the leader wait for a majority on BeginLaunch before launching?
11. Why identify a launch by job + scheduled time?
12. The scheduler is down and a launch fails. Is EndLaunch still written? Why?
13. Why can't a new leader know whether a single RPC was executed?
14. Explain why `foo` causes a double run in the slow-failover case but `foo-12:34` doesn't.
15. You click "Pay" twice after the page freezes. How does the system charge only once?
16. Why is losing a snapshot worse than losing the log?
17. Why store snapshots on GFS but not logs?
18. Snapshots every 10 minutes and all 3 machines die. What's the worst-case loss? How would you reduce it, and at what cost?
19. Why hash the job config instead of picking a random time?
20. A daily push notification to 10M users at 8:00 AM crashes your servers. Fix it.

## Read next
- SRE Book Ch. 23: Managing Critical State (Distributed Consensus)
- Raft paper: "In Search of an Understandable Consensus Algorithm"
- DDIA Ch. 9: Consistency and Consensus
- AWS Builders' Library: "Timeouts, retries, and backoff with jitter"
