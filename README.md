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

## Usage
`simple-locate` is basically a veneer thin wrapper for `egrep`, so `egrep` syntax, rather than `locate` applies.  All options available to `egrep` apply to `simple-locate` too, so for example you can do a case-insensitive search using `-i`.

## Compatibility
So far this has been tested on CentOS 4.7, Centos 6.7, Ubuntu 16.04, FreeBSD 7, Solaris 9 and Solaris 11.  Updates have been applied to suit while maintaining relative portability.  I'll be interested to hear if it works elsewhere, specifically on AIX and/or HPUX.  I'm also interested in any feedback on issues etc.

For the record: FreeBSD provided some interesting challenges and solutions (e.g. `chown 0:0`), but Solaris 9 has forced the most regressions.

### A note about 'find' compatibility
In order to try and eke out some performance, `simple-locate` directs `find` to ignore certain filesystem types and paths.  A full command might look something like this:

```
find / \( -fstype afs -o -fstype autofs -o -fstype coda -o -fstype ctfs -o -fstype dev -o -fstype devfs -o -fstype devpts -o -fstype fd -o -fstype ftpfs -o -fstype hsfs -o -fstype iso9660 -o -fstype lofs -o -fstype mfs -o -fstype mntfs -o -fstype namefs -o -fstype ncpfs -o -fstype nfs -o -fstype NFS -o -fstype objfs -o -fstype pcfs -o -fstype proc -o -fstype shfs -o -fstype smbfs -o -fstype sysfs -o -fstype tmpfs -o -fstype uvfs -o -path /cdrom -o -path /media -o -path /proc -o -path /tmp -o -path /var/tmp \) -prune -o -type d -mtime -1 -print
```

Unfortunately, not all versions of `find` support `-path`, primarily Solaris and AIX.  I have tested various ways that claim to be portable workarounds and here's what I found:

`-o -type d -name '/path'`

This one upsets GNU `find` which prints an error.  Slashes might not be interpreted, which means you can't exclude e.g. `/var/tmp`, so instead you have to exclude all directories named `tmp`, which might not be what you want.  You lose flexibility and risk having unintended matches, which are then excluded.

`-o -inum $inum_of_/path`

This one had a lot of promise.  It is fast and portable but it's unpredictable.  It works on the premise of excluding by inode number, which can be easily found e.g. `ls -di /some/path | awk '{print $1}'`.  Unfortunately, in testing on a Solaris 9 host, it showed that multiple filesystems shared an inode, and so we had unintended matches again.

```
sol9test{root}: ls -di /proc
         2 /proc
sol9test{root}: ls -di /opt
         2 /opt
sol9test{root}: ls -di /home
         2 /home
```

See?  We want to exclude `/proc`, it has an inode number of `2`, but so does `/opt` and `/home` at least, which we do want to index.  So let's move on...

`-o -exec test "{}" = "/path" \;`      

Last up is this one.  It is brutally slow compared to all other options, but it's reliable.  Do NOT chain it like this:

`-o -exec test "{}" = "/path" -o -exec test "{}" = "/some/other/path" \;`

Instead, chain it like this:

`-o -exec test "{}" = "/path" -o "{}" = "/some/other/path" \;`

To give you an example, in a test using the full chaining method, on my SSD, finding directories modified in the last 24 hours, we get:

```
real	12m11.409s
user	0m16.300s
sys	1m52.700s
```

Using the more compact chaining method, however:

```
real	1m45.280s
user	0m5.744s
sys	0m19.556s
```

These results are repeatable, no matter which order etc you run the commands in.  They will obviously differ per system and day.

So ultimately, `simple-locate` will test whether `find` can use `-path`, and if so, use it.  It will otherwise failover to the slower `-exec test` method.  If, on Solaris or AIX, you can provide GNU `find`, then that will be to your benefit.  Not just for `simple-locate`.

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

## Known Problems
Most problems have been dealt with, the major remaining one that I'm aware of is duplication in merging.  For example, let's say that the update script finds the following directories as having been modified in the last 24 hours:

* `/home/userA`
* `/home/userA/.local/someApp/cache`
* `/home/userA/.secret/stashedfiles/`

So the update script will filter `^/home/userA` out of the index files, which covers the other two directories.  It will then reindex `/home/userA`, which includes the other two directories.  But in due course it will reindex the other two directories as well.

I currently have no great incentive to fix this, for two reasons:

* Indexing a directory is pretty fast.  While it's inelegant, there's no great performance loss with the reindexing.  The main performance hit is in generating the list of directories modified in the last 24 hours
* The searching script runs `sort` and `uniq` across its output anyway, which removes duplicates.

This is a "maybe on a rainy day" fix.
