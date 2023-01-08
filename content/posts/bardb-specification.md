---
title: "BarDB specification"
date: 2023-01-08T10:08:00+01:00
draft: false
---
# BarDB specification

Time series of low resolution charts can quickly consume a large amount of storage.
The goal of BarDB is to store market data structured as OHLCV candles as efficiently as possible. 
This document aims to explain an implemenation of a database that is optimized to solve this specific problem.

To keep in mind if we should, instead of being to busy trying to figure out if we could, it is important to compare this solution against other implementations with a relational database and a schemaless database.

## Binary storage format

The candles are written as a binary file to disk. This section documents the layout of the binary storage format.

### Symbol File Table (SFT)

The symbol file table is a high level index table of the data that is stored in the database. 

If data is requested from the database, the symbol file table (SFT) is used to determine in which blocks the requested data is located.

This symbol file table is stored as a B-tree where the key is the time. Each symbol and resolution has their own B-tree. 

The symbol file table is stored in `bardb.sft`. Ideally the database runs as a server and the symbol file table is present in memory at all times.

### Block

Data is stored in so called blocks. The first type is the time $t$, which is Unix time (UTC) in seconds, and the second type is the resolution, or $t_{inc}$ (time increment). The third type is the size $s$ of the block. 

 * The total size in bytes of a block is $20s + 16$.
 * The offset of each bar is $t + k\cdot t_{inc}$, where $k \in \mathbb N$.
 * To get the index of a bar given a Unix time $T$, first get the remainder $r = T - t$, and then the index is $i = \lfloor r / t_{inc} \rfloor $.

```
i64 time           8 bytes
u32 resolution     4 bytes 
u32 size           4 bytes
bar[] bars         20 bytes per bar
```

In theory, there is no maximum size that a block can have. The only condition is that the bars are stored contiguously because the time and resolution is used to determine the offset of a bar.  

Perhaps this is not feasible due to file system limitations. In this case a block can be splitted it if a maximum file size is reached.

A block is stored with the following naming convention: `SYMBOL.RESOLUTION.bin`.

### Bar

A bar represents one OHLCV candle. The data of a candle is stored with the following types. The total size of one bar is 20 bytes.

```
f32 open           4 bytes
f32 high           4 bytes
f32 low            4 bytes
f32 close          4 bytes
f32 volume         4 bytes
```

The time which a bar represents doesn't need to be stored next to the bar itself, because this is determined by the offset at which the bar is stored.

### File size

Using 20 bytes for each candle, the following table can be constructed. A year of minute candles consumes 10.02Mb of storage.

<p>
 
|        | candles | bytes (bars) | kilobytes (bars) | megabytes (bars) |
| ------ | ------- | ------------ | ---------------- | ---------------- |
| year   | 1       | 20           | 0.01953125       | 1.90735E-05      |
| month  | 12      | 240          | 0.234375         | 0.000228882      |
| week   | 52      | 1040         | 1.015625         | 0.000991821      |
| day    | 365     | 7300         | 7.12890625       | 0.006961823      |
| hour   | 8760    | 175200       | 171.09375        | 0.16708374       |
| minute | 525600  | 10512000     | 10265.625        | 10.02502441      |

</p>

## Bar Query Language (BQL)

The interface with BarDB is specified with the Bar Query Language (BQL).
This query language is a subset of SQL, but adapted to the needs of BarDB.

### Creating a symbol

A symbol can be created with:

```sql
CREATE SYMBOL <SYMBOL_NAME:EXCHANGE>
WITH RESOLUTION 3600 -- hourly
```

Note that if a symbol is tracked on multiple exchanges, the symbol name should include the name of the exchange.

### Dropping a symbol

A symbol can be dropped from the database with

```sql
DROP SYMBOL <SYMBOL_NAME:EXCHANGE>
```

### Inserting bars

Bars can be inserted with the `INSERT` command, like so

```sql
INSERT INTO <SYMBOL_NAME:EXCHANGE>
WITH RESOLUTION 3600
VALUES 
    (T, O, H, L, C, V),
    (T, O, H, L, C, V),
    (T, O, H, L, C, V),
    (T, O, H, L, C, V),
    (T, O, H, L, C, V),
    (T, O, H, L, C, V);
```

### Retrieving bars

Bars can be retrieved from the database like so

```sql
SELECT FROM <SYMBOL_NAME:EXCHANGE>
WITH RESOLUTION 3600
WHERE T BETWEEN T_START AND T_END
```

### Updating bars

It is not possible to update bars in the first version of BarDB.

### Deleting bars

Likewise, bars can be deleted with the `DELETE`:

```sql
DELETE FROM <SYMBOL_NAME:EXCHANGE>
WITH RESOLUTION 3600
WHERE T BETWEEN T_START AND T_END
```

## Bucketization of a ticker stream

Suppose we get a continuous stream of buy or sells as a tuple $(t, p, v)$ where $t$ is the time, and $p$ the price, and $v$ the volume.

It would be _very convenient_ if this stream can be inserted into the database, and that the database bucketizes this into the correct resolution, e.g. that the database calculates the bars based on the tuple stream.

Although it is very convenient, this shouldn't be in the scope of the project for the first version.

## Performance

Compare performance of this solution against MS-SQL and NoSQL databases.

## Proof of concept

During the development of a proof of concept of BarDB, the following key question should be answered:

  1. Is it worthwile to develop BarDB as a solution? 

An answer can be given based on the following sub-questions:
  
  1. If we add a `time` column to the bar, and store it in a conventional SQL database, is this better or worse than this solution?
  2. Why is a NoSQL database better suited to store this type of data versus a relational database? Why?
  3. How does BarDB perform versus a relational database and schemaless database?

## Footnotes

 * The type `i64` is chosen as the type to represent the time. One could argue that an unsigned type can be used, but with this it is not possible to represent values before the Unix epoch naturally. Because of this, the `i64` type has been chosen.
 * It is only possible to create a symbol with a name and a resolution. One could argue that data can be collected for different exchanges as well, but this should be added to the symbol name instead for simplicity.