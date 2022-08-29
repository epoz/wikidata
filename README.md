# Alternative Wikidata querying

## Introduction

For many projects, we would like to query Wikidata but the type of queries are either too complex, resulting in timeouts, or too large leading to service blocks. Can we do queries [using the raw dumps](https://www.wikidata.org/wiki/Wikidata:Database_download), without loading and duplicating the [effort of loading the system into a system like Blazegraph?](https://addshore.com/2019/10/your-own-wikidata-query-service-with-no-limits/) [(part2)](https://addshore.com/2020/10/faster-munging-for-the-wikidata-query-service-using-hadoop/)

[Others have tried](https://wiki.bitplan.com/index.php/Get_your_own_copy_of_WikiData) similar things, or [are working on related projects](https://twitter.com/ftrain/status/1559385956549099520).

There are several alternative Wikidata query services to other than the official one, like [this](https://wikidata.metaphacts.com/resource/app:Start), [this](https://qlever.cs.uni-freiburg.de/wikidata) or [this](https://wikidata.demo.openlinksw.com/sparql). But they all have their own quirks.

These are my notes on exploring some other ways of querying the data outside of a a "regular" triplestore, but using a combination of [SQLite](https://www.sqlite.org/) and [Duckdb](https://duckdb.org/).

### Constraints

- Keep the RDF data model 💪

- Work with the full data dumps 🌪

- Try not to exclude any sub-sections if possible 🍒

- Avoid extensive pre-processing if possible (but [even the official loading process](https://gerrit.wikimedia.org/r/plugins/gitiles/wikidata/query/rdf/+/refs/heads/master/tools/src/main/java/org/wikidata/query/rdf/tool/rdf/Munger.java) does it...) 🦨

- Try to go from download to queryable system in less than 24 hours. 🤪

## Preparations

The start was downloading a dump, in this case `May 12 2022 latest-all.nt.gz`, it was `190275365497` bytes. That's 177GB of compressed text in one file! A non-trivial chunk of data. Yum. This was split up into some smaller files like this: `zcat latest-all.nt.gz | split -u --lines=500000000` which resulted into 35 smaller files.

As an aside, the nice thing about using the .nt format files is that each triple is a single line. Unix tools generally are really well-suited to beavering through line-oriented data formats. If it were .ttl files they are easier to read for humans, but not as nicely self-contained per line.

Then we want to have a different way to store IRIs. In stead of having the IRIs repeated in whatever format we decide to convert or load the data, can we hash them into an integer? My first though was using a hash-function into a 32-bit integer of some kind. We do not need a cryptographically strong hash, just something that [does not give collisions](<[SipHash](https://en.wikipedia.org/wiki/SipHash)>). But after some experiments with smaller hash sizes, and in particularly using [MurmurHash](https://en.wikipedia.org/wiki/MurmurHash) it quickly became apparent that for 32-bit hashes on the Wikidata IRIs there are too many collisions too quickly, and using a 128-bit negates the whole idea.

An alternative possibility is to store **all** the IRIs in a single database table, and use the database rowids as the unique ID... SQLite to the rescue, extract all the IRIs and load them into a database that looks like:

```SQL
CREATE TABLE uris(uri);
CREATE UNIQUE INDEX uris_u ON uris(uri);
```

This results in a single (indexed) database file that has 1 950 144 749 rows, and is 334GB That's big, because of the index... More than twice as large as the original (compressed) dump. Is it worth it? Remains to be seen, but it does allow us to have quick IRI -> integer ID lookups.

And then we can extract only the structure into s,p,o tables, bu only looking at the triples which are IRIs.
Running the split using Python code, gives some stats like this:

`Mon Aug 29 09:56:31 2022 x/xas.gz 408895911 49999.5 |2.783333333333333 hours| |167 min|`
Showing the estimate for processing a chunk at around 3 hours. This means that processing all 35 files would be around 100 hours. Not exactly fast. Some were lower eg.
`Mon Aug 29 09:58:42 2022 x/xas.gz 415495845 99999.0 |1.3833333333333333 hours| |83 min|` but that is still not fast enough.
So the splitting also needs to be speeded up, trying Go.

For querying, we can load the data into Duckdb like this:

```SQL
CREATE TABLE spo AS SELECT * FROM read_csv_auto('<some_filename>', delim='\t', header=False, columns={'s':'UINTEGER', 'p':'UINTEGER', 'o':'UINTEGER'});
```
