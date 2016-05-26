# simple-locate
A (formerly?) simple shell based alternative for `locate`

These scripts are intended to be drop-in for hosts that don't have `locate` or a `locate` type package e.g. `mlocate` or `slocate`.

`locate`, `mlocate`, `slocate` and any others of their ilk use a database for historical reasons.  Back in the early 80's when `locate` was first designed, hard drive space was in short supply, and so the database format was primarily a choice made to save space.  In testing back then, the first `locate` + db combo was supposedly also faster than `grep` + plaintext.

It's 2016 now.  Hard drive space is running rampant, so we can do away with the database and use plaintext, and then that of course means we can simply use `egrep` as our frontend.  I'm very sure that other people have done this, and in `perl`, `python` or whatever else.  This is simply my attempt at it.

## slocate?  mlocate?  What?
The short version is that `slocate` tries to prevent you from getting search results for files you wouldn't have access to.  You don't want the users seeing `/root/people_to_murder.txt`, for example.  So, obviously the `s` stands for *secure*.

With `mlocate` it seeks to advance on `slocate` by *merging* differences that it finds.  In other words, it checks each directory it finds against its own db, and if the modified timestamp differs, it reindexes that directory.  At a certain scale (large filesystems, millions of files) merging gives a significant performance boost.

There was another one called `rlocate` which, if my memory serves, was a kernel module that hooked into filesystem events and essentially kept the db updated in near real-time.  Hence the `r`.

### Does simple-locate do any of that?
`simple-locate` attempts, in a rudamentary way, to replicate `slocate` and `mlocate` functionality.  It keeps the following indices:

* locate.root - an index of everything, and only root (or sufficient sudo) can search against it.
* locate.user - an index of everything that the user 'nobody' can get to.  What 'nobody' can get to, exactly, differs per system.  This is the default index for non-root users.
* locate.username (e.g. locate.rawiri) - if you run the update script as yourself, it will generate an index of files that you can access.  If you have your own index, you will search against it.  If not, you'll search against locate.user

Merging can be enabled or disabled in the update script.  As mentioned earlier, at some point of scale it will be faster.  By default, on a quiet system, it's maybe 30% slower.  This is down to a few engineering choices forcing the use of loops.

## Installation

To install, simply drop these scripts into an appropriate place in your `$PATH` e.g. `/usr/local/bin`, then obviously ensure they are executable with `chmod +x`.  Next, setup a `cron` entry to your liking for the update script.  Optionally go through the update script and double check that the variables are to your liking.  Then you could optionally manually run it.

If you don't have an existing `locate` binary, you can rename `simple-locate` to `locate`, or symlink it.  These scripts will otherwise not conflict with, and will happily live side by side with a classic flavour of `locate`.

## Configuration

There's not much to configure, but if you like you can edit the update script.  All the comments you need are there.

## Compatibility
So far this has been tested on CentOS 4.7, Centos 6.7, Ubuntu 16.04, FreeBSD 7, Solaris 9 and Solaris 11.  Updates have been applied to suit while maintaining relative portability.  I'll be interested to hear if it works elsewhere, specifically on AIX and/or HPUX.  I'm also interested in any feedback on issues etc.

For the record: FreeBSD provided some interesting challenges and solutions (e.g. `chown 0:0`), but Solaris 9 has forced the most regressions.

## Performance
### Searching
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
### Indexing
`updatedb` (`mlocate`):

```
# time updatedb

real	0m6.218s
user	0m0.428s
sys	0m2.184s
```

`simple-locate` (merging disabled)
```
# time ./update-smlocate

real	0m9.422s
user	0m6.992s
sys	0m2.056s
```

I'd be lying if I said this was a reliable test result.  Realistically it varies depending on system load.  Here's the worst test result:

```
# time ./update-smlocate

real	0m37.312s
user	0m10.288s
sys	0m11.120s
```

Which, for an overnight `cron` job isn't the end of the world anyway.

`simple-locate` (merging enabled)
```
# time ./update-smlocate

real	0m22.068s
user	0m10.672s
sys	0m4.528s
```

As we can tell, at this scale, merging is slightly punitive over our non-merging best-case, but in repeated testing it seems to stick around the 19-22s mark.
