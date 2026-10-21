+++
title = "Paganel: migrations with a receipt"
description = "Data migration with a receipt you can verify later."
date = 2026-10-21
[taxonomies]
tags = ["data-migration", "data-integrity", "postgres", "mysql", "rust"]
+++

Paganel is a single-binary tool that moves data and schema between systems,
then proves the destination still matches what it wrote, down to the row.
It's open source (AGPL) and written in Rust.

![Terminal recording: pag verify reports both tables matching, one row in the actor table is edited by hand with psql, and a second pag verify reports a mismatch pointing at actor_id=5 as the changed row.](demo.gif)

## Where it came from

Paganel comes from an old work problem: I had a large schema migration with
lots of transformations and several environments to check. One of the main
requirements was to validate whether the data had moved correctly, and I was
not allowed to use any SaaS tools because it was forbidden for data to leave
our network. Since I couldn't find any air-gapped tool that could do both the
transformation and the verification (basic row counts aren't enough for
transformed data), I had to write custom scripts for the entire process.

Ever since then, the idea of automating complex migrations has stuck in my
head, which led me to start building Paganel a year and a half ago.

## One file per migration

In Paganel, a migration is just a single declarative file. You specify where
the data comes from and where it's going, and the rest goes in the same file:
joins, filters, computed fields, and validation rules.

```hcl
connection "source" {
  driver = "mysql"
  url    = env("MYSQL_URL")
}

connection "dest" {
  driver = "postgres"
  url    = env("POSTGRES_URL")
}

pipeline "customers" {
  from { connection = connection.source table = "customers" }
  to   { connection = connection.dest   table = "customers" mode = "replace" }

  where "active" {
    customers.deleted_at is null
  }

  select {
    id    = customers.id
    name  = customers.name
    email = lower(trim(customers.email))
  }
}
```

Save it as `customers.ppl` and run:

```sh
pag plan   -c customers.ppl              # schema, DDL, estimates, sample rows
pag apply  -c customers.ppl --integrity  # migrate and write the receipt
pag verify -c customers.ppl              # check the destination against the receipt
```

If you are missing a specific source, sink, transform, or filter, you can
easily extend Paganel by writing your own WASM plugin in Rust or JS. These
execute inside a strict sandbox, so they don't have any filesystem or network
access unless you allow it in the migration config.

I also wanted to make large migrations simple, so before running a real
migration, you can run `pag plan` to review the output schema, the DDL, and
the time and memory estimates. You can also use the plan to see a sample of
real rows. They get processed through the whole pipeline, which gives you
absolute clarity on what will actually be inserted.

## The receipt

One of the biggest challenges isn't just doing the migration, but verifying
that everything actually moved correctly. For that, there is the
`--integrity` flag. Pass it, and Paganel hashes the rows on the fly as they
migrate, storing those hashes on disk, keyed by primary key. After the run,
it uses them to spit out a 32-byte root hash for the table. Together with the
row hashes, this gives you a complete "migration receipt."

Minutes or weeks later, you can run `pag verify`, and it re-reads the
destination, recomputes the hashes and compares them with the receipt,
without connecting to the source. When the root doesn't match, you know
something changed, and then verify goes through the row hashes and tells you
which rows it was.

```bash
$ pag apply  -c examples/configs/merkle.ppl -e .env --integrity
$ pag verify -c examples/configs/merkle.ppl -e .env
✓ migrate_actor/actor - match (200 rows, root 2424b83ccd0a9852426c011ad58bd8af43696f295b03ed15c77dfdd9115b3553, 21ms)

# hand-edit one row on the destination, then verify again
$ psql "$DEST_URL" -c "UPDATE actor SET last_name = 'TAMPERED' WHERE actor_id = 5"
$ pag verify -c examples/configs/merkle.ppl -e .env
✗ migrate_actor/actor - MISMATCH (0 missing, 1 changed, 0 extra; 200 rows expected, 200 found; 19ms)
  expected root 2424b83ccd0a9852426c011ad58bd8af43696f295b03ed15c77dfdd9115b3553
  actual   root 014c68b47d327dd3779c68b06476294d9e8db65d70d339f06d2b4003d1dc240a
  actor_id=5 - changed: expected e0e4fb52b84373ee actual 193e5586b0ed90af
```

It exits non-zero on a mismatch, so you can run it in cron or CI.

It tells you that the destination still contains what Paganel wrote, but it
can't tell you whether the transformation itself was correct.

## Speed

Another huge factor is migration speed. I created a small benchmark to
measure Paganel's speed and tested it on a 100M-row migration from MySQL to
PostgreSQL (with the databases on separate servers). With the default config,
which is a single stream, I got about 390k rows a second. Splitting the table
into four parallel streams got it to almost 940k. The number of streams is
configurable for tables with an integer primary key. Enabling the
`--integrity` flag does add some overhead, dropping those numbers to 340k
and 670k, respectively.

## Rough edges

Paganel is still pre-v1.0, so there are some rough edges to be aware of:

- **No CDC yet:** Currently, it only handles batch processing. Continuous
  sync with verification is the ultimate goal, but nailing the batch engine
  comes first.
- **Limited sources:** Right now, Paganel natively supports MySQL,
  PostgreSQL and CSV.
- **API changes:** The project is under heavy development, meaning the
  config syntax might change slightly from release to release.
- **One-man team:** I'm developing this solo. You will likely run into edge
  cases I haven't caught yet.
- **Platform testing:** The full integration suite and all my own manual
  testing run on Linux. macOS and Windows get pre-built binaries too, and
  every release runs a real migration and verify on each of them, but that's
  much less coverage than Linux gets.

## What's next

A receipt checks a copy once. The next step is checking it again and again,
because copies drift after they're made. Here's what I'm working on, none
of it released yet:

- **Drift checks:** re-run verify on a schedule and send a daily report of
  what stopped matching.
- **Comparing two live databases:** point Paganel at a source and a
  destination it didn't copy, read-only on both sides, and get the rows
  that differ.
- **Repair:** re-copy only the rows that differ, then verify again.

After that, Postgres CDC, so the copy stays in sync continuously and is
checked the same way.

## Try it

The quickstart in the README spins up some sample databases in Docker for
you: <https://github.com/stanstork/paganel>