title: 'How to [partially] retreat from a database upgrade'
date: 2015-06-21 18:00:17
tags:
authorId: 'dwood'
---

This post has turned into a bit of a long story.
If you are just looking for how to perform a `pg_restore` from a newer version of PostgreSQL to an older version of PostgreSQL, look down toward the bottom.

We recently upgraded our worn-out PostgreSQL 8.4 database running on a Cents 5.5 VM to a shiny new PostgreSQL 9.4 database on top of Ubuntu 14.04.
During a three week testing period, we encountered and fixed a coulple of upgrade-induced bugs in our staging environment.
At the end of three weeks of testing, we felt confident that the upgrade would go smoothly in production... and it did (mostly).

The day after the upgrade, users started to submit tickets complaining that our data export tool was running very slowly in some cases, and just hanging in other cases.
Myself and two other engineers spent the next day and a half benchmarking the new database servers over and over, and looking at `Explain Analyze` plans.
Eventually, we convinced ourselves that the issue was not with the underlying virtual machine, or the OS, but with our configuration of postgres.

To better debug, we restarted our old database server, and ran the offending queries there as well as in the new server in our staging environment.
We were able to gain some insights into the issue by comparing the `Explain Analyze` output from both servers:
The new database was not using the same indices that the old database was. This resulted in more nested loops and analyzing more rows than necessary.

By increasing the `random_page_cost` from 4 to 15, we were able to get the query explain plans to look more similar, but performance did not improve.
The new database was still choosing different indices to scan.

At this point, our users had been without a useful query-building-export tool for two business days, so it was time to switch tactics and implement a work-around solution.
I decided that it would be easiest to direct the queries used by our export tool to a copy of our old production database.
We would be able to keep the copy relatively up to date by loading nightly backups from our production infrastructure.

Modifying the application layer to send requests to the old database server was trivial, since there was a dedicated endpoint just for the low-performig export tool.
Getting the old database to refresh from a backup of the new database was a little trickier. 

First, I set up a cron job to run a `pg_dump` on our hot standby database server every night, and store the dump on our network storage.
I have always used the *custom* format (`-Fc`) for pg_dumps, as they allow a lot of flexibility when performing the restore.
This was not an option in this case because I received the following error when trying to restore on the PG 8.4 server: `pg_restore: [archiver] unsupported version (1.12) in file header`.

My initial attempts to circumvent this included running the pg_dump of the new database remotely from the old database server unsuccessfully, and attempting to upgrade only *postgres-contrib* on the old database server.
Neither of these solutions worked out, so I decided to use the *plain* pg_dump format (`-Fp`). This outputs plain SQL statements to rebuild the schema and data. 
There are still a few errors during the restore, because the `CREATE EXTENSION` functionality does not exist in PG 8.4, but I can simply rebuild the necessary extensions manually after the rebuild.

To reduce the time taken by the dump and restore process, I only dump the schema used by the export tool.
In addition, I omit all history tables (a construct we use to track changes made to data in the database) and some of the larger tables not used by the query tool. 
This also reduces the size of the restored database considerably, and allows me to restore into a temporary database while the primary database is still running, allowing for near-zero downtime.

A simplified diagram of the current system is shown below:
{% asset_img QBPG84.png %}

Here is the cron task that dumps the data. This is placed in its own file in `/etc/cron.d`
```
30 23 * * * postgres pg_dump -vFp -p 6117 -T 'mrsdba.mrs_series_data' -T '*_hist'  -T mrsdba.mrs_series  -T mrsdba.mrs_analysis_files -T mrsdba.mrs_assessment_events -T mrsdba.mrs_asmt_resp_hist_new -n 'mrsdba' postgres > '/coins/mn    t/ni/prodrepdbcoin.sql' > /tmp/rep_dump.log 2>&1
```

Here is the script that creates a new Postgres 8.4 DB from the dump of the Postgres 9.4 database.
```

# Author: Dylan Wood
# Date: June.20.2015
# Script is called from /etc/cron.d/coins_query_restore

echo 'Starting COINS DB refresh as user:'
echo `id`
echo `date`

# Define variables
ARCHIVE_DIR="/export/ni/prodrepdbcoin.sql"
ARCHIVE_FILE=`ls -1t $ARCHIVE_DIR | head -1`
DBNAME="postgres"
DBPORT=6117

# Create temp database
# Create empty DB
echo 'Creating empty DB'
createdb -p $DBPORT ${DBNAME}_temp

# Create lang
echo 'create plpgsql lang'
psql -p $DBPORT -d ${DBNAME}_temp -c 'CREATE LANGUAGE plpgsql'

# Restore DB
echo 'restoring db from latest dump'
psql -p $DBPORT -d ${DBNAME}_temp -f $ARCHIVE_FILE

# Edit default search path
echo 'Setting default search path'
psql -p $DBPORT -d ${DBNAME}_temp -c "ALTER DATABASE ${DBNAME}_temp SET search_path=mrsdba, casdba, public;"

# Truncate qb temp tables
echo 'Truncating QB temp tables'
psql -p $DBPORT -d ${DBNAME}_temp -c "TRUNCATE TABLE mrsdba.mrs_qb_asmt_data_sort_temp; TRUNCATE TABLE mrsdba.mrs_qb_asmt_pivot_categories_temp;"

# Add empty schemas
echo 'Adding casdba, dxdba and dtdba'
psql -p $DBPORT -d ${DBNAME}_temp -c "CREATE schema casdba; CREATE schema dxdba; CREATE schema dtdba;"

# Create tablefunc extension
echo 'Create tablefunc extension'
psql -p $DBPORT -d ${DBNAME}_temp -f /usr/share/pgsql/contrib/tablefunc.sql

# VACUUM ANALYZE THE DB
echo 'VACUUM ANALYZE'
psql -p $DBPORT -d ${DBNAME}_temp -c "VACUUM ANALYZE"

# Drop database

# First, disconnect all connections
echo 'Terminating connections to DB'
psql -d $DBNAME -p $DBPORT -c "SELECT pg_terminate_backend(procpid) FROM pg_stat_activity WHERE procpid <> pg_backend_pid() AND datname = '$DBNAME';"

# Drop DB
echo 'Dropping DB'
dropdb -p $DBPORT $DBNAME

# Rename temp DB
echo 'Renaming temp database'
psql -d habaridb -p $DBPORT -c "ALTER DATABASE ${DBNAME}_temp RENAME TO ${DBNAME};"

echo 'Finished with COINS DB refresh'
echo `date`
```

