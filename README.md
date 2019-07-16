# zfs-usage-watch

One thing lacking in ZFS-on-Linux (ZOL) is a means to watch write bandwidth hitting a specific file system (dataset).  The `zpool iostat` utility shows aggregate read and write statistics against the underlying VDEVs.  But when the pool sits beneath multiple file systems and those file systems are NFS-shared to a cluster, for example, it's difficult to determine which file system is hitting the pool most heavily.

This utility checks the ZFS *used* property on one or more file systems on an arbitrary interval and reports change in excess of a threshold.  Since ZFS is copy-on-write, any change (removal + write) should be significant.

## Usage

```
$ zfs-usage-watch --help
usage: zfs-usage-watch [-h] [-v] [-m] [-S] [-T] [-P] [-i <seconds>]
                       [-t <rate>]
                       <fs-name> [<fs-name> ...]

positional arguments:
  <fs-name>             one or more ZFS file system names

optional arguments:
  -h, --help            show this help message and exit
  -v, --verbose         show additional information during execution
  -m, --match-regex     ZFS file system names are regular expressions to match
                        against the list of all file systems
  -S, --show-si-units   display rate/byte values in S.I. units, not binary
  -T, --show-timestamp  display a timestamp on each iteration
  -P, --show-percentage
                        display each filesystem percent of the total
  -i <seconds>, --interval <seconds>
                        number of seconds between checks
  -t <rate>, --threshold <rate>
                        any rate greater-than this value is displayed; units
                        are permissible (e.g. K, KiBps, kBps, kbps)
```

## Examples

The server is experiencing high write-intensive i/o load.  I first want to see if user home directories or workgroup directories are the source of the problem:

```
$ zfs-usage-watch r00nfs0/home r00nfs0/work
will sleep for 5 second(s) on each iteration
r00nfs0/home                                        1.97 KiBps (9.84 KiB)
                                                    1.97 KiBps (9.84 KiB)

r00nfs0/home                                      173.17 MiBps (865.84 MiB)
                                                  173.17 MiBps (865.84 MiB)

r00nfs0/work                                       23.62 KiBps (118.12 KiB)
r00nfs0/home                                        6.93 MiBps (34.65 MiB)
                                                    6.95 MiBps (34.77 MiB)

```

The last line on each event is the total over the file systems listed.  I see that a user is probably to blame, so I exit with Ctrl-C.  Now I watch all home file systems:

```
$ ./zfs-usage-watch --match-regex ^r00nfs0/home/
will sleep for 5 second(s) on each iteration
r00nfs0/home/1001                                  98.45 MiBps (492.26 MiB)
                                                   98.45 MiBps (492.26 MiB)

r00nfs0/home/1001                                 150.65 MiBps (753.27 MiB)
r00nfs0/home/1802                                   1.97 KiBps (9.84 KiB)
                                                  150.65 MiBps (753.27 MiB)

r00nfs0/home/1001                                 131.09 MiBps (655.45 MiB)
                                                  131.09 MiBps (655.45 MiB)

```

Whoever has `r00nfs0/home/1001` as his/her home directory on the cluster seems to be writing at a rather high rate, and is probably the reason for the write-intensive i/o load on the server.
