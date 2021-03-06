=====================
Administrator's Guide
=====================

------------------
Managing the Rings
------------------

You need to build the storage rings on the proxy server node, and
distribute them to all the servers in the cluster. Storage rings
contain information about all the Swift storage partitions and how
they are distributed between the different nodes and disks. For more
information see :doc:`overview_ring`.

Removing a device from the ring::

    swift-ring-builder <builder-file> remove <ip_address>/<device_name>
    
Removing a server from the ring::

    swift-ring-builder <builder-file> remove <ip_address>
    
Adding devices to the ring:

See :ref:`ring-preparing`
    
See what devices for a server are in the ring::

    swift-ring-builder <builder-file> search <ip_address>

Once you are done with all changes to the ring, the changes need to be
"committed"::

    swift-ring-builder <builder-file> rebalance
    
Once the new rings are built, they should be pushed out to all the servers
in the cluster.

-----------------------
Scripting Ring Creation
-----------------------
You can create scripts to create the account and container rings and rebalance. Here's an example script for the Account ring. Use similar commands to create a make-container-ring.sh script on the proxy server node.

1. Create a script file called make-account-ring.sh on the proxy
   server node with the following content::

    #!/bin/bash
    cd /etc/swift
    rm -f account.builder account.ring.gz backups/account.builder backups/account.ring.gz
    swift-ring-builder account.builder create 18 3 1
    swift-ring-builder account.builder add z1-<account-server-1>:6002/sdb1 1
    swift-ring-builder account.builder add z2-<account-server-2>:6002/sdb1 1
    swift-ring-builder account.builder rebalance

   You need to replace the values of <account-server-1>,
   <account-server-2>, etc. with the IP addresses of the account
   servers used in your setup. You can have as many account servers as
   you need. All account servers are assumed to be listening on port
   6002, and have a storage device called "sdb1" (this is a directory
   name created under /drives when we setup the account server). The
   "z1", "z2", etc. designate zones, and you can choose whether you
   put devices in the same or different zones.

2. Make the script file executable and run it to create the account ring file::

    chmod +x make-account-ring.sh
    sudo ./make-account-ring.sh

3. Copy the resulting ring file /etc/swift/account.ring.gz to all the
   account server nodes in your Swift environment, and put them in the
   /etc/swift directory on these nodes. Make sure that every time you
   change the account ring configuration, you copy the resulting ring
   file to all the account nodes.

-----------------------
Handling System Updates
-----------------------

It is recommended that system updates and reboots are done a zone at a time.
This allows the update to happen, and for the Swift cluster to stay available
and responsive to requests.  It is also advisable when updating a zone, let
it run for a while before updating the other zones to make sure the update
doesn't have any adverse effects.

----------------------
Handling Drive Failure
----------------------

In the event that a drive has failed, the first step is to make sure the drive
is unmounted.  This will make it easier for swift to work around the failure
until it has been resolved.  If the drive is going to be replaced immediately,
then it is just best to replace the drive, format it, remount it, and let
replication fill it up.

If the drive can't be replaced immediately, then it is best to leave it
unmounted, and remove the drive from the ring. This will allow all the
replicas that were on that drive to be replicated elsewhere until the drive
is replaced.  Once the drive is replaced, it can be re-added to the ring.

-----------------------
Handling Server Failure
-----------------------

If a server is having hardware issues, it is a good idea to make sure the 
swift services are not running.  This will allow Swift to work around the
failure while you troubleshoot.

If the server just needs a reboot, or a small amount of work that should
only last a couple of hours, then it is probably best to let Swift work
around the failure and get the machine fixed and back online.  When the
machine comes back online, replication will make sure that anything that is
missing during the downtime will get updated.

If the server has more serious issues, then it is probably best to remove
all of the server's devices from the ring.  Once the server has been repaired
and is back online, the server's devices can be added back into the ring.
It is important that the devices are reformatted before putting them back
into the ring as it is likely to be responsible for a different set of
partitions than before.

-----------------------
Detecting Failed Drives
-----------------------

It has been our experience that when a drive is about to fail, error messages
will spew into `/var/log/kern.log`.  There is a script called
`swift-drive-audit` that can be run via cron to watch for bad drives.  If 
errors are detected, it will unmount the bad drive, so that Swift can
work around it.  The script takes a configuration file with the following
settings:

[drive-audit]

==================  ==========  ===========================================
Option              Default     Description
------------------  ----------  -------------------------------------------
log_facility        LOG_LOCAL0  Syslog log facility
log_level           INFO        Log level
device_dir          /srv/node   Directory devices are mounted under
minutes             60          Number of minutes to look back in
                                `/var/log/kern.log`
error_limit         1           Number of errors to find before a device
                                is unmounted
==================  ==========  ===========================================

This script has only been tested on Ubuntu 10.04, so if you are using a
different distro or OS, some care should be taken before using in production.

--------------
Cluster Health
--------------

There is a swift-dispersion-report tool for measuring overall cluster health.
This is accomplished by checking if a set of deliberately distributed
containers and objects are currently in their proper places within the cluster.

For instance, a common deployment has three replicas of each object. The health
of that object can be measured by checking if each replica is in its proper
place. If only 2 of the 3 is in place the object's heath can be said to be at
66.66%, where 100% would be perfect.

A single object's health, especially an older object, usually reflects the
health of that entire partition the object is in. If we make enough objects on
a distinct percentage of the partitions in the cluster, we can get a pretty
valid estimate of the overall cluster health. In practice, about 1% partition
coverage seems to balance well between accuracy and the amount of time it takes
to gather results.

The first thing that needs to be done to provide this health value is create a
new account solely for this usage. Next, we need to place the containers and
objects throughout the system so that they are on distinct partitions. The
swift-dispersion-populate tool does this by making up random container and
object names until they fall on distinct partitions. Last, and repeatedly for
the life of the cluster, we need to run the swift-dispersion-report tool to
check the health of each of these containers and objects.

These tools need direct access to the entire cluster and to the ring files
(installing them on a proxy server will probably do). Both
swift-dispersion-populate and swift-dispersion-report use the same
configuration file, /etc/swift/dispersion.conf. Example conf file::

    [dispersion]
    auth_url = http://saio:11000/auth/v1.0
    auth_user = test:tester
    auth_key = testing

There are also options for the conf file for specifying the dispersion coverage
(defaults to 1%), retries, concurrency, etc. though usually the defaults are
fine.

Once the configuration is in place, run `swift-dispersion-populate` to populate
the containers and objects throughout the cluster.

Now that those containers and objects are in place, you can run
`swift-dispersion-report` to get a dispersion report, or the overall health of
the cluster. Here is an example of a cluster in perfect health::

    $ swift-dispersion-report
    Queried 2621 containers for dispersion reporting, 19s, 0 retries
    100.00% of container copies found (7863 of 7863)
    Sample represents 1.00% of the container partition space
    
    Queried 2619 objects for dispersion reporting, 7s, 0 retries
    100.00% of object copies found (7857 of 7857)
    Sample represents 1.00% of the object partition space

Now I'll deliberately double the weight of a device in the object ring (with
replication turned off) and rerun the dispersion report to show what impact
that has::

    $ swift-ring-builder object.builder set_weight d0 200
    $ swift-ring-builder object.builder rebalance
    ...
    $ swift-dispersion-report
    Queried 2621 containers for dispersion reporting, 8s, 0 retries
    100.00% of container copies found (7863 of 7863)
    Sample represents 1.00% of the container partition space
    
    Queried 2619 objects for dispersion reporting, 7s, 0 retries
    There were 1763 partitions missing one copy.
    77.56% of object copies found (6094 of 7857)
    Sample represents 1.00% of the object partition space

You can see the health of the objects in the cluster has gone down
significantly. Of course, I only have four devices in this test environment, in
a production environment with many many devices the impact of one device change
is much less. Next, I'll run the replicators to get everything put back into
place and then rerun the dispersion report::

    ... start object replicators and monitor logs until they're caught up ...
    $ swift-dispersion-report
    Queried 2621 containers for dispersion reporting, 17s, 0 retries
    100.00% of container copies found (7863 of 7863)
    Sample represents 1.00% of the container partition space

    Queried 2619 objects for dispersion reporting, 7s, 0 retries
    100.00% of object copies found (7857 of 7857)
    Sample represents 1.00% of the object partition space

Alternatively, the dispersion report can also be output in json format. This 
allows it to be more easily consumed by third party utilities::

    $ swift-dispersion-report -j
    {"object": {"retries:": 0, "missing_two": 0, "copies_found": 7863, "missing_one": 0, "copies_expected": 7863, "pct_found": 100.0, "overlapping": 0, "missing_all": 0}, "container": {"retries:": 0, "missing_two": 0, "copies_found": 12534, "missing_one": 0, "copies_expected": 12534, "pct_found": 100.0, "overlapping": 15, "missing_all": 0}}


--------------------------------
Cluster Telemetry and Monitoring
--------------------------------

Various metrics and telemetry can be obtained from the account, container, and
object servers using the recon server middleware and the swift-recon cli. To do
so update your account, container, or object servers pipelines to include recon
and add the associated filter config.

object-server.conf sample::

    [pipeline:main]
    pipeline = recon object-server

    [filter:recon]
    use = egg:swift#recon
    recon_cache_path = /var/cache/swift

container-server.conf sample::

    [pipeline:main]
    pipeline = recon container-server

    [filter:recon]
    use = egg:swift#recon
    recon_cache_path = /var/cache/swift

account-server.conf sample::

    [pipeline:main]
    pipeline = recon account-server

    [filter:recon]
    use = egg:swift#recon
    recon_cache_path = /var/cache/swift

The recon_cache_path simply sets the directory where stats for a few items will
be stored. Depending on the method of deployment you may need to create this
directory manually and ensure that swift has read/write access.

Finally, if you also wish to track asynchronous pending on your object
servers you will need to setup a cronjob to run the swift-recon-cron script
periodically on your object servers::

    */5 * * * * swift /usr/bin/swift-recon-cron /etc/swift/object-server.conf

Once the recon middleware is enabled a GET request for "/recon/<metric>" to
the server will return a json formatted response::

    fhines@ubuntu:~$ curl -i http://localhost:6030/recon/async
    HTTP/1.1 200 OK
    Content-Type: application/json
    Content-Length: 20
    Date: Tue, 18 Oct 2011 21:03:01 GMT

    {"async_pending": 0}

The following metrics and telemetry are currently exposed::

========================    ========================================================================================
Request URI                 Description
------------------------    ----------------------------------------------------------------------------------------
/recon/load                 returns 1,5, and 15 minute load average
/recon/mem                  returns /proc/meminfo
/recon/mounted              returns *ALL* currently mounted filesystems
/recon/unmounted            returns all unmounted drives if mount_check = True
/recon/diskusage            returns disk utilization for storage devices
/recon/ringmd5              returns object/container/account ring md5sums
/recon/quarantined          returns # of quarantined objects/accounts/containers
/recon/sockstat             returns consumable info from /proc/net/sockstat|6
/recon/devices              returns list of devices and devices dir i.e. /srv/node
/recon/async                returns count of async pending
/recon/replication          returns object replication times (for backward compatability)
/recon/replication/<type>   returns replication info for given type (account, container, object)
/recon/auditor/<type>       returns auditor stats on last reported scan for given type (account, container, object)
/recon/updater/<type>       returns last updater sweep times for given type (container, object)
=========================   =======================================================================================

This information can also be queried via the swift-recon command line utility::

    fhines@ubuntu:~$ swift-recon -h
    Usage: 
            usage: swift-recon <server_type> [-v] [--suppress] [-a] [-r] [-u] [-d]
            [-l] [--md5] [--auditor] [--updater] [--expirer] [--sockstat]

            <server_type>   account|container|object
            Defaults to object server.

            ex: swift-recon container -l --auditor


    Options:
      -h, --help            show this help message and exit
      -v, --verbose         Print verbose info
      --suppress            Suppress most connection related errors
      -a, --async           Get async stats
      -r, --replication     Get replication stats
      --auditor             Get auditor stats
      --updater             Get updater stats
      --expirer             Get expirer stats
      -u, --unmounted       Check cluster for unmounted devices
      -d, --diskusage       Get disk usage stats
      -l, --loadstats       Get cluster load average stats
      -q, --quarantined     Get cluster quarantine stats
      --md5                 Get md5sum of servers ring and compare to local copy
      --sockstat            Get cluster socket usage stats
      --all                 Perform all checks. Equal to -arudlq --md5 --sockstat
      -z ZONE, --zone=ZONE  Only query servers in specified zone
      -t SECONDS, --timeout=SECONDS
                            Time to wait for a response from a server
      --swiftdir=SWIFTDIR   Default = /etc/swift

For example, to obtain container replication info from all hosts in zone "3"::

    fhines@ubuntu:~$ swift-recon container -r --zone 3
    ===============================================================================
    --> Starting reconnaissance on 1 hosts
    ===============================================================================
    [2012-04-02 02:45:48] Checking on replication
    [failure] low: 0.000, high: 0.000, avg: 0.000, reported: 1
    [success] low: 486.000, high: 486.000, avg: 486.000, reported: 1
    [replication_time] low: 20.853, high: 20.853, avg: 20.853, reported: 1
    [attempted] low: 243.000, high: 243.000, avg: 243.000, reported: 1

---------------------------
Reporting Metrics to StatsD
---------------------------

If you have a StatsD_ server running, Swift may be configured to send it
real-time operational metrics.  To enable this, set the following
configuration entries (see the sample configuration files)::

    log_statsd_host = localhost
    log_statsd_port = 8125
    log_statsd_default_sample_rate = 1
    log_statsd_metric_prefix =                [empty-string]

If `log_statsd_host` is not set, this feature is disabled.  The default values
for the other settings are given above.

.. _StatsD: http://codeascraft.etsy.com/2011/02/15/measure-anything-measure-everything/
.. _Graphite: http://graphite.wikidot.com/
.. _Ganglia: http://ganglia.sourceforge.net/

The sample rate is a real number between 0 and 1 which defines the
probability of sending a sample for any given event or timing measurement.
This sample rate is sent with each sample to StatsD and used to
multiply the value.  For example, with a sample rate of 0.5, StatsD will
multiply that counter's value by 2 when flushing the metric to an upstream
monitoring system (Graphite_, Ganglia_, etc.).  To get the best data, start
with the default `log_statsd_default_sample_rate` value of 1 and only lower
it as needed.

The metric prefix will be prepended to every metric sent to the StatsD server
For example, with::

    log_statsd_metric_prefix = proxy01

the metric `proxy-server.errors` would be sent to StatsD as
`proxy01.proxy-server.errors`.  This is useful for differentiating different
servers when sending statistics to a central StatsD server.  If you run a local
StatsD server per node, you could configure a per-node metrics prefix there and
leave `log_statsd_metric_prefix` blank.

Note that metrics reported to StatsD are counters or timing data (which
StatsD usually expands out to min, max, avg, count, and 90th percentile
per timing metric).  Some important "gauge" metrics will still need to
be collected using another method.  For example, the
`object-server.async_pendings` StatsD metric counts the generation of
async_pendings in real-time, but will not tell you the current number
of async_pending container updates on disk at any point in time.

Note also that the set of metrics collected, their names, and their semantics
are not locked down and will change over time.  StatsD logging is currently in
a "beta" stage and will continue to evolve.

Metrics for `account-auditor`:

==========================  =========================================================
Metric Name                 Description
--------------------------  ---------------------------------------------------------
`account-auditor.errors`    Count of audit runs (across all account databases) which
                            caught an Exception.
`account-auditor.passes`    Count of individual account databases which passed audit.
`account-auditor.failures`  Count of individual account databases which failed audit.
`account-auditor.timing`    Timing data for individual account database audits.
==========================  =========================================================

Metrics for `account-reaper`:

==============================================  ====================================================
Metric Name                                     Description
----------------------------------------------  ----------------------------------------------------
`account-reaper.errors`                         Count of devices failing the mount check.
`account-reaper.timing`                         Timing data for each reap_account() call.
`account-reaper.return_codes.X`                 Count of HTTP return codes from various operations
                                                (eg. object listing, container deletion, etc.). The
                                                value for X is the first digit of the return code
                                                (2 for 201, 4 for 404, etc.).
`account-reaper.containers_failures`            Count of failures to delete a container.
`account-reaper.containers_deleted`             Count of containers successfully deleted.
`account-reaper.containers_remaining`           Count of containers which failed to delete with
                                                zero successes.
`account-reaper.containers_possibly_remaining`  Count of containers which failed to delete with
                                                at least one success.
`account-reaper.objects_failures`               Count of failures to delete an object.
`account-reaper.objects_deleted`                Count of objects successfully deleted.
`account-reaper.objects_remaining`              Count of objects which failed to delete with zero
                                                successes.
`account-reaper.objects_possibly_remaining`     Count of objects which failed to delete with at
                                                least one success.
==============================================  ====================================================

Metrics for `account-server` ("Not Found" is not considered an error and requests
which increment `errors` are not included in the timing data):

=================================  ====================================================
Metric Name                        Description
---------------------------------  ----------------------------------------------------
`account-server.DELETE.errors`     Count of errors handling DELETE requests: bad
                                   request, not mounted, missing timestamp.
`account-server.DELETE.timing`     Timing data for each DELETE request not resulting in
                                   an error.
`account-server.PUT.errors`        Count of errors handling PUT requests: bad request,
                                   not mounted, conflict.
`account-server.PUT.timing`        Timing data for each PUT request not resulting in an
                                   error.
`account-server.HEAD.errors`       Count of errors handling HEAD requests: bad request,
                                   not mounted.
`account-server.HEAD.timing`       Timing data for each HEAD request not resulting in
                                   an error.
`account-server.GET.errors`        Count of errors handling GET requests: bad request,
                                   not mounted, bad delimiter, account listing limit
                                   too high, bad accept header.
`account-server.GET.timing`        Timing data for each GET request not resulting in
                                   an error.
`account-server.REPLICATE.errors`  Count of errors handling REPLICATE requests: bad
                                   request, not mounted.
`account-server.REPLICATE.timing`  Timing data for each REPLICATE request not resulting
                                   in an error.
`account-server.POST.errors`       Count of errors handling POST requests: bad request,
                                   bad or missing timestamp, not mounted.
`account-server.POST.timing`       Timing data for each POST request not resulting in
                                   an error.
=================================  ====================================================

Metrics for `account-replicator`:

==================================  ====================================================
Metric Name                         Description
----------------------------------  ----------------------------------------------------
`account-replicator.diffs`          Count of syncs handled by sending differing rows.
`account-replicator.diff_caps`      Count of "diffs" operations which failed because
                                    "max_diffs" was hit.
`account-replicator.no_changes`     Count of accounts found to be in sync.
`account-replicator.hashmatches`    Count of accounts found to be in sync via hash
                                    comparison (`broker.merge_syncs` was called).
`account-replicator.rsyncs`         Count of completely missing accounts where were sent
                                    via rsync.
`account-replicator.remote_merges`  Count of syncs handled by sending entire database
                                    via rsync.
`account-replicator.attempts`       Count of database replication attempts.
`account-replicator.failures`       Count of database replication attempts which failed
                                    due to corruption (quarantined) or inability to read
                                    as well as attempts to individual nodes which
                                    failed.
`account-replicator.removes`        Count of databases deleted because the
                                    delete_timestamp was greater than the put_timestamp
                                    and the database had no rows or because it was
                                    successfully sync'ed to other locations and doesn't
                                    belong here anymore.
`account-replicator.successes`      Count of replication attempts to an individual node
                                    which were successful.
`account-replicator.timing`         Timing data for each database replication attempt
                                    not resulting in a failure.
==================================  ====================================================

Metrics for `container-auditor`:

============================  ====================================================
Metric Name                   Description
----------------------------  ----------------------------------------------------
`container-auditor.errors`    Incremented when an Exception is caught in an audit
                              pass (only once per pass, max).
`container-auditor.passes`    Count of individual containers passing an audit.
`container-auditor.failures`  Count of individual containers failing an audit.
`container-auditor.timing`    Timing data for each container audit.
============================  ====================================================

Metrics for `container-replicator`:

====================================  ====================================================
Metric Name                           Description
------------------------------------  ----------------------------------------------------
`container-replicator.diffs`          Count of syncs handled by sending differing rows.
`container-replicator.diff_caps`      Count of "diffs" operations which failed because
                                      "max_diffs" was hit.
`container-replicator.no_changes`     Count of containers found to be in sync.
`container-replicator.hashmatches`    Count of containers found to be in sync via hash
                                      comparison (`broker.merge_syncs` was called).
`container-replicator.rsyncs`         Count of completely missing containers where were sent
                                      via rsync.
`container-replicator.remote_merges`  Count of syncs handled by sending entire database
                                      via rsync.
`container-replicator.attempts`       Count of database replication attempts.
`container-replicator.failures`       Count of database replication attempts which failed
                                      due to corruption (quarantined) or inability to read
                                      as well as attempts to individual nodes which
                                      failed.
`container-replicator.removes`        Count of databases deleted because the
                                      delete_timestamp was greater than the put_timestamp
                                      and the database had no rows or because it was
                                      successfully sync'ed to other locations and doesn't
                                      belong here anymore.
`container-replicator.successes`      Count of replication attempts to an individual node
                                      which were successful.
`container-replicator.timing`         Timing data for each database replication attempt
                                      not resulting in a failure.
====================================  ====================================================

Metrics for `container-server` ("Not Found" is not considered an error and requests
which increment `errors` are not included in the timing data):

===================================  ====================================================
Metric Name                          Description
-----------------------------------  ----------------------------------------------------
`container-server.DELETE.errors`     Count of errors handling DELETE requests: bad
                                     request, not mounted, missing timestamp, conflict.
`container-server.DELETE.timing`     Timing data for each DELETE request not resulting in
                                     an error.
`container-server.PUT.errors`        Count of errors handling PUT requests: bad request,
                                     missing timestamp, not mounted, conflict.
`container-server.PUT.timing`        Timing data for each PUT request not resulting in an
                                     error.
`container-server.HEAD.errors`       Count of errors handling HEAD requests: bad request,
                                     not mounted.
`container-server.HEAD.timing`       Timing data for each HEAD request not resulting in
                                     an error.
`container-server.GET.errors`        Count of errors handling GET requests: bad request,
                                     not mounted, parameters not utf8, bad accept header.
`container-server.GET.timing`        Timing data for each GET request not resulting in
                                     an error.
`container-server.REPLICATE.errors`  Count of errors handling REPLICATE requests: bad
                                     request, not mounted.
`container-server.REPLICATE.timing`  Timing data for each REPLICATE request not resulting
                                     in an error.
`container-server.POST.errors`       Count of errors handling POST requests: bad request,
                                     bad x-container-sync-to, not mounted.
`container-server.POST.timing`       Timing data for each POST request not resulting in
                                     an error.
===================================  ====================================================

Metrics for `container-sync`:

===============================  ====================================================
Metric Name                      Description
-------------------------------  ----------------------------------------------------
`container-sync.skips`           Count of containers skipped because they don't have
                                 sync'ing enabled.
`container-sync.failures`        Count of failures sync'ing of individual containers.
`container-sync.syncs`           Count of individual containers sync'ed successfully.
`container-sync.deletes`         Count of container database rows sync'ed by
                                 deletion.
`container-sync.deletes.timing`  Timing data for each container database row
                                 sychronization via deletion.
`container-sync.puts`            Count of container database rows sync'ed by PUTing.
`container-sync.puts.timing`     Timing data for each container database row
                                 sychronization via PUTing.
===============================  ====================================================

Metrics for `container-updater`:

==============================  ====================================================
Metric Name                     Description
------------------------------  ----------------------------------------------------
`container-updater.successes`   Count of containers which successfully updated their
                                account.
`container-updater.failures`    Count of containers which failed to update their
                                account.
`container-updater.no_changes`  Count of containers which didn't need to update
                                their account.
`container-updater.timing`      Timing data for processing a container; only
                                includes timing for containers which needed to
                                update their accounts (i.e. "successes" and
                                "failures" but not "no_changes").
==============================  ====================================================

Metrics for `object-auditor`:

============================  ====================================================
Metric Name                   Description
----------------------------  ----------------------------------------------------
`object-auditor.quarantines`  Count of objects failing audit and quarantined.
`object-auditor.errors`       Count of errors encountered while auditing objects.
`object-auditor.timing`       Timing data for each object audit (does not include
                              any rate-limiting sleep time for
                              max_files_per_second, but does include rate-limiting
                              sleep time for max_bytes_per_second).
============================  ====================================================

Metrics for `object-expirer`:

========================  ====================================================
Metric Name               Description
------------------------  ----------------------------------------------------
`object-expirer.objects`  Count of objects expired.
`object-expirer.errors`   Count of errors encountered while attempting to
                          expire an object.
`object-expirer.timing`   Timing data for each object expiration attempt,
                          including ones resulting in an error.
========================  ====================================================

Metrics for `object-replicator`:

===================================================  ====================================================
Metric Name                                          Description
---------------------------------------------------  ----------------------------------------------------
`object-replicator.partition.delete.count.<device>`  A count of partitions on <device> which were
                                                     replicated to another node because they didn't
                                                     belong on this node.  This metric is tracked
                                                     per-device to allow for "quiescence detection" for
                                                     object replication activity on each device.
`object-replicator.partition.delete.timing`          Timing data for partitions replicated to another
                                                     node because they didn't belong on this node.  This
                                                     metric is not tracked per device.
`object-replicator.partition.update.count.<device>`  A count of partitions on <device> which were
                                                     replicated to another node, but also belong on this
                                                     node.  As with delete.count, this metric is tracked
                                                     per-device.
`object-replicator.partition.update.timing`          Timing data for partitions replicated which also
                                                     belong on this node.  This metric is not tracked
                                                     per-device.
`object-replicator.suffix.hashes`                    Count of suffix directories whose has (of filenames)
                                                     was recalculated.
`object-replicator.suffix.syncs`                     Count of suffix directories replicated with rsync.
===================================================  ====================================================

Metrics for `object-server`:

================================  ====================================================
Metric Name                       Description
--------------------------------  ----------------------------------------------------
`object-server.quarantines`       Count of objects (files) found bad and moved to
                                  quarantine.
`object-server.async_pendings`    Count of container updates saved as async_pendings
                                  (may result from PUT or DELETE requests).
`object-server.POST.errors`       Count of errors handling POST requests: bad request,
                                  missing timestamp, delete-at in past, not mounted.
`object-server.POST.timing`       Timing data for each POST request not resulting in
                                  an error.
`object-server.PUT.errors`        Count of errors handling PUT requests: bad request,
                                  not mounted, missing timestamp, object creation
                                  constraint violation, delete-at in past.
`object-server.PUT.timeouts`      Count of object PUTs which exceeded max_upload_time.
`object-server.PUT.timing`        Timing data for each PUT request not resulting in an
                                  error.
`object-server.GET.errors`        Count of errors handling GET requests: bad request,
                                  not mounted, header timestamps before the epoch.
                                  File errors resulting in a quarantine are not
                                  counted here.
`object-server.GET.timing`        Timing data for each GET request not resulting in an
                                  error.  Includes requests which couldn't find the
                                  object (including disk errors resulting in file
                                  quarantine).
`object-server.HEAD.errors`       Count of errors handling HEAD requests: bad request,
                                  not mounted.
`object-server.HEAD.timing`       Timing data for each HEAD request not resulting in
                                  an error.  Includes requests which couldn't find the
                                  object (including disk errors resulting in file
                                  quarantine).
`object-server.DELETE.errors`     Count of errors handling DELETE requests: bad
                                  request, missing timestamp, not mounted.  Includes
                                  requests which couldn't find or match the object.
`object-server.DELETE.timing`     Timing data for each DELETE request not resulting
                                  in an error.
`object-server.REPLICATE.errors`  Count of errors handling REPLICATE requests: bad
                                  request, not mounted.
`object-server.REPLICATE.timing`  Timing data for each REPLICATE request not resulting
                                  in an error.
================================  ====================================================

Metrics for `object-updater`:

============================  ====================================================
Metric Name                   Description
----------------------------  ----------------------------------------------------
`object-updater.errors`       Count of drives not mounted or async_pending files
                              with an unexpected name.
`object-updater.timing`       Timing data for object sweeps to flush async_pending
                              container updates.  Does not include object sweeps
                              which did not find an existing async_pending storage
                              directory.
`object-updater.quarantines`  Count of async_pending container updates which were
                              corrupted and moved to quarantine.
`object-updater.successes`    Count of successful container updates.
`object-updater.failures`     Count of failed continer updates.
============================  ====================================================

Metrics for `proxy-server` (in the table, `<type>` may be `Account`, `Container`,
or `Object`, and corresponds to the internal Controller object which handled the
request):

=========================================  ====================================================
Metric Name                                Description
-----------------------------------------  ----------------------------------------------------
`proxy-server.errors`                      Count of errors encountered while serving requests
                                           before the controller type is determined.  Includes
                                           invalid Content-Length, errors finding the internal
                                           controller to handle the request, invalid utf8, and
                                           bad URLs.
`proxy-server.<type>.errors`               Count of errors encountered after the controller
                                           type is known.  The details of which responses are
                                           errors depend on the controller type and request
                                           type (GET, PUT, etc.).  Failed
                                           authentication/authorization and "Not Found"
                                           responses are not counted as errors.
`proxy-server.<type>.client_timeouts`      Count of client timeouts (client did not read from
                                           queue within `client_timeout` seconds).
`proxy-server.<type>.client_disconnects`   Count of detected client disconnects.
`proxy-server.<type>.method_not_allowed`   Count of MethodNotAllowed responses sent by the
`proxy-server.<type>.auth_short_circuits`  Count of requests which short-circuited with an
                                           authentication/authorization failure.
`proxy-server.<type>.GET.timing`           Timing data for GET requests (excluding requests
                                           with errors or failed authentication/authorization).
`proxy-server.<type>.HEAD.timing`          Timing data for HEAD requests (excluding requests
                                           with errors or failed authentication/authorization).
`proxy-server.<type>.POST.timing`          Timing data for POST requests (excluding requests
                                           with errors or failed authentication/authorization).
                                           Requests with a client disconnect ARE included in
                                           the timing data.
`proxy-server.<type>.PUT.timing`           Timing data for PUT requests (excluding requests
                                           with errors or failed authentication/authorization).
                                           Account PUT requests which return MethodNotAllowed
                                           because allow_account_management is disabled ARE
                                           included.
`proxy-server.<type>.DELETE.timing`        Timing data for DELETE requests (excluding requests
                                           with errors or failed authentication/authorization).
                                           Account DELETE requests which return
                                           MethodNotAllowed because allow_account_management is
                                           disabled ARE included.
`proxy-server.Object.COPY.timing`          Timing data for object COPY requests (excluding
                                           requests with errors or failed
                                           authentication/authorization).
=========================================  ====================================================

Metrics for `tempauth` (in the table, `<reseller_prefix>` represents the actual configured
reseller_prefix or "`NONE`" if the reseller_prefix is the empty string):

=========================================  ====================================================
Metric Name                                Description
-----------------------------------------  ----------------------------------------------------
`tempauth.<reseller_prefix>.unauthorized`  Count of regular requests which were denied with
                                           HTTPUnauthorized.
`tempauth.<reseller_prefix>.forbidden`     Count of regular requests which were denied with
                                           HTTPForbidden.
`tempauth.<reseller_prefix>.token_denied`  Count of token requests which were denied.
`tempauth.<reseller_prefix>.errors`        Count of errors.
=========================================  ====================================================


------------------------
Debugging Tips and Tools
------------------------

When a request is made to Swift, it is given a unique transaction id.  This
id should be in every log line that has to do with that request.  This can
be useful when looking at all the services that are hit by a single request.

If you need to know where a specific account, container or object is in the
cluster, `swift-get-nodes` will show the location where each replica should be.

If you are looking at an object on the server and need more info,
`swift-object-info` will display the account, container, replica locations
and metadata of the object.

If you want to audit the data for an account, `swift-account-audit` can be
used to crawl the account, checking that all containers and objects can be
found.

-----------------
Managing Services
-----------------

Swift services are generally managed with `swift-init`. the general usage is
``swift-init <service> <command>``, where service is the swift service to 
manage (for example object, container, account, proxy) and command is one of:

==========  ===============================================
Command     Description
----------  -----------------------------------------------
start       Start the service
stop        Stop the service
restart     Restart the service
shutdown    Attempt to gracefully shutdown the service
reload      Attempt to gracefully restart the service
==========  ===============================================

A graceful shutdown or reload will finish any current requests before 
completely stopping the old service.  There is also a special case of 
`swift-init all <command>`, which will run the command for all swift services.

--------------
Object Auditor
--------------

On system failures, the XFS file system can sometimes truncate files it's
trying to write and produce zero byte files. The object-auditor will catch
these problems but in the case of a system crash it would be advisable to run
an extra, less rate limited sweep to check for these specific files. You can
run this command as follows:
`swift-object-auditor /path/to/object-server/config/file.conf once -z 1000`
"-z" means to only check for zero-byte files at 1000 files per second.

-------------
Swift Orphans
-------------

Swift Orphans are processes left over after a reload of a Swift server.

For example, when upgrading a proxy server you would probaby finish
with a `swift-init proxy-server reload` or `/etc/init.d/swift-proxy
reload`. This kills the parent proxy server process and leaves the
child processes running to finish processing whatever requests they
might be handling at the time. It then starts up a new parent proxy
server process and its children to handle new incoming requests. This
allows zero-downtime upgrades with no impact to existing requests.

The orphaned child processes may take a while to exit, depending on
the length of the requests they were handling. However, sometimes an
old process can be hung up due to some bug or hardware issue. In these
cases, these orphaned processes will hang around
forever. `swift-orphans` can be used to find and kill these orphans.

`swift-orphans` with no arguments will just list the orphans it finds
that were started more than 24 hours ago. You shouldn't really check
for orphans until 24 hours after you perform a reload, as some
requests can take a long time to process. `swift-orphans -k TERM` will
send the SIG_TERM signal to the orphans processes, or you can `kill
-TERM` the pids yourself if you prefer.

You can run `swift-orphans --help` for more options.


------------
Swift Oldies
------------

Swift Oldies are processes that have just been around for a long
time. There's nothing necessarily wrong with this, but it might
indicate a hung process if you regularly upgrade and reload/restart
services. You might have so many servers that you don't notice when a
reload/restart fails, `swift-oldies` can help with this.

For example, if you upgraded and reloaded/restarted everything 2 days
ago, and you've already cleaned up any orphans with `swift-orphans`,
you can run `swift-oldies -a 48` to find any Swift processes still
around that were started more than 2 days ago and then investigate
them accordingly.
