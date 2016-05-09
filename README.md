# simple-locate
A simple shell based alternative for 'locate'

These scripts are intended to be drop-in for hosts that don't have `locate` or a `locate` type package e.g. `mlocate` or `slocate`.

`locate`, `mlocate`, `slocate` and any others of their ilk use a database for historical reasons.  Back in the early 80's when `locate` was first designed, hard drive space was in short supply, and so the database format was primarily a choice made to save space.  In testing back then, the first `locate` + db combo was supposedly also faster than `grep` + plaintext.

It's 2016 now.  Hard drive space is running rampant, so we can do away with the database and use plaintext, and then that of course means we can simply use `egrep` as our frontend.  I'm very sure that other people have done this, and in `perl`, `python` or whatever else.  This is simply my attempt at it.

## Installation

To install, simply drop these scripts into an appropriate place in your `$PATH` e.g. `/usr/local/bin`, then obviously ensure they are executable with `chmod +x`.  Next, setup a `cron` entry to your liking for the update script.  Then you could optionally manually run it.

If you don't have an existing `locate` binary, you can rename `simple-locate` to `locate`, or symlink it.  These scripts will otherwise not conflict with, and will happily live side by side with a classic flavour of `locate`.

## Configuration

There's not much to configure, but if you like you can edit the update script.  All the comments you need are there.

## Compatibility
So far this has been tested on CentOS 4.7, Centos 6.7, Ubuntu 16.04, FreeBSD 7, Solaris 9 and Solaris 11.  Updates have been applied to suit while maintaining relative portability.  I'll be interested to hear if it works elsewhere, specifically on AIX and/or HPUX.  I'm also interested in any feedback on issues etc.

## Performance
On my workstation, which has an SSD, let's test `find`:

```
# time find / -type d -name "bin"
...
real	0m16.184s
user	0m2.224s
sys	0m5.888s
```

Followed by `locate` (i.e. `mlocate`):
```
# time locate -r '.*/bin$'
...
real	0m9.778s
user	0m9.656s
sys	0m0.032s
```

Followed by `simple-locate`:
```
# time ./smlocate /bin$
...
real	0m0.074s
user	0m0.028s
sys	0m0.028s
```
