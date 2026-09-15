# Awesome Process-First Concurrency [![Awesome](https://awesome.re/badge.svg)](https://awesome.re) [![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

> Reach for OS processes and simple queues before actors or a distributed message bus. Justify the complexity.

A curated, decision-guide awesome-list on getting real concurrency out of the primitives your OS already gives you — processes, pipes, `cron`, and a queue table — before reaching for an actor runtime or a Kafka cluster.

## Contents

- [Introduction](#introduction)
- [Decision Heuristics](#decision-heuristics)
- [Process-Based Parallelism](#process-based-parallelism)
- [Simple Job & Queue Mechanisms](#simple-job--queue-mechanisms)
- [Message Brokers at "Just Enough" Scale](#message-brokers-at-just-enough-scale)
- [Actor Systems — and When They're the Right Call](#actor-systems--and-when-theyre-the-right-call)
- [Further Reading](#further-reading)
- [When You Actually Need an Actor System or Distributed Bus](#when-you-actually-need-an-actor-system-or-distributed-bus)
- [Contributing](#contributing)
- [License](#license)

## Introduction

Unix's oldest idea is still one of its best: small, single-purpose processes, composed with pipes and files, each one independently schedulable by a kernel that has been load-balancing processes across cores for decades. A shell pipeline, a pool of worker processes reading off a directory of files, or a handful of cron jobs writing to a lock file will comfortably carry a huge fraction of real-world background-work and fan-out workloads.

Yet it's common to reach straight for an actor framework or a Kafka topic for problems that a `xargs -P` invocation or a Redis list would solve in an afternoon. Actor systems and distributed message buses solve real problems — location transparency, back-pressure across a fleet of machines, exactly-once delivery semantics at scale — but they also import real costs: new failure modes, new operational surface, and a mental model (supervision trees, partitioning, consumer groups) that a process pool never asks you to learn.

This list is about the boring, load-bearing option: OS processes and simple queues first. Escalate to actors or a distributed bus only once you have a concrete reason — not because it's what the tutorial started with.

## Decision Heuristics

Rough thresholds for "processes are still fine" vs. "you've outgrown them":

- **Single machine, bounded fan-out** (tens to low thousands of concurrent units of work) → a process pool or `xargs -P` is enough. You don't need a scheduler that spans machines to schedule work on one machine.
- **Work items are independent and idempotent-ish** → a simple queue table (even SQLite) with a `SELECT ... WHERE claimed_at IS NULL` claim pattern gets you most of what a message broker gives you, without a broker to operate.
- **You need cross-machine fan-out** → this is the first real signal you need something beyond a single box's process table — but that alone doesn't mean you need Kafka; a lightweight broker like NATS core often bridges the gap.
- **You need in-order, partitioned, replayable delivery at high sustained throughput** (event sourcing, stream processing, multiple independent consumer groups replaying history) → this is genuinely Kafka/Pulsar territory.
- **Your unit of concurrency needs to hold long-lived, mutable, addressable state and receive messages over its lifetime** (a game session, a per-user connection, a saga/workflow instance) → this is what actor systems are actually for. A process pool re-derives state from scratch per task; actors don't.
- **You're reaching for actors or a bus mainly because of language/library hype, not a concrete scaling or state-modeling problem you've hit** → that's the signal to stay with processes and queues longer.

## Process-Based Parallelism

- [GNU Parallel](https://www.gnu.org/software/parallel/) — build and run parallel pipelines from the shell; handles job control, retries, and remote execution over SSH with one tool.
- [`xargs -P`](https://man7.org/linux/man-pages/man1/xargs.1.html) — the simplest possible parallel-process runner; already on every Unix box.
- [Python `multiprocessing`](https://docs.python.org/3/library/multiprocessing.html) — process-based parallelism with a `Pool` API, sidesteps the GIL for CPU-bound work.
- [Joblib](https://joblib.readthedocs.io/) — lightweight parallel-for helper built on `multiprocessing`/`loky`, popular in the Python data/scientific stack.
- [`concurrent.futures.ProcessPoolExecutor`](https://docs.python.org/3/library/concurrent.futures.html) — standard-library process pool with a futures-style API.
- [Ray](https://www.ray.io/) — the natural next step once you've genuinely outgrown a single machine's process pool and need distributed task/actor scheduling across a cluster.

## Simple Job & Queue Mechanisms

- [`cron`](https://man7.org/linux/man-pages/man8/cron.8.html) + a lock file (`flock`) — the original scheduled-job system; still correct and still running most servers' background work.
- [systemd timers](https://www.freedesktop.org/software/systemd/man/systemd.timer.html) — cron's more observable modern sibling, with logging and dependency ordering for free on systemd hosts.
- [Redis Lists / Streams](https://redis.io/docs/latest/develop/data-types/streams/) used as a plain queue — `LPUSH`/`BRPOP` or `XADD`/`XREADGROUP` give you a durable, shared queue without standing up a broker cluster.
- [SQLite as a queue](https://github.com/litements/litequeue) (e.g. `litequeue`) — a transactional claim-and-process queue backed by a single file; no server process to run at all.
- [Procrastinate](https://procrastinate.readthedocs.io/) — a Python task queue built directly on PostgreSQL, no separate broker.
- [pg-boss](https://github.com/timgit/pg-boss) — the same idea for Node.js: job queue on top of Postgres.

## Message Brokers at "Just Enough" Scale

- [NATS](https://nats.io/) (core pub/sub) — a single small binary, sub-millisecond latency, no ZooKeeper/controller cluster to run — often the right stop between "a Redis list" and "a Kafka cluster."
- [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/) with consumer groups — durable, ordered, multi-consumer semantics without a separate broker technology if you already run Redis.
- [Kafka](https://kafka.apache.org/) — reach for this once you need partitioned, replayable, high-throughput log semantics with multiple independent consumer groups; it is a real distributed system to operate, not a drop-in queue.
- [Apache Pulsar](https://pulsar.apache.org/) — Kafka-adjacent, with built-in multi-tenancy and tiered storage; same "you need this much" bar applies.

## Actor Systems — and When They're the Right Call

- [Erlang/OTP](https://www.erlang.org/) — the origin of the actor/supervision-tree model, built for systems that must keep running while individual processes crash and restart.
- [Elixir](https://elixir-lang.org/) — Erlang/OTP's ergonomics on the BEAM; a common modern entry point to the actor model.
- [Akka](https://akka.io/) (JVM) — mature actor toolkit for Scala/Java when you need addressable, stateful, message-driven units at scale.
- [Microsoft Orleans](https://learn.microsoft.com/en-us/dotnet/orleans/) — the "virtual actor" model: actors are always addressable and the runtime handles activation/placement for you.
- [Proto.Actor](https://proto.actor/) — a lighter cross-language (Go/C#/Kotlin) actor library if you want the model without the BEAM or JVM.

## Further Reading

- Rob Pike, ["Concurrency is not Parallelism"](https://go.dev/blog/waza-talk) — the talk that made the process/goroutine vs. parallelism distinction mainstream; useful for separating "I want things to happen concurrently" from "I need more throughput."
- The Unix philosophy as described in *The Art of Unix Programming* (Eric S. Raymond) — the "do one thing, compose via pipes" design ethic underlying process-first concurrency.
- Pat Helland, ["Life Beyond Distributed Transactions"](https://www.ics.uci.edu/~cs223/papers/cidr07p15.pdf) — good background on why distributed messaging designs get complicated once you actually need cross-node guarantees.

## When You Actually Need an Actor System or Distributed Bus

Be honest about the signal, not the hype:

- You have **long-lived, addressable, stateful** units of work (per-session, per-device, per-workflow) that need to receive messages over time — actors model this far better than re-hydrating state from a database on every task.
- You need **partitioned, replayable, ordered** delivery to multiple independent consumer groups at sustained high throughput — that's Kafka/Pulsar's actual job.
- You need **cross-machine fault tolerance with supervision** — a process crashing on one box and a queue message needing to be picked up by a different box, automatically, with retry/backoff policy — actor runtimes and brokers both do this natively; a bare process pool does not.
- You're past the point where "a bigger box and more worker processes" is a real option — genuine horizontal scale-out needs, not anticipated ones.

If none of those apply yet, a process pool and a queue table will very likely outlast your assumptions about needing more.

## Contributing

Contributions welcome! Please read [CONTRIBUTING.md](CONTRIBUTING.md) first — in short: resources should be real, actively relevant to the process-first-concurrency thesis, added in the right category in alphabetical order, and free of self-promotion spam. Open an issue with the [suggest a resource](../../issues/new?template=suggest-resource.yml) template or send a PR directly.

## License

[![CC0](https://mirrors.creativecommons.org/presskit/buttons/88x31/svg/cc-zero.svg)](LICENSE)

To the extent possible under law, the contributors have waived all copyright and related or neighboring rights to this work. See [LICENSE](LICENSE) (CC0 1.0 Universal).
