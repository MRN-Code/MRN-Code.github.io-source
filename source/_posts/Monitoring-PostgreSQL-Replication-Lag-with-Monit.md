title: Monitoring PostgreSQL Replication Lag with Monit
date: 2015-12-29 10:45:55
authorId: dwood
tags: postgres postgresql monit m/monit monitoring COINS3.0
---

For some time, we have been utilizing PostgreSQL's hot standby replication feature in both our staging and production environments. Currently, the hot standby serves three functions:

1. Standby server for maximum uptime if the master fails.
1. Disaster recovery if the master fails completely.
1. Read-only batch operations like taking nightly backups.

All three of these functions are critical to the safety of our data, so we need to be sure that the master and slave are properly communicating at all times. We use [Monit](https://mmonit.com/monit/)Monit and [M/Monit](https://mmonit.com) for most of our application and server monitoring. Monit is a daemon that runs on each of our servers, and performs checks at regular intervals. M/Monit is a centralized dashboard and alert service to which all of the Monit instances report. To help ensure that we get alerts even if our network is completely offline, our M/Monit host is hosted by AWS.

Because replication is so important, I have taken a belt and suspenders approach to monitoring the replication lag. This means that Monit is checking the replication status on both the master and the slave servers. The approach uses Monit's `check program` functionality to run a simple python script. If the script exits with an error (non-zero) status, then Monit will send an alert to our M/Monit server. M/Monit will then send emails and slack notifications to us.

On to the code:

### On the master server:

#### `/etc/monit/conf.d/pg-master-replication-check`
```
check program replication_check
        with path "/coins/pg-monitoring/master-replication-check.py"
        as uid "{{postgresql_service_user}}"
        and gid "webadmins"
    if status != 0 for 3 cycles then alert
```

### `/coins/pg-monitoring/master-replication-check.py`

This script queries the database to ascertain that it is in the right state (WAL streaming), and that the replication position reported by the slave is in line with that expected by the master.

```
#!/usr/bin/python
import subprocess

repLagBytesLimit = 128

try:
    repModeRes = subprocess.check_output('psql -t -p {{postgresql_port}} -c "SELECT state FROM pg_stat_replication"', shell=True)

    isInRepMode = repModeRes.strip() == 'streaming'

    repLagRes = subprocess.check_output('psql -t -p {{postgresql_port}} -c "SELECT pg_xlog_location_diff(sent_location, replay_location) FROM pg_stat_replication"', shell=True)

    repLagBytes = float(repLagRes)

except subprocess.CalledProcessError as e:
    print "Error retrieving stats: {0}".format(e)
    exit(1)

if isInRepMode != True:
    print ('Master server is not streaming to standby')
    exit(1)

if repLagBytes > repLagBytesLimit:
    print 'Slave replay is lagging behind by %f bytes' % repLagBytes
    exit(1)

print('All clear!')
exit(0)
```

### On the slave server

#### `/etc/monit/conf.d/pg-slave-replication-check`
```
check program replication_check
        with path "/coins/pg-monitoring/slave-replication-check.py"
        as uid "{{postgresql_service_user}}"
        and gid "webadmins"
    if status != 0 for 3 cycles then alert
```

#### `/coins/pg-monitoring/slave-replication-check.py`
This script queries the database to ascertain that it is in the right
state (recovery). It also queries the current xlog position **from the master**,
and compares it to the last reply location of the slave.

```
#!/usr/bin/python
import subprocess

slaveXlogDiffLimitBytes = 128

try:
    repModeRes = subprocess.check_output('psql -t -p {{postgresql_port}} -c "SELECT pg_is_in_recovery()"', shell=True)
    isInRepMode = repModeRes.strip() == 't'

    masterXlogLocationRes = subprocess.check_output('psql -t -p {{postgresql_port}} -h {{postgres_basebackup_host}} -U {{postgres_basebackup_user}} {{postgres_db_name}} -c "select pg_current_xlog_location();"', shell=True)
    masterXlogLocationStr = masterXlogLocationRes.strip()

    slaveXlogDiffRes = subprocess.check_output('psql -t -p {{postgresql_port}} {{postgres_db_name}} -c "select pg_xlog_location_diff(pg_last_xlog_replay_location(), \'' + masterXlogLocationStr + '\'::pg_lsn);"', shell=True)
    slaveXlogDiffBytes = float(slaveXlogDiffRes.strip())
except subprocess.CalledProcessError as e:
    print "Error retrieving stats: {0}".format(e)
    exit(1)

if isInRepMode != True:
    print ('Slave server is not in recovery mode')
    exit(1)

if slaveXlogDiffBytes > slaveXlogDiffLimitBytes:
    print "Slave server replication is behind master by %f bytes" % slaveXlogDiffBytes
    exit(1)

print('All clear!')
exit(0)
```

You may wonder why I chose python instead of Bash or my usual favorite: Node.js. Python is installed in our base server image, while Node is not, and I want to keep out database servers as _stock_ as possible. I chose python over bash because I find that bash scripts are brittle and difficult to debug.
