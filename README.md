# dw-query-digest

Alternative to `pt-query-digest`.

[![Build Status](https://travis-ci.org/devops-works/dw-query-digest.svg?branch=master)](https://travis-ci.org/devops-works/dw-query-digest)
[![Go Report Card](https://goreportcard.com/badge/github.com/devops-works/dw-query-digest)](https://goreportcard.com/report/github.com/devops-works/dw-query-digest)
[![SonarQube](https://sonarcloud.io/api/project_badges/measure?project=dw-query-digest&metric=alert_status)](https://sonarcloud.io/dashboard?id=dw-query-digest)
[![MIT Licence](https://badges.frapsoft.com/os/mit/mit.svg?v=103)](https://opensource.org/licenses/mit-license.php)

## Purpose

`dw-query-digest` reads slow query logs files and extracts a number of
statistics. It can list top queries sorted by specific metrics.

It is reasonably fast and can process ~450k slow-log lines per second.

A 470MB slow query log containing 10M lines took approx 150s with `pt-query-digest`
and approx 23s with `dw-query-digest` (6x faster).

`dw-query-digest` normalizes SQL queries so identical queries can be aggregated
in a report.

## Usage

### Binary

`bin/dw-query-digest [options] <file>`

### Source

Requires Go 1.11+.

```bash
make
bin/dw-query-digest [options] <file>
```

### Docker

To run using [Docker](https://hub.docker.com/r/devopsworks/dw-query-digest):

```bash
cd where_your_slow_query_log_is
docker run -v $(pwd):/data devopsworks/dw-query-digest /data/slow-query.log
```

### Options

- `--debug`: show debugging information; this is very verbose, and meant for debugging
- `--progress`: display a progress bar
- `--output <fmt>`: produce report using _fmt_ output plugin (default: `terminal`; see "Outputs" below)
- `--list-outputs`: list possible output plugins
- `--quiet`: display only the report (no log)
- `--reverse`: reverse sort (i.e. lowest first)
- `--sort <string>`: Sort key
  - `time` (default): sort by cumulative execution time
  - `count`: sort by query count
  - `bytes`: sort by query bytes size
  - `lock[time]`: sort by lock time (`lock` & `locktime` are synonyms)
  - `[rows]sent`: sort by rows sent (`sent` and `rowssent` are synonyms)
  - `[rows]examined`: sort by rows examined
  - `[rows]affected`: sort by rows affected
- `--top <int>`: Top queries to display (default 20)
- `--nocache`: Disables cache (writing & reading)
- `--version`: Show version & exit

## Outputs

The default output is "terminal".

### `terminal`

Simple terminal output, designed to be read by humans.

### `greppable`

CSV format output designed to be used in combination with `grep`.
In this format, the two first lines are:

- meta information (server version, duration, etc...)
- columns header

Both lines start with `#`, so if you only want queries, you can filter them
using `dw-query-digest ... | grep -v '^#'`.

Column headers are prefixed by a number. You can directly use this number with
`cut` if you only need a specific column. For instance, if I just want the max
time for a query, and since the column header is `16_Max(s)`, I can use:

```bash
$ dw-query-digest ... | grep -v '^#' | cut -f16 -d';'
2.378111
0.243685
0.335063
0.715469
```

### `json`

Pretty-printed JSON output containing general information in the `meta` key and
queries informations in the `stats` key. Note that the keys layout are subject
to change.

### `null`

Does not output anything, everything goes to `/dev/null`. This can be used for
benchmarking, to prime cache, and whatnot.

## Cache

When run against a file, `dw-query-digest` will try to find a cache file having the same name and ending with `.cache`. For instance, if you invoke:

```bash
dw-query-digest -top 1 dbfoo-slow.log
```

then `dw-query-digest` will try to find `dbfoo-slow.log.cache`. If the cache
file is found, the original file is not touched and results are read from the
cache. This lets you rerun the command if needed without having to re-analyze
the whole original file.

Since all results are cached, you can use different paramaters. For instance,
`--top`, `--sort` or `output` can be different and will (hopefully) work.

If you don't want to read or write from/to the cache at all, you can use the
``--nocache` option. You can also remove the file anytime.

If the analyzed file is newer than it's cache, the cache will not be used.

Cache format is not guaranteed to work between different versions.

## Caveats

Some corners have been cut regarding query normalization. So YMMV regarding
aggregations.

## Contributing

If you spot something missing, or have a slow query log that is not parsed
correctly, please open an issue and attach the log file.

Comments, criticisms, issues & pull requests welcome.

## Roadmap

- [ ] cache
- [ ] `tail -f` reading (disable linecount !) with periodic reporting (in a TUI ?)
- [ ] `web` live output
- [ ] `pt-query-digest` output ?
- [ ] UDP json streamed output (no stats) for filebeat/logstash/graylog ?

## Licence

MIT
