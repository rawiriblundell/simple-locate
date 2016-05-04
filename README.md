# simple-locate
A simple shell based alternative for 'locate'

These scripts are intended to be drop-in for hosts that don't have `locate` or a `locate` type package e.g. `mlocate` or `slocate`.

`locate`, `mlocate`, `slocate` and any others of their ilk use a database for historical reasons.  Back in the early 80's when `locate` was first designed, hard drive space was in short supply, and so the database format was primarily a choice made to save space.  In testing back then, the first `locate` + db combo was supposedly also faster than `grep` + plaintext.

It's 2016 now.  Hard drive space is running rampant, so we can do away with the database and use plaintext, and then that of course means we can simply use `egrep` as our frontend.  I'm very sure that other people have done this, and in `perl`, `python` or whatever else.  This is simply my attempt at it.

## Installation

To install, simply drop these scripts into an appropriate place in your `$PATH` e.g. `/usr/local/bin`, then obviously ensure they are executable with `chmod +x`.  Next, setup a `cron` entry to your liking for the update script.  Then you could optionally manually run it.

If you don't have an existing `locate` binary, you can rename `simple-locate` to `locate`, or symlink it.
