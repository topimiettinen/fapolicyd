File Access Policy Daemon
=========================

[![Build Status](https://travis-ci.com/linux-application-whitelisting/fapolicyd.svg?branch=master)](https://travis-ci.com/linux-application-whitelisting/fapolicyd)

This is a simple application whitelisting daemon for Linux.

DEPENDENCIES (fedora)
---------------------
* gcc
* autoconf
* automake
* libtool
* make
* libudev-devel
* kernel-headers
* systemd-devel
* libgcrypt-devel
* rpm-devel (optional)
* file-devel
* libcap-ng-devel
* libseccomp-devel
* lmdb-devel
* python3-devel

BUILDING
--------
To build from the repo after cloning:

```
$ cd fapolicyd
$ ./autogen.sh
$ ./configure
$ make
$ make dist
```

This will create a tarball. You can use the new tarball with the spec file
and create your own rpm. If you want to experiment without installing, just
run make with no arguments. It should run fine from where it was built as
long as you put the configuration files in /etc/fapolicyd. The fapolicyd.rules
and fapolicyd.mounts files go there.

Note that the shipped policy expects that auditing is enabled. This is done
by passing --with-audit to ./configure.

The use of rpm as a trust source is now optional. You can run ./configure
passing --without-rpm and it will not link against librpm. In this mode, it
purely uses the file database in fapolicyd.trust. If rpm is used, then the
file trust database can be used in addition to rpmdb.


RUNNING
-------
You might want to look at the fapolicyd.rules file to see what the sample
policy looks like. The policy is designed with 3 goals in mind.

1. No bypass of security by executing programs via ld.so.
2. All approved executables are trusted. Untrusted programs can't run.
3. Elf binaries, python, and shell scripts are enabled for trusted
   applications/libraries. Other languages are not allowed or must be enabled.

You can test by starting the daemon from the command line. Before starting
the daemon, cp /usr/bin/ls /usr/bin/my-ls just to setup for testing. When
testing new policy, its highly recommended to use the permissive mode to
make sure nothing bad happens. It really is not too hard to deadlock your
system. Continuing on with the tutorial, as root start the daemon as follows:
```
/usr/sbin/fapolicyd --permissive --debug
```
Then in another window do the following:

1. /usr/lib64/ld-2.29.so /usr/bin/ls
2. my-ls
3. run a python file in your home directory.
4. run a program from /tmp

In permissive + debug mode you will see dec=deny which means
"decision is to deny". But the program will actually be allowed to run.

You can run the daemon from the command line with --debug-deny command
line option. This culls the event notification to only print the denials.
If this is running cleanly, then you can remove the --permissive option
and get true denials. Now retest above steps and see the difference.


DEBUG MODE
----------
In debug mode, you will see events such as this:

```
rule:9 dec=deny_audit perm=execute auid=1001 pid=14137 exe=/usr/bin/bash : file=/home/joe/my-ls ftype=application/x-executable
```

What this is saying is rule 9 made the ultimate Decision that was followed.
The Decision is to deny access and create an audit event. The subject is the
user that logged in as user id 1001. The subject's process id that is trying
to perform an action is 14137. The current executable that the subject is
using is bash. Bash wanted permission to execute /home/joe/my-ls which is the
object. And the object is an ELF executable.


WRITING RULES
-------------
The rules follow a simple "decision permission subject : object" recipe. For
more information, see the fapolicyd.rules man page.


REPORT
------
On shutdown the daemon will write an object access report to
/var/log/fapolicyd-access.log. The report is from oldest access to newest.
Timestamps are not included because that would be a severe performance hit.
The report gives some basic forensic information about what was being accessed.

PERFORMANCE
-----------
When a program opens a file or calls execve, that thread has to wait for 
fapolicyd to make a decision. Fapolicyd caches everything it looks up so
that subsequent access uses the cache rather than looking things up from
scratch. But the cache is only so big. You are in control of it though.
You can make both subject and object caches bigger. When the program ends,
it will output some performance statistic like this:

```
Inter-thread max queue depth 7
Allowed accesses: 116103
Denied accesses: 17
Subject cache size: 1031
Subject slots in use: 1023
Subject hits: 115097
Subject misses: 6770
Subject evictions: 5747
Object cache size: 6151
Object slots in use: 6086
Object hits: 110034
Object misses: 34984
Object evictions: 28898
```

In this report, you can see that the internal request queue maxed out at 7.
This means that the application at most had 7 threads/processes waiting at
the most. This shows that it got a little backed up but was handling requests
pretty quick.

Another statistic worth looking at is the hits to evictions ratio. When a
request has nowhere to put information, it has to evict something to make
room. This is done by a LRU cache which naturally determines what's not
getting used and makes it available for recycling.

In the above statistics, the subject hit ratio was 95%. The object cache was
not quite as lucky. For it, we get a hit ration of 79%. This is still good,
but could be better. This would suggest that for the workload on that system,
the cache could be a little bigger. If the number used for the cache size is
a prime number, you will get less cache churn due to collisions than if it
had a common denominator.


NOTES
-----
* Its highly recommended to run in permissive mode while you are testing the
daemon's policy.

* Stracing the daemon can deadlock the system.

* About shell script restrictions...there's not much difference between
running a script or someone typing things in by hand. The aim at this
point is to check that any programs it calls meets the policy.

* If on startup you find that the database has a miscompare every single time,
then you probably have some i686 rpms which is causing duplicate entries
where the sha256 hash is different. To fix, remove the offending i686 rpm.

* If for some reason rpm database errors are detected, you may need to do
the following:

```
1. db_verify /var/lib/rpm/Packages
if OK, then
2. rm -f /var/lib/rpm/__db*
3. rpm --rebuilddb
```
