# Day 18 — Cosmos DB Design: Generate and Govern

**Birlasoft FORGE FDE Academy · Sprint 1 Day 18 · Learner Handout**

---

## How this works

You are not going to write this code. You are going to **specify it, read what
comes back, and find what is wrong with it.**

Cosmos is unusually good practice for this, because a model has absorbed a
great deal of relational and generic-NoSQL advice — and several pieces of it
are **wrong here in ways that cost real money**:

> *"Use `/id` as the partition key."* · *"Add an index for that query."*
> *"Measure the latency."* · *"Provision throughput for production."*

Each is reasonable somewhere. Here, each one produces a container you cannot
fix without migrating the data.

Each exercise gives you:

| | |
|---|---|
| **The brief** | What has to be achieved. Read this first. |
| **The prompt** | Paste into Gemini. Improve it if you can. |
| **Accept it when** | Your checklist before moving on. |
| **It will probably get wrong** | Do not read until you have reviewed the output yourself. |

> **Work in that order.** Reading the failure list first turns a judgement
> exercise into spot-the-difference.

---

## Setup — before Exercise 1

Run by hand. Not part of the exercise.

```python
%pip install -q azure-cosmos
import os, json, time, uuid, random, getpass
from collections import Counter
from datetime import datetime, timezone
from azure.cosmos import CosmosClient, PartitionKey, exceptions

URL = input("Cosmos URI: ").strip()
KEY = getpass.getpass("Primary key (hidden): ").strip()

client = CosmosClient(URL, credential=KEY)
db = client.create_database_if_not_exists(id="forge_day18")

N_CASES = 400          # NOT 4,000. Raise it only once everything works.
PREFIX  = "ds"         # every container you create must start with this
print("ready")
```

### Two rules for today, before you generate anything

**One.** Every container you create must be named `ds_something`. Exercise 6
deletes everything with that prefix. A container you cannot find is a container
you keep paying for.

**Two.** This must be a **Core (SQL) API** account. If yours is MongoDB or
Cassandra API, stop now and get a different one — none of this will work and
you will spend twenty minutes finding out why.

---

# Exercise 1 · The RU meter

**15 minutes**

### The brief

Every measurement today depends on reading one header: `x-ms-request-charge`.

> **RU is the currency.** You provision RU, you consume RU, and the bill is RU.
> Latency follows from RU rather than the other way round.

There is a trap in this cell that catches almost everybody, and it is worth
finding yourself.

### The prompt

```
Write a Python helper for a Colab notebook that measures the Request Unit cost
of Cosmos DB operations.

Context: `db` is an existing DatabaseProxy from azure-cosmos.

Requirements:
- A function `charge(container)` returning the RU consumed by the most recent
  operation on that container, read from the x-ms-request-charge response header
- A function `q(container, query, params=None)` that runs a SQL query and
  returns the results as a list
- Both must work with azure-cosmos 4.16 or later

Important: azure-cosmos returns queries as a LAZY iterator. Explain in a
comment what that means for when the RU charge becomes available, and make
sure `q()` handles it correctly.

Also write a short comment on why we measure RU rather than milliseconds.
```

### Accept it when

- [ ] `q()` **fully exhausts** the iterator before `charge()` could be called
- [ ] The comment explains the lazy-iterator consequence correctly
- [ ] The RU/milliseconds comment is right
- [ ] It does not pass `enable_cross_partition_query`

### It will probably get wrong

> Read only after your own review.

- **Reading the RU charge before consuming the iterator.** This is the trap. A
  query in azure-cosmos does not execute until you iterate it, so
  `container.query_items(...)` followed immediately by reading the header gives
  you **0**, or worse, the charge from the *previous* operation. You get a
  plausible-looking table of numbers that are all wrong and none of them look
  wrong. Wrap in `list()` first.
- **`enable_cross_partition_query=True`.** Required in older SDKs, **removed in
  current ones** — passing it raises `TypeError`. Generated code copies it from
  older documentation constantly. If yours has it, that is a live example of
  why volatile vendor details belong in speaker notes rather than slide bodies.
- **Timing with `time.perf_counter()` and reporting milliseconds.** Not wrong,
  but not the point. Latency here is a *consequence* of RU and of your network
  position. The RU number is the one that transfers to a bill.

---

# Exercise 2 · Three containers, three partition keys

**30 minutes · the headline measurement**

### The brief

Your brief lists three query patterns:

| Pattern | Volume | What it is |
|---|---|---|
| `get by case_id` | 40,000 / month | the hot path |
| `list by applicant_id` | 3,000 / month | a handler checking history |
| `filter by status` | 6,000 / month | the operations queue |

You get **one** partition key, and it is **immutable**.

Load the same documents three times, partitioned three ways, and measure what
each pattern actually costs.

### The prompt

```
Write a Colab cell that compares three Cosmos DB partition key candidates by
measuring real RU cost.

Context: `db` is a DatabaseProxy. `charge()` and `q()` exist from the previous
cell. N_CASES = 400. PREFIX = "ds".

Requirements:
- Generate N_CASES synthetic case documents with fields: id, case_id (unique,
  format CLM-2026-NNNN), applicant_id (about N_CASES/3 distinct values),
  status, and a nested ai_response object of roughly 2 KB
- status must be drawn from [pending, extracting, generating, complete,
  needs_human] with a LOPSIDED distribution — about 74% complete, because most
  cases finish
- Create three containers named ds_by_case_id, ds_by_applicant_id, ds_by_status,
  partitioned on those fields. Delete and recreate if they already exist.
- Load the same documents into all three, recording total write RU for each
- Then run each of the three query patterns against each container, 12 samples
  each, recording the MEDIAN RU:
    get by case_id        — use a POINT READ where the partition key allows it
    list by applicant_id
    filter by status
- Print a 3x3 table of RU

Do not use enable_cross_partition_query.
```

### Accept it when

- [ ] `get by case_id` on the `case_id` container is a **point read** (`read_item`), not a query
- [ ] That cell shows roughly **1–2 RU**; the cross-partition cells show hundreds
- [ ] The status distribution is genuinely lopsided
- [ ] Containers are named with the `ds_` prefix
- [ ] Write RU is roughly **the same** across all three

### It will probably get wrong

- **Using a query for every pattern, including `get by case_id`.** A query
  filtered on the partition key is cheap, but a **point read** — `read_item`
  with both id and partition key — is the cheapest operation Cosmos offers and
  it is what your 40,000/month pattern should be. If the generated code queries
  throughout, your headline number is several times too high and the argument
  weakens.
- **An even status distribution.** If `complete` is 20% rather than 74%, the
  hot partition in Exercise 3 never appears and the session's most useful
  finding evaporates. Check the actual counts.
- **Explaining the RU differences as "indexing".** Generated commentary often
  suggests adding a composite index to fix the expensive patterns. That is a
  relational instinct. In Cosmos, indexing helps within a partition; it does not
  stop a cross-partition query from **visiting every partition**.

### Then answer

Nothing wins all three. **What do you actually do about the other two
patterns?**

The answer is not a second index. It is a **second copy of the data**, shaped
for that question, kept current by the change feed.

> You do not solve a second access pattern with a second index.
> You solve it with a second copy of the data.

---

# Exercise 3 · Skew, and the hot partition

**20 minutes**

### The brief

A container-level RU figure hides the thing that will actually page you at
9am. Measure how the documents are really distributed, then work out what that
does at peak.

### The prompt

```
Extend the previous cell to measure partition skew and predict throttling.

Requirements:
- For each of the three containers, count how many documents land on each
  distinct partition key value. Use a GROUP BY query, and fall back to
  counting client-side if GROUP BY is unavailable on this account tier.
- Report, per container: number of distinct logical partitions, and the share
  of documents on the largest one
- Flag any container where the largest share exceeds 35%

Then model a peak-load scenario:
  500 operations/second at 8 RU each = 4,000 RU/s demand
  6,000 RU/s provisioned at the container level
  A single PHYSICAL partition is capped at approximately 1,000 RU/s

- For each partition key, compute the demand landing on the largest partition
  and print whether it throttles
- Explain in a comment the difference between a logical partition and a
  physical partition, and which one the 1,000 RU/s ceiling applies to
```

### Accept it when

- [ ] Logical and physical partitions are **distinguished correctly**
- [ ] The 1,000 RU/s ceiling is attributed to the **physical** partition
- [ ] `status` throttles; `case_id` and `applicant_id` do not
- [ ] The container-level headroom is shown alongside, so the contradiction is visible

### It will probably get wrong

- **Conflating logical and physical partitions.** These are different things and
  most explanations blur them. A logical partition is *all documents sharing one
  partition key value*. A physical partition is *the machine several logical
  partitions sit on*. The ceiling is physical, which is why cardinality alone
  does not save you — you need cardinality **and** even distribution.
- **Concluding "provision more RU".** This is the most expensive wrong answer in
  the session, because it costs money and changes nothing. A single physical
  partition cannot exceed its ceiling no matter what the container is
  provisioned at.
- **Not showing the headroom.** The whole point is the contradiction: 429s while
  the dashboard shows 2,000 RU/s spare. If the output does not put those two
  numbers next to each other, the finding does not land.

### Then answer

You get paged at 9:05am. The container dashboard shows plenty of spare capacity
and the API is returning 429s. **What do you look at first, and what do you say
to the person asking you to increase the provisioned throughput?**

---

# Exercise 4 · Document shape

**20 minutes**

### The brief

Cosmos charges write RU by **document size**. A case changes status four times
in its life.

Measure what that costs in each of two designs.

### The prompt

```
Write a Colab cell comparing two document designs by measuring real write RU.

Design A — one fat document: status, extracted_fields and a ~2.5 KB
ai_response object all in the same document.

Design B — hot/cold split: a small document carrying only id, case_id,
applicant_id, status, updated_at and a result_ref pointer; with the large
result written once to a separate document.

Requirements:
- Create containers ds_shape_fat and ds_shape_split, both partitioned on case_id
- Write 120 cases into each
- Then apply THREE status updates to every case, as a real lifecycle would
  (pending -> extracting -> generating -> complete)
- Record: document size in KB, RU per initial write, and RU per STATUS UPDATE
- Print a comparison table and the ratio between the two designs
- Project both to 16,000 status changes per month

Explain in a comment what is being rewritten on each status update in
design A, and whether it needed to be.
```

### Accept it when

- [ ] Status updates are measured **separately** from the initial write
- [ ] The split design shows roughly a **6× lower** RU per status update
- [ ] The comment identifies that the narrative is rewritten unnecessarily
- [ ] The monthly projection uses the **update** figure, not the initial write

### It will probably get wrong

- **Only measuring the initial write.** The initial write happens once. The
  status update happens four times, and it is where the cost lives. If the cell
  reports "3.0 KB vs 0.5 KB, 15 RU vs 2.5 RU" and stops, ask what a month
  actually costs.
- **Using `patch_item` to make the fat document cheap.** Partial update is a
  real feature and worth knowing, but it changes the exercise into a different
  one. If the generated code reaches for it, that is a good conversation — ask
  whether patch is available on your account and API version, and whether you
  want your write cost to depend on remembering to use it.
- **Not mentioning the cost of splitting.** There is one: an extra read when
  somebody wants the full result. It is a point read, which is the cheapest
  thing Cosmos does, but an honest comparison states it.

---

# Exercise 5 · The change feed, and what your audit will miss

**25 minutes · raise this today, not in Sprint 2**

### The brief

The change feed is enabled today so it can become your audit trail in Sprint 2.

Before then, find out what it actually captures.

### The prompt

```
Write a Colab cell that measures what the Cosmos DB change feed observes.

Requirements:
- Create a container ds_changefeed partitioned on case_id
- Read the change feed once from "now" to establish a starting position
- Then move 30 cases through four states each — pending, extracting,
  generating, complete — by upserting the same document id each time.
  That is 120 writes.
- Wait a few seconds, then read the change feed from the beginning
- Print: how many transitions actually happened, how many changes the feed
  reported, and how many were LOST
- Print a Counter of which statuses the feed actually reported

Handle the fact that query_items_change_feed has changed signature across SDK
versions — try is_start_from_beginning, then start_time, then no arguments.

Then explain what the default change feed mode returns, and what that means
for an audit trail that must evidence every state a case passed through.
```

### Accept it when

- [ ] The feed reports far fewer changes than transitions — roughly **75% lost**
- [ ] The reported statuses are almost entirely `complete`
- [ ] The explanation distinguishes *latest version* from *every version*
- [ ] The SDK signature variation is handled

### It will probably get wrong

- **Claiming the change feed is a full audit log.** It is the most likely
  generated statement in this exercise, and it is the one that costs you Sprint
  2. The default mode returns the **latest version of a changed document**, not
  every version it passed through. A case that moves through four states faster
  than the polling interval is observed **once**.
- **Reading from the beginning without establishing a start position**, so the
  count includes writes from earlier cells and the arithmetic is meaningless.
- **Not waiting before reading.** The feed is not instantaneous. Without a
  pause you may see nothing at all and conclude something much more dramatic
  than what is happening.

### Then do the design work

Two ways out. Cost both:

**All-versions mode** — every version reaches the consumer, no application
change, more feed volume and more cost. Availability varies by account
configuration and API version, so **check yours rather than assuming.**

**An explicit transition record** — an append-only event on every state change.
Never mutated, so nothing can be lost, and you control what is captured.

> The second is what an auditor actually wants, because it is a **log** rather
> than a stream of snapshots. A change feed tells you what a document looks
> like *now*. A transition record tells you *what happened and in what order*.
> Only one of those survives an investigation.

It also fits the Sprint 0 Day 6 ordering constraint — the audit record is
written **before** the result is released. A transition record can be. A change
feed observation cannot, because it happens after the write.

**Measure it.** Write the transition records for those same 120 transitions and
report the RU per record and the monthly cost at 16,000 status changes. Put
that number in the ADR next to the sentence about what the change feed misses,
and let somebody decide with both in front of them.

---

# Exercise 6 · The design record — and clean up

**20 minutes**

### The brief

Produce the container design that goes into the ADR, and **delete everything
this session created**.

### The prompt

```
Write two Colab cells.

CELL ONE — the design record. Build a JSON object capturing:
- the chosen partition key, the two rejected candidates, and the measured
  monthly RU for each so the rejection is evidenced rather than asserted
- measured RU per query pattern, per candidate
- partition skew per candidate
- document shape: KB and RU per status update for both designs
- change feed: transitions, observed, percentage lost, and the RU cost of the
  transition-record mitigation
- throughput mode recommendation, with the monthly RU figure behind it
- review triggers

Project the monthly cost for each partition key using these volumes:
  get by case_id 40,000 / list by applicant 3,000 / filter by status 6,000 /
  writes 16,000
At serverless pricing of roughly $0.28 per million RU.
Note that provisioned throughput has a 400 RU/s minimum.

Write it to cosmos_design.json.

CELL TWO — cleanup. Delete every container in the database whose id starts
with "ds_". List what was deleted. Leave the database itself, but include a
clearly marked flag to drop it too.
```

### Accept it when

- [ ] The rejected candidates are named **with their measured cost**
- [ ] The cleanup cell finds and deletes every `ds_` container
- [ ] The 400 RU/s provisioned floor is stated
- [ ] Review triggers are specific, not "review periodically"

### It will probably get wrong

- **Recommending provisioned throughput** because it is "for production". At
  this volume you are far below the 400 RU/s floor, so you would pay for
  capacity you never use. Serverless is right for the POC — and it gives you a
  real RU figure to size a provisioned tier with later, which beats a guess.
- **A cleanup cell that deletes a hardcoded list of names.** If an earlier cell
  created something the list does not know about, it survives and you keep
  paying for it. Enumerate by prefix.
- **Deleting the database outright without asking.** If your account has other
  work in it, that is somebody's afternoon.
- **Vague review triggers.** *"Review the design periodically"* triggers
  nothing. *"A new access pattern; volume beyond the tested range; partition
  skew above 35%"* is something a person can actually notice.

### The sentence that matters

Row one of your ADR is the one you cannot revisit.

> **Name the rejected candidates.** In two years somebody will ask why
> `case_id`, and *"it seemed right"* is not an answer.

You have the numbers now. Write them down.

---

# The defence — 15 minutes

Present to another pod. No notebook open.

1. What is your partition key, and what did the other two candidates cost?
   **Say the numbers.**
2. Which access pattern did you optimise for, and how do you serve the other
   two?
3. A 429 arrives on a container showing plenty of headroom. What is happening,
   and what do you say to the person who wants to provision more?
4. What does the change feed **not** capture, and what does the fix cost?
5. Your hot and cold documents — what happens if one write succeeds and the
   other fails? Are they in the same logical partition?
6. **What did Gemini get wrong that you caught** — and what did you only catch
   because the acceptance criteria told you to look?

The last question is what is being assessed.

---

## Trainer notes

**Do not release the failure lists in advance.** Print with those sections
folded, or hand out briefs and prompts first and release the rest at halfway.

**Exercise 1's iterator trap is the most valuable failure of the day**, and it
is nearly invisible. A pod that reads the RU charge before exhausting the query
iterator gets a full table of plausible numbers that are all zero or all
belong to the previous operation. Nothing errors. If you see suspiciously
round or suspiciously equal RU figures in Exercise 2, that is the cause — and
it is worth letting them present the table before you say so, because *"the
numbers looked fine"* is exactly how this reaches production.

**Watch the status distribution in Exercise 2.** If a pod's generated data is
evenly distributed across the five statuses, the hot partition in Exercise 3
never appears and they will conclude the whole concern is overstated. Check
the counts before they move on.

**Exercise 6's cleanup is not optional.** Five containers per pod, twenty-five
pods, on a provisioned account is a real bill. Walk the room.

**Three lanes.** The non-coding lane here is the one that produces the actual
decision: **ranking the access patterns by frequency**. That is not a technical
task — it needs somebody who knows how the business will use the system, and
the top of that list is the partition key.

**Budget.** About 1,200 writes and 100 queries per pod at `N_CASES=400` —
roughly 15,000 RU, pennies on serverless. But confirm the account is **Core
(SQL) API** before the session, and confirm it is serverless or has enough
provisioned throughput for twenty-five pods writing concurrently. Thirty
learners against one 400 RU/s container will throttle — which is a Day 18
lesson, but not one you want to teach by accident at 6:35pm.
