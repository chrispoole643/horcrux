# Horcrux: A Wrapper for Duplicity

[Duplicity] is a backup program that allows you to store encrypted backups offsite
while also allowing for incremental updates.

I take [the _Voldemort_ approach to backups][backup strategy], backing up mostly
the same stuff to several different places, hence using similar options with
[Duplicity]. To make this use case easier, I wrote Horcrux, a wrapper for
Duplicity's most commonly-used options. It also includes functionality to check
a restored backup to periodically ensure your backups are working.

After a simple setup, you'll be able to run

    horcrux auto offsite1

to backup selected files and directories as part of your _offsite1_ backup. If
you want to add a new backup, the setup is minimal, and you can run

    horcrux auto offsite2

Both backups keep separate caches and use separate log files. They can backup
different files and directories, and to different locations.


## Installation

Download horcrux, and place it in your `PATH` somewhere, usually `~/bin`.


## First run

Run `horcrux` with no arguments. It'll tell you that it's created a file,
`~/.horcrux/horcrux.conf`. This is the global configuration file, which'll be
consulted for each backup. Open this file in your favourite text editor and edit
the options (perhaps only the source directory and the encryption key ID).

Note that if you add a sign key (a good idea) and you use a subkey, you should
give the subkey's ID, not the ID of your main key. If you give the main key, the
backup will still work (since gpg will choose your signing key automatically),
but upon restore Duplicity will complain that the backup files have been signed
with a different key.

The source directory is the directory containing the files and directories to be
backed up. Each backup will then consult an exclude file, which will tell
Duplicity which files should be ignored, and which should be part of the backup.


## Basic usage

Horcrux is run from the command line with the following syntax:

    horcrux [options] operation backup_name [restore path]

`options` and `restore path` are optional, but you must always tell Horcrux what
you want it to do (the operation), and the name of the backup. The backup name
can use most characters (letters, numbers, etc.) but _should not use hyphens_.


### Configuration setup

Each backup run by Horcrux has a name and an exclude file. For example, let's
create a backup called _offsite1_. Create the following two files in the
`~/.horcrux` directory:

- _offsite1-config_
- _offsite1-exclude_

The config file should contain at least one variable: the destination path. If
you're backing up to a directory on a remote server somewhere, _offsite1-config_
might look like this:

    destination_path="rsync://username@your_server//home/username/backup/"

The URL follows the definitions allowed as described in [Duplicity's man
page][man]. You can also override variables set in the global configuration file; this
is useful to be able to change the source directory or encryption key for one
specific backup, for example. 

The _offsite1-exclude_ file also follows the syntax explains in Duplicity's man
page. Assuming the source directory is `$HOME/`, it looks something like this:

    **/.DS_Store
    + /Users/username/.emacs
    + /Users/username/emacs
    + /Users/username/bin
    + /Users/username/.git*
    + /Users/username/.ssh
    + /Users/username/.zsh*
    + /Users/username/.horcrux
    + /Users/username/Documents/**/*.md
    + /Users/username/Documents/**/*.css
    **/*

Duplicity likes the exclude syntax to use absolute path names, hence the
redundancy here. Items starting with a `+` are to be included. Those without are
to be excluded. The above lines will cause Duplicity to ignore any _.DS_Store_
files (used by Mac OS X to store icon information in the Finder), then include
several files and directories. Any `.css` files in any directory within my
_Documents_ directory will get backed up, as will the _.horcrux_ directory, my
_emacs_ directory, and several files and directories starting with _.zsh_.

Duplicity stores a local cache of changes in `~/.cache/duplicity` by
default. If your source directory is the root of your home directory (the
Horcrux default), then make sure you exclude this directory. I add the following
line to some of my exclude files:

    **/*[Cc]ache*

This will ensure that Duplicity excludes files and directories with the word
_cache_ in it, which also includes `~/.cache`.


### Your first backup

Now you should be ready to make your first backup. Duplicity first produces a
full backup, and from then on can make incremental backups (i.e., only uploading
the changed bits, to save time and space). To do this automatically, run

    horcrux auto offsite1

All output sent to the screen will also be put in a log file in `~/.horcrux`
with the appropriate name. It'll be compressed with `bzip2`, or `pbzip2` if
available.

If at any point you want to check the names of options to use, you can run
`horcrux help` or `horcrux -h`.


#### Zero configuration for local drives

You might have a few external USB drives to use for backups, each of which have
the directory _backup_ inside them, so for each one you'd add

    destination_path="file:///Volumes/drive_name/backup/"

or

    destination_path="file:///media/drive_name/backup/"

to its configuration file[^zero] (assuming you're using the same name for both
the drive and the backup). In this case, there's no need to have a configuration
file at all. Horcrux will automatically look for such a directory, and if it
exists, will use this as the destination path[^dest].

For example, if you connect a drive called _offsite2_, and add a directory
_backup_, you can then add an exclude file _offsite2-exclude_ to your Horcrux
directory, and the backup files will automatically be sent to
`/Volumes/offsite2/backup`.


## Operations

Horcrux supports the following operations:

`auto`
: Full backup if no filesets or last full backup is old (default 360 days), else
  incremental.

`check`
: Check md5 hashes of files in a restored fileset against local fileset.
: Requires [md5deep].

`clean`
: Clean local caches, removing files no longer needed.
: Maps to `duplicity cleanup --extra-clean --force`.

`full`
: Perform full backup.

`inc`
: Perform incremental backup.

`list`
: List files backed up in fileset.
: Maps to `duplicity list-current-files`.

`remove`
: Remove filesets, leaving latest n (default 3) full+inc filesets.
: Maps to `duplicity remove-all-but-n-full n --force`.

`restore`
: Restore certain files/directories or complete filesets.
: Maps to `duplicity restore`.

`status`
: Check collection status on fileset.
: Maps to `duplicity collection-status`.

`verify`
: Verify a backup. Not as thorough as `check`.
: Maps to `duplicity verify`.

Running `full` will force a full backup; if `auto` is passed, Duplicity will
choose this method if it doesn't find a "recent" fileset at the backup
destination[^recent], otherwise it'll run an incremental backup.


### Restoring a backup

You can restore certain files from your backups, or certain directories, or
the entire fileset. To do the latter, create a directory where you want to
restore to, and then run

    horcrux restore offsite1 /Some/restore/directory

The backup _offsite1_ will then be restored to `/Some/restore/directory`. You'll
be prompted for your GPG passphrase by Duplicity.

If you want to restore a certain file or directory, use the `-f` option:

    horcrux -f somefile.txt restore offsite1 ~/restore_directory/somefile.txt

The file `~/restore_directory/somefile.txt` will be written with the contents of
`somefile.txt` from your backup. This file named with `-f` is relative to the
source directory of the backup.


#### Restoring from a certain time

You can use the `-t` option to restore a file or directory from a certain time
(assuming this time is between the first backup date and now). The following
syntax is used:

> YYYY-MM-DD, or interval using characters s, m, h, D, W, M, or Y. 1h78m, etc.

For example, a file could be restored from 2 weeks ago using

    horcrux -t 2W -f somefile.txt restore offsite1 ~/restore_directory/somefile.txt

Again, this time format is shown in the output from `horcrux help`.


### Checking a backup

This is a feature that isn't provided by [duplicity]. It ensures that a restored
backup is exactly the same as the local files that the backup is based
on. Alternatively, it'll show you exactly what files have been changed. It's
useful if you want to periodically check your backups.

First, you need to restore your backup to a directory. Then, run the command

    horcrux check offsite1 /Some/restore/directory

[md5deep] will run through all files in the restore directory, and then compare
the md5 hashes of each file with its equivalent in the backup's source
directory. If just one byte of a file is different its hash will also be
different, and this will be recorded. The list of changed files will be stored
in `~/.horcrux/offsite1-changed-hashes.log.bz2`.


### Removing old backup filesets

Performing a full backup followed by a few incremental backups produces the
following structure on the backup side:

    - full 1
        - incremental 1
        - incremental 2

If we then force a full backup then perform a few incremental backups, we'll end
up with

    - full 1
        - incremental 1
        - incremental 2
    - full 2
        - incremental 3
        - incremental 4

Future incremental backups (included those started by the `auto` operation) will
be attached to _full 2_. If space is at a premium, the old backup chains can be
removed using the `remove` operation. By default it'll leave 3 chains, but this
can be changed either globally in `~/.horcrux/horcrux.conf` or for a particular
backup using the `-c` option. If we run

    horcrux -c1 remove offsite1

for the above backup, the result will be

    - full 2
        - incremental 3
        - incremental 4


## Options

Horcrux supports a number of options that can be used to change an option from
the default. For example, perhaps you want to increase the verbosity of output
from the default of 5 to the maximum of 9. You would run

    horcrux -v9 auto offsite1

There are a number of possible options:

- `-a`: use gpg-agent
- `-c`: number of full filesets to leave during remove operation
- `-f`: file to restore
- `-h`: show help (this text)
- `-k`: encryption key
- `-i`: signing key
- `-n`: dry run
- `-o`: make auto operation run full backup if older than some time
- `-s`: backup source
- `-t`: time to restore file or directory from
- `-v`: verbosity
- `-z`: volume size in MB

If you run `horcrux help` then you'll be reminded of these options, together
with their current values (as set in `~/.horcrux/horcrux.conf`).


## Frequently asked questions

_I want to check what will happen before I run a backup_

: Use the dry run option, giving a command like `horcrux -n auto offsite1`.
: You could also use the `paramtest` operation, to very quickly check what
  parameters will be passed to Duplicity.

_I want to backup to the same place, but with different names_

: I backup to a home server. Locally it has an address `192.168.1.2`, but
  outside of my home network it has another address provided by [DynDNS].

: Use multiple configuration files, one for each location, with names like
  _offsite3-local-config_ and _offsite3-remote-config_. In each one specify the
  correct `destination_path`.

: Since it's actually the same backup, it'll just use the same exclude file as
  usual, _offsite3-exclude_.

: Run the backup as either `horcrux auto offsite3-local` or `horcrux auto
  offsite3-remote`. The correct configuration file will be loaded.

: All output will go into the same log file as usual, and the same local cache
  will be used.

_I want to backup locally first, then upload somewhere_

: Create your _backup-config_ file as usual, but also create a file called
  _backup-local-config_ (for example). In this second file, make sure the backup
  destination is set to a local drive; the former file can still have a remote
  path.

: Backup using `horcrux auto backup-local`, and then use `scp`, `rsync`, or
  whatever, to copy these files to the remote destination. You can run `horcrux
  auto backup` from then on. I often run something like this to maximise my
  upload bandwidth:

{% highlight bash %}
cd local_backup_dir
find . -name '*.gpg' -print0 | xargs -0I{} -n1 -P8 rsync -avhP {} \
    your.remote.server:backup
{% endhighlight %}


## Reporting bugs

Please send me email to [the usual address](mailto:chris@chrispoole.com).


## Requirements

- [duplicity]
- bzip2 or [pbzip2] (to compress log files)
- [md5deep] (for the `verify` operation)


## Change log

_20120327_
: Add support for passing extra parameters to Duplicity.
: Thanks to Roland Wirth for the suggestion.

_20110718_
: Add support for gpg-agent with `-a` and `use_agent` in horcrux.conf.
: Check for _backup_ directory in `/media/drive_name` as well as
  `/Volumes/drive_name`.
: Produce more output with `paramtest` operation (for debugging).



[duplicity]: http://duplicity.nongnu.org/
[backup strategy]: http://chrispoole.com/article/backup-strategy
[download horcrux]: http://chrispoole.com/downloads/horcrux
[md5deep]: http://md5deep.sourceforge.net/
[DynDNS]: http://www.dyndns.com/
[pbzip2]: http://compression.ca/pbzip2/
[man]: http://duplicity.nongnu.org/duplicity.1.html

[^recent]: See variable `full_if_old` in _horcrux.conf_. By default it's 360 days.
[^dest]: See variable `backup_basename` in _horcrux.conf_.
[^zero]: This feature assumes Mac OS X or another unix system where files are
    mounted in `/media`, such as Ubuntu.
