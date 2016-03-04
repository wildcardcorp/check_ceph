# check_ceph.py

This is an Icinga compatible plugin for monitoring ceph. It should work with Nagios based products as well. Overall it works well, i've noticed a couple time when a result came back unknown or warning when it shouldn't have, but it shortly corrected itself. I'm still working on figuring what what cluster/osd/pg states are causing that.

It does many things such as:

 - Check OSD status. You set the warning and critical values at how many OSD not up and in you want to be warned at.
 - General health status. Like the HEALTH_OK, HEALTH_WARN you see with ceph status
 - PG Status. It doesn't take a warning or critical yet, but it goes to warning if there are PGs not active+clean.
 - Performance metrics. No warning or critical on this either. It is for gathering perfdata for graphing. It currently captures IOPS and Read/Write bytes/sec. Most other checks report perfdata as well. These are simply metrics that can't be gathered through the other commands.
 - Disk space. It either alerts on a single pool or all pools. If you pass it --pool it will use only that pool to alert on, without --pool it is all pools. Warning and critical are in GB/TB free in that pool. This is based on the Max Avail fields you get with `ceph df`. So it is not raw free space but based on pool replication size. This captures disk space metrics for all pools and global free space




## Icinga Integration

You need to create a ceph user and get a keyring. The user only needs  `caps: [mon] allow r` permissions. Name it whatever you like.

I used this CheckCommand object in my icinga configuration.
```sh
object CheckCommand "check_ceph" {
  import "plugin-check-command"
  command = [PluginDir + "/check_ceph.py"]
  timeout = 20
  arguments = {
    "-c" = "$conf$"
    "--conf" = "$conf$"
    "--id" = "$id$"
    "-k" = "$keyring$"
    "--keyring" = "$keyring$"
    "--health" = {
      set_if = "$health$"
    }
    "--osd" = {
       set_if = "$osd$"
    }
    "--pg" = {
     set_if = "$pg$"
    }
    "--df" = {
      set_if = "$df$"
    }
    "--perf" = {
      set_if = "$perf$"
    }
    "--pool" = "$pool$"
    "-b" = "$byte$"
    "--byte" = "$byte$"
    "-w" = "$warning$"
    "--warning" = "$warning$"
    "-c" = "$critical$"
    "--critical" = "$critical$"
  }
}
```
The services are applied like...
```sh
apply Service "ceph_health" {
  import "generic-5m-service"
  display_name = "Ceph Health"
  check_command = "check_ceph"
  vars.conf = "/etc/icinga2/ceph/ceph.conf"
  vars.id = "icinga"
  vars.keyring = "/etc/icinga2/ceph/ceph.client.icinga.keyring"
  vars.health = true
  vars.notification.mute = true
  assign where host.vars.type.contains("ceph-mon")
}
```

## Usage

```sh
usage: check_ceph.py [-h] [-C CONF] -id ID [-k KEYRING] [--health] [-o]
                     [-m MON] [-p] [--perf] [--df] [-b BYTE] [--pool POOL]
                     [--objects OBJECTS] [-w WARNING] [-c CRITICAL]

Runs health checks against a ceph cluster. This is designed to run on the
monitoring server using the ceph client software. Supply a ceph.conf, keyring,
and user to access the cluster.

optional arguments:
  -h, --help            show this help message and exit
  -C CONF, --conf CONF  ceph.conf file, defaults to /etc/ceph/ceph.conf.
  -id ID, --id ID       Ceph authx user
  -k KEYRING, --keyring KEYRING
                        Path to ceph keyring if not in
                        /etc/ceph/client.\$id.keyring
  --health              Get general health status. ex. HEALTH_OK, HEALTH_WARN
  -o, --osd             OSD status. Thresholds are in number of OSDs missing
  -m MON, --mon MON     MON status. Thesholds are in number of mons missing
  -p, --pg              PG status. No thresholds due to the large number of pg
                        states.
  --perf                collects additional ceph performance statistics
  --df                  Disk/cluster usage. Reports global and all pools
                        unless --pool is used. Warning and critical are number
                        of -b free to the pools. This is not Raw Free, but Max
                        Avail to the pools based on rep or k,m settings. If
                        you do not define a pool the threshold is run agains
                        all the pools in the cluster.
  -b BYTE, --byte BYTE  Format to use for displaying DF data. G=Gigabyte,
                        T=Terabyte. Use with the --df option. Defults to TB
  --pool POOL           Pool. Use with df
  --objects OBJECTS     Object counts based on pool
  -w WARNING, --warning WARNING
                        Warning threshold. See specific checks for value types
  -c CRITICAL, --critical CRITICAL
                        Critical threshold. See specific checks for value
                        types
```

## Examples

```sh
./check_ceph.py -C /etc/icinga2/ceph/ceph.conf --id icinga -k /etc/icinga2/ceph/ceph.client.icinga.keyring --osd -w 2 -c 3
ALL OSDs are up and in. 264 OSDS. 264 up, 264 in|num_osds=264 num_up_osds=264 num_in_osds=264
./check_ceph.py -C /etc/icinga2/ceph/ceph.conf --id icinga -k /etc/icinga2/ceph/ceph.client.icinga.keyring --health
HEALTH_OK
./check_ceph.py -C /etc/icinga2/ceph/ceph.conf --id icinga -k /etc/icinga2/ceph/ceph.client.icinga.keyring --pg
All PGs are active+clean: 20480 PGs Total, active+clean=20480 |active+clean=20480
./check_ceph.py -C /etc/icinga2/ceph/ceph.conf --id icinga -k /etc/icinga2/ceph/ceph.client.icinga.keyring --pg
All PGs are active+clean: 20480 PGs Total, active+clean=20480 |active+clean=20480
./check_ceph.py -C /etc/icinga2/ceph/ceph.conf --id icinga -k /etc/icinga2/ceph/ceph.client.icinga.keyring --df -w 100 -c 50
Healthy: All ceph pools are within free space thresholds|global_total_bytes=1699TB global_used_bytes=1179TB global_avail_bytes=520TB dev_bytes_used=756TB dev_max_avail=270TB dev_objects=6995252 ops_bytes_used=183TB ops_max_avail=270TB ops_objects=2817297
```

