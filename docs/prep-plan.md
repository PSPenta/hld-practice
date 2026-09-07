# Interview Prep Plan (DSA · LLD · HLD)

Daily/weekly schedule for cracking **SDE3**-style loops at FAANG / Razorpay / tier-1 product companies.

**Assumes:** this repo for HLD, [lld-practice](https://github.com/PSPenta/lld-practice) for LLD, LeetCode (Blind 75+) for DSA.

**Language (Go / JS):** no separate daily slot — practice by **solving DSA and writing LLD in Go and/or JS/TS** (alternate languages by week or by problem).  
**HLD theory:** **1–2× per week** focused doc revision (not every day).

← [README](../README.md) · [Docs index](./README.md)

---

## Index

- [What this plan covers](#what-this-plan-covers)
- [Are these resources enough?](#are-these-resources-enough)
- [Day window (11:00 – 21:00)](#day-window-1100--2100)
- [Daily targets](#daily-targets)
- [Weekly rhythm](#weekly-rhythm)
- [Weekly targets](#weekly-targets)
- [Theory revision (1–2× / week)](#theory-revision-12--week)
- [Language via DSA & LLD](#language-via-dsa--lld)
- [8–10 week phases](#810-week-phases)
- [Win condition](#win-condition)

---

## What this plan covers

| Track | How |
|-------|-----|
| **DSA** | Timed LeetCode in **Go or JS** |
| **LLD** | Design + code in lld-practice in **Go or JS/TS** |
| **HLD practice** | Spoken redesigns from diagrams |
| **HLD theory** | **1–2 sessions/week** — read docs, notes, then apply next redesign |
| **Language depth** | Embedded in DSA + LLD (idioms, concurrency, errors) — not a standalone block |
| **Behavioral** | Sunday + story bank |

---

## Are these resources enough?

| Track | Resource | Verdict |
|-------|----------|---------|
| **HLD** | This repo (docs + diagrams) | Enough with weekly theory + timed spoken redesigns |
| **LLD** | [lld-practice](https://github.com/PSPenta/lld-practice) | Enough if you **code** designs |
| **DSA** | Blind 75 + ongoing LeetCode | Required; push into Medium/Hard mix |
| **Tech** | Go / Node on resume | Kept sharp via DSA/LLD in those languages |
| **You** | Ownership stories | Strong for Razorpay / fintech SDE3 |

**Still add:** weekly mocks, STAR stories, company depth (payments/ledger/idempotency).

---

## Day window (11:00 – 21:00)

~10 hours total. Typical: **office ~3h**, **lunch ~45m**, **breaks ~45m** → **~5.5h deep prep**.

| Time | Block | Focus |
|------|--------|--------|
| 11:00 – 11:15 | Warm-up | Plan day / yesterday mistakes |
| 11:15 – 13:15 | **DSA** (2h) | Timed problems in Go **or** JS → review |
| 13:15 – 14:00 | Lunch + walk | Off screens |
| 14:00 – 17:00 | **Office** (2–4h; default 3h) | Real work only |
| 17:00 – 17:15 | Break | |
| 17:15 – 18:45 | **HLD or LLD** (1.5h) | Alternate per [weekly rhythm](#weekly-rhythm) |
| 18:45 – 19:00 | Break | |
| 19:00 – 20:30 | Other design track or DSA weak topic (1.5h) | |
| 20:30 – 21:00 | Review | Mistakes + 1 takeaway |

**Flex:** office 2h → +1h DSA/mock; office 4h → one evening design block (1.5h) + shorter review.

**Theory days (1–2×/week):** replace one evening 1.5h block with **doc revision** (see below), or use part of Saturday.

---

## Daily targets

| Track | Target |
|-------|--------|
| **DSA** | **2** timed problems (or 1 Hard + 1 Medium) in Go or JS + review |
| **HLD** | **1** spoken redesign (40–50 min) on HLD days |
| **LLD** | **1** design coded in Go or JS/TS on LLD days |
| **Don’t** | Video binge without coding / redrawing |

---

## Weekly rhythm

| Day | Morning (2h) | Evening (after office) |
|-----|----------------|-------------------------|
| **Mon** | DSA | **HLD** redesign |
| **Tue** | DSA | **LLD** + code |
| **Wed** | DSA | **HLD** redesign |
| **Thu** | DSA | **LLD** |
| **Fri** | DSA | **HLD** Staff extras **or** mock **or** theory session |
| **Sat** | DSA 2–3h | **Theory** 1–1.5h (if not done Fri) + HLD/LLD mock or practice |
| **Sun** | Light DSA 1h | **Behavioral** 1h + rest |

Pick **Fri evening and/or Sat** for theory so it happens **1–2× weekly** without eating every design day.

---

## Weekly targets

| Track | Weekly |
|-------|--------|
| **DSA** | **12–15** problems (mix languages across the week); 1 contest if possible |
| **HLD practice** | **4–5** spoken designs from [Popular HLDs](../README.md#popular-hlds) |
| **HLD theory** | **1–2** sessions (2–4 doc sections total) — notes + apply next redesign |
| **LLD** | **4–5** designs coded in Go/JS |
| **Mocks** | **1** coding + **1** HLD |
| **Behavioral** | **3–5** STAR stories |

---

## Theory revision (1–2× / week)

**Not daily.** One solid pass beats shallow daily scrolling.

### Session (60–90m)

1. Pick **2–3** `##` sections (use each doc’s **Index**).  
2. Read + write **5–10 lines** each: when to use, vs alternative, pitfall.  
3. Next HLD practice day: **force** those concepts into the spoken design.

### First-pass order

Follow [docs curriculum](./README.md) A→E over weeks 1–4, then **spaced repeat** weak topics (Kafka vs SQS, cache stampede, saga, etc.).

---

## Language via DSA & LLD

| Practice | Language habit |
|----------|----------------|
| **DSA** | Week A mostly **Go**, week B mostly **JS** — or alternate problems |
| **LLD** | Implement the same design once in your primary interview language; optional second pass in the other |
| **Depth** | When a solution uses channels / goroutines / event-loop / async pools — **pause 5m** and explain it aloud |

No separate 30m lang slot. If an interview is language-locked, bias that week’s DSA/LLD to **~80%** that language.

Optional light platform refresh (ECS, SQS, Postgres `EXPLAIN`) folds into a **theory Saturday** or resume story prep — not a daily block.

---

## 8–10 week phases

| Weeks | Priority |
|-------|----------|
| **1–3** | DSA past Blind 75; HLD: redesign ✅ diagrams + **theory 1–2×/week**; LLD top patterns in Go/JS |
| **4–6** | DSA Medium→Hard; HLD stretch; LLD under timer; weekly mocks; continue theory spaced repeat |
| **7–10** | Interview mode: ~60% DSA + mocks, ~25% HLD, ~15% LLD/behavioral/theory refresh |

---

## Win condition

Weekly **mocks** + **spoken HLD** + **coded LLD** (in Go/JS) + **1–2 theory sessions** with notes you can recite.

Not: finishing every doc once with zero redesign, or a separate language course on the side.
