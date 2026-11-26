# Upswing

**Upswing is the engine behind a "what's trending right now" page, like the front page of Reddit or Hacker News.**

People share links. Other people upvote or downvote them and click on the ones they find interesting. Upswing takes all of that activity as it happens and keeps an up-to-date ranking of what's hot, refreshed within seconds.

> 🚧 **Status:** work in progress. This README describes what's being built.

## What it does, in plain terms

Imagine a busy notice board in a town square:

1. **Someone pins up a link.** That's a *post*.
2. **Passers-by give it a thumbs up or thumbs down.** Those are *votes*. Each person gets one vote per post and can change their mind, but can't vote twice.
3. **People stop and read the posts that catch their eye.** Those are *clicks*.
4. **Someone keeps rearranging the board** so the most popular, freshest posts sit at the top, and old ones slowly slide down even if they were popular once.

Upswing automates step 4, and it keeps working when the square gets very crowded.

## Why it's harder than it sounds

Counting votes is easy when ten people use a site. It gets hard when thousands do at once:

- **A viral post.** Suddenly half the visitors are looking at one post. The system mustn't buckle under that one hotspot.
- **Double counting.** On the internet, messages sometimes get sent twice (a flaky connection retries, a server restarts mid-task). Upswing has to make sure one click is counted exactly once.
- **Things break.** Servers crash, connections drop. When they come back, the counts must still be correct. No lost votes, no phantom votes.
- **Cheating.** A burst of votes from brand-new accounts in a few seconds looks suspicious and should be caught.

Upswing is built to handle each of these on purpose, and to **measure** how well it does.

## How it works (the 30-second version)

```
  You vote or click
        │
        ▼
  ┌────────────┐   saves it permanently   ┌────────────┐
  │    API     │ ───────────────────────► │  Database  │
  └────────────┘                          └────────────┘
        │
        │  announces "something happened"
        ▼
  ┌────────────┐                          ┌────────────┐
  │ Event feed │ ───────────────────────► │  Ranking   │
  │  (Kafka)   │                          │   worker   │
  └────────────┘                          └────────────┘
                                                │ updates the scoreboard
                                                ▼
  Front page  ◄──────── reads instantly ── ┌────────────┐
                                           │ Scoreboard │
                                           │  (Redis)   │
                                           └────────────┘
```

- **API:** the front desk. It takes your vote or click.
- **Database (PostgreSQL):** the permanent record. If everything else were wiped, the truth would still be here.
- **Event feed (Kafka):** a conveyor belt of "this just happened" messages, so the front desk doesn't have to wait for the ranking work to finish.
- **Ranking worker:** reads the conveyor belt and updates the scores.
- **Scoreboard (Redis):** a super-fast, in-memory copy of the rankings, so loading the front page takes milliseconds.

## How "hot" is decided

A post's score depends on two things: **how many net upvotes it has** and **how new it is**. The first few votes matter a lot; after that, each extra vote counts a little less. Meanwhile, newer posts get a steady head start over older ones. The result: a brand-new post with a handful of votes can beat a day-old post with hundreds, which keeps the front page fresh.

There's also a **"trending in the last hour"** view based on clicks, so you can see what people are reading *right now*.

## Where the content comes from

To make it realistic, Upswing is filled with real stories from the public Hacker News feed. Load tests then simulate thousands of users browsing, voting and piling onto a single viral post.

## Built with

Java 21 · Spring Boot · Apache Kafka · Valkey (Redis-compatible) · PostgreSQL · Prometheus & Grafana (dashboards) · Docker Compose · k6 (load testing)

Everything is open source and runs on a single laptop with one command.

## License

[MIT](LICENSE)
