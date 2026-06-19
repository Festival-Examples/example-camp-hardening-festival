# Camp Hardening Festival — A Real Festival Example

A real, completed **Festival** you can browse end to end. Festivals are how agents and humans plan and execute serious work together: a goal broken into **phases → sequences → tasks**, each with its own gates, so long autonomous runs stay on track and stay reviewable.

This one — `camp-hardening-CH0001` — hardened the `camp` CLI against a class of data-loss bugs. It shipped: **4 phases, 12 sequences, 145 tasks, 100% complete.**

<p align="center">
  <img src="docs/img/fest-show.gif" alt="Animated fest show tree for the camp-hardening CH0001 festival" width="640">
</p>

<p align="center"><sub>Replay of the festival's real execution, rendered from <code>fest show</code> + its event log.</sub></p>

## Explore it

Everything is just files — open the festival and read it like the project plan it is:

```
camp-hardening-CH0001/
├── 001_INGEST/      # gather findings, define scope
├── 002_PLAN/        # structure the work
├── 003_IMPLEMENT/   # 12 sequences of tasks
└── 004_POLISH/      # verify, document, close out
```

Start at [`camp-hardening-CH0001/FESTIVAL_GOAL.md`](camp-hardening-CH0001/FESTIVAL_GOAL.md), then drill into any phase. Each task document is a self-contained unit of work an agent can pick up and finish.

## Festival

Festival is an open planning system for AI agents and humans — organize complex,
multi-phase work into auditable, long-running agentic loops.

- 🌐 **[fest.build](https://fest.build)** — homepage and docs
- ⭐ **[Star Festival on GitHub](https://github.com/Obedience-Corp/festival)** — if this looks useful, a star genuinely helps others find it

## Learn more

- Festival (camp + fest distribution): [Obedience-Corp/festival](https://github.com/Obedience-Corp/festival)
- The `fest` CLI and methodology: [Obedience-Corp/fest](https://github.com/Obedience-Corp/fest)
- Source of this example: [Obedience-Corp/camp#324](https://github.com/Obedience-Corp/camp/pull/324)
