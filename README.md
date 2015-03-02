# NAME

ec2-consistent-snapshot - Create EBS snapshots on EC2 w/consistent filesystem/db

# SYNOPSIS

    ec2-consistent-snapshot [opts] VOLUMEID...

# OPTIONS

- \-h --help

    Print help and exit.

- \-d --debug

    Debug mode.

- \-q --quiet

    Quiet mode.

- \-n --noaction

    Dry run. Just say what you would have done, don't do it.

- \--region REGION

    Specify a different EC2 region like "eu-west-1".  Defaults to
    "us-east-1".

- \--profile PROFILE

    Specify a named profile for the AWS cli command.  You can configure
    additional profiles by using `aws configure` with the `--profile` option
    or by adding entries to the config and credentials files.

- \--description DESCRIPTION

    Specify a description string for the EBS snapshot.  Defaults to the
    name of the program.

    You may specify this option multiple times if you need to customize
    descriptions of multiple volumes snapshots. If specified multiple
    times descriptions count has to match volumes count and they will be
    applied on the same order.

- \--freeze-filesystem MOUNTPOINT
- \--xfs-filesystem MOUNTPOINT \[OBSOLESCENT form of the same option\]

    Indicates that the filesystem at the specified mount point should be
    flushed and frozen during the snapshot. Requires the xfs\_freeze or
    fsfreeze program. Note that xfs\_freeze is equivalent to fsfreeze and
    works on any filesystems that support freezing, provided the kernel
    you are using supports it. (Linux Ext3/4, ReiserFS, JFS, XFS.)
    fsfreeze comes with newer versions of util-linux.

    You may specify this option multiple times if you need to freeze multiple
    filesystems on the the EBS volume(s).

- \--mongo

    Indicates that the volume contains data files for a running Mongo
    database, which will be flushed and locked during the snapshot.

- \--mongo-host HOST
- \--mongo-port PORT
- \--mongo-username USER
- \--mongo-password PASS

    Mongo host, port, username, and password used to flush logs if there
    is authentication required on the admin database.

- \--mongo-stop

    Indicates that the volume contains data files for a running Mongo
    instance.  The instance is shutdown before the snapshot is initiated
    and restarted afterwards. \[EXPERIMENTAL\]

- \--mysql

    Indicates that the volume contains data files for a running MySQL
    database, which will be flushed and locked during the snapshot.

- \--mysql-defaults-file FILE

    MySQL defaults file, containing host, username and password, this
    option will ignore the --mysql-host, --mysql-username,
    \--mysql-password parameters

- \--mysql-host HOST
- \--mysql-socket PATH
- \--mysql-username USER
- \--mysql-password PASS

    MySQL host, socket path, username, and password used to flush logs and
    lock tables.  User must have appropriate permissions.  Defaults to
    $HOME/.my.cnf file contents.

- \--mysql-master-status-file FILE

    Store the MASTER STATUS output in a file on the snapshot. It will be
    removed after the EBS snapshot is taken.  This option will be ignored
    with --mysql-stop

- \--mysql-stop

    Indicates that the volume contains data files for a running MySQL
    database.  The database is shutdown before the snapshot is initiated
    and restarted afterwards. \[EXPERIMENTAL\]

- \--snapshot-timeout SECONDS

    How many seconds to wait for the snapshot-create to return.  Defaults
    to 10.0

- \--lock-timeout SECONDS

    How many seconds to wait for a database lock. Defaults to 0.5.
    Making this too large can force other processes to wait while this
    process waits for a lock.  Better to make it small and try lots of
    times.

- \--lock-tries COUNT

    How many times to try to get a database lock before failing.  Defaults
    to 60.

- \--lock-sleep SECONDS

    How many seconds to sleep between database lock tries.  Defaults
    to 5.0.

- \--pre-freeze-command COMMAND

    Command to run after MySQL stop/lock and before filesystem freeze.

- \--post-thaw-command COMMAND

    Command to run immediately after filesystem unfreeze and before MySQL
    start/unlock.

# ARGUMENTS

- VOLUMEID

    EBS volume id(s) for which a snapshot is to be created.

# DESCRIPTION

This program creates an EBS snapshot for an Amazon EC2 EBS volume.  To
help ensure consistent data in the snapshot, it tries to flush and
freeze the filesystem(s) first as well as flushing and locking the
database, if applicable.

Filesystems can be frozen during the snapshot. Prior to Linux kernel
2.6.29, XFS must be used for freezing support. While frozen, a filesystem
will be consistent on disk and all writes will block.

There are a number of timeouts to reduce the risk of interfering with
the normal database operation while improving the chances of getting a
consistent snapshot.

If you have multiple EBS volumes in a RAID configuration, you can
specify all of the volume ids on the command line and it will create
snapshots for each while the filesystem and database are locked.  Note
that it is your responsibility to keep track of the resulting snapshot
ids and to figure out how to put these back together when you need to
restore the RAID setup.

If you have multiple EBS volumes which are hosting different file
systems, it might be better to simply run the command once for each
volume id.

NOTE: This is a modified version of Eric Hammond's original
ec2-consistent-snapshot program.  This version uses the AWS CLI tool,
instead of the Net::Amazon::EC2 Perl module.  The original program
is available here: https://github.com/alestic/ec2-consistent-snapshot

# EXAMPLES

Snapshot a volume with a frozen filesystem under /vol containing a
MySQL database:

    ec2-consistent-snapshot --mysql --freeze-filesystem /vol vol-VOLUMEID

Snapshot a volume with a frozen filesystem under /data containing a
Mongo database:

    ec2-consistent-snapshot --mongo --freeze-filesystem /data vol-VOLUMEID

Snapshot a volume mounted with a frozen filesystem on /var/local but
with no MySQL database:

    ec2-consistent-snapshot --freeze-filesystem /var/local vol-VOLUMEID

Snapshot four European volumes in a RAID configuration with MySQL,
saving the snapshots with a description marking the current time:

    ec2-consistent-snapshot                                      \
      --mysql                                                    \
      --freeze-filesystem /vol                                   \
      --region eu-west-1                                         \
      --description "RAID snapshot $(date +'%Y-%m-%d %H:%M:%S')" \
      vol-VOL1 vol-VOL2 vol-VOL3 vol-VOL4

Snapshot four us-east-1 volumes in a RAID configuration with Mongo,
saving the snapshots with a description marking the current time:

    ec2-consistent-snapshot                                      \
      --mongo                                                    \
      --freeze-filesystem /data                                  \
      --region us-east-1                                         \
      --description "RAID snapshot $(date +'%Y-%m-%d %H:%M:%S')" \
      vol-VOL1 vol-VOL2 vol-VOL3 vol-VOL4

Snapshot two volumes with customized descriptions:

    ec2-consistent-snapshot                                      \
      --description "Description 1st Volume"                     \
      --description "Description 2nd Volume"                     \
      vol-VOL1 vol-VOL2

# FILES

- $HOME/.my.cnf

    Default values for MySQL user and password are sought here in the
    standard format.

# INSTALLATION

This program relies on the AWS CLI tool.  See installation instructions
here: http://docs.aws.amazon.com/cli/latest/userguide/installing.html

This program also relies on the JSON perl module, which can be installed
with cpan:

    cpan JSON


# SEE ALSO

- Amazon EC2
- Amazon EC2 EBS (Elastic Block Store)
- ec2-create-snapshot

# CAVEATS

Freezing the root filesystem is not recommended. Be sure to test
each filesystem you use it on before putting this into production, in
exactly the way you would run it in production (e.g., inside cron if
that's how you invoke it).

ec2-consistent-snapshot can hang if its output is directed at a
filesystem that is being frozen, leading to a dead machine.

EBS snapshots are a critical part of protecting your valuable data.
This program or the environment in which it is run may contain defects
that cause snapshots to not be created.  Please test and check to make
sure that snapshots are getting created for the volumes as you intend.

EBS snapshots cost money to create and to store in your AWS account.
Be aware of and monitor your expenses.

You are responsible for what happens in your EC2 account.  This
software is intended, but not guaranteed, to help in that effort.

This program assumes that you have the AWS CLI tool installed and
configured with the appropriate access keys.  You can configure these
by running `aws configure`.

# CREDITS

Thanks to the following for performing tests on early versions,
providing feature development, feedback, bug reports, and patches:

    David Erickson
    Steve Caldwell
    Gryp
    Ken Huang
    Jefferson Noxon
    Bobb Crosbie
    Craig Tracey
    Diego Salvi
    Christian Marquardt
    Todd Roman
    Ben Tucker
    David Rogeres
    Kevin Lewis
    Eric Lubow
    Seth de l'Isle
    Peter Waller
    yalamber
    Daniel Beardsley

# AUTHOR/MAINTAINER

Matt Lyon <talisto@gmail.com>

# LICENSE

Original work Copyright (C) 2009-2014 Eric Hammond <ehammond@thinksome.com>

Modified work Copyright (C) 2015 Matt Lyon <talisto@gmail.com>

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
