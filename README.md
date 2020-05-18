# scorch (Silent CORruption CHecker)

scorch is a tool to catalog files and their hashes to help in discovering file corruption, missing files, duplicate files, etc.

### Usage

```
usage: scorch [<options>] <instruction> [<directory>]

scorch (Silent CORruption CHecker) is a tool to catalog files, hash
digests, and other metadata to help in discovering file corruption,
missing files, duplicates, etc.

positional arguments:
  instruction:           * add: compute & store digests for found files
                         * append: compute & store digests for unhashed files
                         * backup: backs up selected database
                         * restore: restore backed up database
                         * list-backups: list database backups
                         * diff-backup: show diff between current & backup DB
                         * hashes: print available hash functions
                         * check: check stored info against files
                         * update: update metadata of changed files
                         * check+update: check and update if new
                         * cleanup: remove info of missing files
                         * delete: remove info for found files
                         * list: md5sum'ish compatible listing
                         * list-unhashed: list files not yet hashed
                         * list-missing: list files no longer on filesystem
                         * list-dups: list files w/ dup digests
                         * list-solo: list files w/ no dup digests
                         * list-failed: list files marked failed
                         * list-changed: list files marked changed
                         * in-db: show if files exist in DB
                         * found-in-db: print files found in DB
                         * notfound-in-db: print files not found in DB
  directory:             Directory or file to scan.

optional arguments:
  -d, --db=:             File to store digests and other metadata in. See
                         docs for info. (default: /var/tmp/scorch/scorch.db)
  -v, --verbose:         Make `instruction` more verbose. Actual behavior
                         depends on the instruction. Can be used multiple
                         times.
  -q, --quote:           Shell quote/escape filenames when printed.
  -r, --restrict=:       * sticky: restrict scan to files with sticky bit
                         * readonly: restrict scan to readonly files
  -f, --fnfilter=:       Restrict actions to files which match regex.
  -F, --negate-fnfilter  Negate the fnfilter regex match.
  -s, --sort=:           Sorting routine on input & output. (default: natural)
                         * random: shuffled / random
                         * natural: human-friendly sort, ascending
                         * natural-desc: human-friendly sort, descending
                         * radix: RADIX sort, ascending
                         * radix-desc: RADIX sort, descending
                         * mtime: sort by file mtime, ascending
                         * mtime-desc: sort by file mtime, descending
                         * checked: sort by last time checked, ascending
                         * checked-desc: sort by last time checked, descending
  -m, --maxactions=:     Max actions before exiting. (default: maxint)
  -M, --maxdata=:        Max bytes to process before exiting. (default: maxint)
  -b, --break-on-error:  Any error or digest mismatch will cause an exit.
  -D, --diff-fields=:    Fields to use to indicate a file has 'changed' (vs.
                         bitrot / modified) and should be rehashed.
                         Combine with ','. (default: size)
                         * size
                         * inode
                         * mtime
                         * mode
  -H, --hash=:           Hash algo. Use 'scorch hashes' get available algos.
                         (default: md5)
  -h, --help:            Print this message.

exit codes:
  *  0 : success, behavior executed, something found
  *  1 : processing error
  *  2 : error with command line arguments
  *  4 : hash mismatch
  *  8 : found
  * 16 : not found, nothing processed
```

### Database

#### Format

The file is simply CSV compressed with gzip.

```
$ # file, hash:digest, size, mode, mtime, inode, state, checked
$ zcat /var/tmp/scorch/scorch.db
/tmp/files/a,md5:d41d8cd98f00b204e9800998ecf8427e,0,33188,1546377833.3844686,123456,0,1588895022.6193066
```

#### --db argument

The `--db` argument can take more than a path.

* /tmp/test/myfiles.db : Full path. Used as is.
* /tmp/test : If /tmp/test is a directory -> /tmp/test/scorch.db
* /tmp/test/ : Force interpretation as directory -> /tmp/test/scorch.db
* /tmp/test : /tmp/test is not a directory -> /tmp/test.db
* ./test : Prepend current working directory and same as above. Any relative path with a '/'.
* test : No forward slashes -> /var/tmp/scorch/test.db

If there is no extension then `.db` will be added.


#### Backup / Restore

To simplify backing up the scorch database there is a backup command. Without a directory defined it will store the database to the same location as the database. If directories are added to the arguments then the database backup will be stored there.

```
$ scorch -v backup
/var/tmp/scorch/scorch.db.backup_2019-07-29T02:35:46Z
$ scorch -v backup /tmp
/tmp/scorch.db.backup_2019-07-29T02:36:12Z
$ scorch list-backups
/var/tmp/scorch/scorch.db.backup_2019-07-29T02:35:46Z
$ scorch list-backups /tmp
/tmp/scorch.db.backup_2019-07-29T02:36:12Z
/tmp/scorch.db.backup_2019-07-29T02:13:34Z
$ scorch restore /tmp/scorch.db.backup_2019-07-29T02:36:12Z
```


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
1/1 /tmp/files/d: md5:2b00042f7481c7b056c4b410d28f33cf

$ scorch -d /tmp/hash.db list-dups /tmp/files
md5:d41d8cd98f00b204e9800998ecf8427e /tmp/files/a /tmp/files/b /tmp/files/c

$ scorch -v -d /tmp/hash.db list-dups /tmp/files
md5:d41d8cd98f00b204e9800998ecf8427e
 - /tmp/files/a
 - /tmp/files/b
 - /tmp/files/c

$ echo foo > /tmp/files/a
$ scorch -v -d /tmp/hash.db check+update /tmp/files
1/4 /tmp/files/b: OK
2/4 /tmp/files/c: OK
3/3 /tmp/files/c: FILE CHANGED
 - size: 0B -> 4B
 - mtime: Tue Jan  1 16:23:57 2019 -> Tue Jan  1 16:24:09 2019
 - hash: d41d8cd98f00b204e9800998ecf8427e -> d3b07384d113edec49eaa6238ad5ff00
4/4 /tmp/files/d: OK

$ scorch -v -d /tmp/hash.db list /tmp/files | cut -d: -f2- | md5sum -c
/tmp/files/c: OK
/tmp/files/d: OK
/tmp/files/a: OK
/tmp/files/b: OK
```


### Automation

A typical setup would probably be initialized manually by using **add** or **append**. After it's finished creating the database a cron job can be created to check, update, append, and cleanup the database. By not placing **scorch** into verbose mode only differences or failures will be printed and the output from the job running will be emailed to the user (if setup to do so).

```
#!/bin/sh

scorch -M 100G check+update /tmp/files
scorch append /tmp/files
scorch cleanup /tmp/files
```


# Support

#### Contact / Issue submission

* github.com: https://github.com/trapexit/scorch/issues
* email: trapexit@spawn.link
* twitter: https://twitter.com/_trapexit
* reddit: https://www.reddit.com/user/trapexit
* discord: https://discord.gg/MpAr69V


#### Support development

This software is free to use and released under a very liberal license. That said if you like this software and would like to support its development donations are welcome.

* PayPal: trapexit@spawn.link
* Patreon: https://www.patreon.com/trapexit
* Bitcoin (BTC): 12CdMhEPQVmjz3SSynkAEuD5q9JmhTDCZA
* Bitcoin Cash (BCH): 1AjPqZZhu7GVEs6JFPjHmtsvmDL4euzMzp
* Ethereum (ETH): 0x09A166B11fCC127324C7fc5f1B572255b3046E94
* Litecoin (LTC): LXAsq6yc6zYU3EbcqyWtHBrH1Ypx4GjUjm
