# scorch (Silent CORruption CHecker)

scorch is a tool to catalog files and their hashes to help in discovering file corruption, missing files, duplicate files, etc.

### Usage
```
usage: scorch [<options>] <instruction> [<directory>]

scorch (Silent CORruption CHecker) is a tool to catalog files and hashes
to help in discovering file corruption, missing files, duplicates, etc.

positional arguments:
  instruction:             * add: compute and store hashes for all found files
                           * append: compute and store for newly found files
                           * check+update: check and update if new
                           * check: check stored hashes against files
                           * cleanup: remove hashes of missing files
                           * delete: remove hashes for found files
                           * list-dups: list files w/ dup hashes
                           * list-missing: list files no longer on filesystem
                           * list-solo: list files w/ no dup hashes
                           * list-unhashed: list files not yet hashed
                           * list: md5sum compatible listing
                           * in-db: show if hashed files exist in DB
                           * found-in-db: print files found in DB
                           * notfound-in-db: print files not found in DB
  directory:               Directory or file to scan

optional arguments:
  -d, --db=:               File to store hashes and other metadata in.
                           (default: /var/tmp/scorch/scorch.db)
  -v, --verbose:           Make `instruction` more verbose. Actual behavior
                           depends on the instruction. Can be used multiple
                           times.
  -q, --quote:             Shell quote/escape filenames when printed.
  -r, --restrict=:         * sticky: restrict scan to files with sticky bit
                           * readonly: restrict scan to readonly files
  -f, --fnfilter=:         Restrict actions to files which match regex
  -s, --sort=:             Sorting routine on input & output (default: natural)
                           * random: shuffled / random
                           * natural: human-friendly sort, ascending
                           * reverse-natural: human-friendly sort, descending
                           * radix: RADIX sort, ascending
                           * reverse-radix: RADIX sort, descending
                           * time: sort by file mtime, ascending
                           * reverse-time: sort by file mtime, descending
  -m, --maxactions=:       Max actions to take before exiting (default: maxint)
  -M, --maxdata=:          Max bytes to process before exiting (default: maxint)
  -b, --break-on-error:    Any error or hash failure will exit
  -h, --help:              Print this message
```

### Database

#### Format

The file is simply CSV compressed with gzip.

```
$ # file, md5sum, size, mode, mtime
$ zcat /var/tmp/scorch/scorch.db
/tmp/files/a,d41d8cd98f00b204e9800998ecf8427e,0,33188,1546377833.3844686
```

#### --db argument

The `--db` argument is takes more than a path.

* /tmp/test/myfiles.db : Full path. Used as is.
* /tmp/test : If /tmp/test is a directory -> /tmp/test/scorch.db
* /tmp/test/ : Force interpretation as directory -> /tmp/test/scorch.db
* /tmp/test : /tmp/test is not a directory -> /tmp/test.db
* ./test : Prepend current working directory and same as above. Any relative path with a '/'.
* test : No forward slashes -> /var/tmp/scorch/test.db

If there is no extension then `.db` will be added.

#### Upgrade

If you're using an older version of scorch with the default database in `/var/tmp/scorch.db` just copy/move the file to `/var/tmp/scorch/scorch.db`. The old format was not compressed but scorch will handle reading it uncompressed and compressing it on write.

### Example

```
$ ls -lh /tmp/files
total 0
-rw-rw-r-- 1 nobody nogroup 0 May  3 16:30 a
-rw-rw-r-- 1 nobody nogroup 0 May  3 16:30 b
-rw-rw-r-- 1 nobody nogroup 0 May  3 16:30 c

$ scorch -v -d /tmp/hash.db add /tmp/files
1/3 /tmp/files/c: d41d8cd98f00b204e9800998ecf8427e
2/3 /tmp/files/a: d41d8cd98f00b204e9800998ecf8427e
3/3 /tmp/files/b: d41d8cd98f00b204e9800998ecf8427e

$ scorch -v -d /tmp/hash.db check /tmp/files
1/3 /tmp/files/a: OK
2/3 /tmp/files/b: OK
3/3 /tmp/files/c: OK

$ echo asdf > /tmp/files/d

$ scorch -v -d /tmp/hash.db list-unhashed /tmp/files
/tmp/files/d

$ scorch -v -d /tmp/hash.db append /tmp/files
1/1 /tmp/files/d: 2b00042f7481c7b056c4b410d28f33cf

$ scorch -v -d /tmp/hash.db list-dups /tmp/files
d41d8cd98f00b204e9800998ecf8427e /tmp/files/a /tmp/files/b /tmp/files/c

$ echo foo > /tmp/files/a
$ scorch -v -d /tmp/hash.db check+update /tmp/files
1/4 /tmp/files/b: OK
2/4 /tmp/files/c: OK
3/3 /tmp/files/c: FILE CHANGED
 - size: 0B -> 4B
 - mtime: Tue Jan  1 16:23:57 2019 -> Tue Jan  1 16:24:09 2019
 - hash: d41d8cd98f00b204e9800998ecf8427e -> d3b07384d113edec49eaa6238ad5ff00
4/4 /tmp/files/d: OK

$ scorch -v -d /tmp/hash.db list /tmp/files | md5sum -c
/tmp/files/c: OK
/tmp/files/d: OK
/tmp/files/a: OK
/tmp/files/b: OK
```

### Automation

A typical setup would probably be initialized manually by using **add** or **append**. After it's finished creating the database a cron job can be created to check, update, append, and cleanup the database. By not placing **scorch** into verbose mode only differences or failures will be printed and the output from the job running will be emailed to the user (if setup to do so).

```
#!/bin/sh

scorch check+update /tmp/files
scorch append /tmp/files
scorch cleanup /tmp/files
```

# Support

#### Contact / Issue submission
* github.com: https://github.com/trapexit/scorch/issues
* email: trapexit@spawn.link
* twitter: https://twitter.com/_trapexit

#### Support development

This software is free to use and released under a very liberal license. That said if you like this software and would like to support its development donations are welcome.

* PayPal: trapexit@spawn.link
* Patreon: https://www.patreon.com/trapexit
* Bitcoin (BTC): 12CdMhEPQVmjz3SSynkAEuD5q9JmhTDCZA
* Bitcoin Cash (BCH): 1AjPqZZhu7GVEs6JFPjHmtsvmDL4euzMzp
* Ethereum (ETH): 0x09A166B11fCC127324C7fc5f1B572255b3046E94
* Litecoin (LTC): LXAsq6yc6zYU3EbcqyWtHBrH1Ypx4GjUjm
