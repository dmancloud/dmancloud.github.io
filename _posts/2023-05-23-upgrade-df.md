---
layout: post
title: "Deepfactor upgrade from 3.2.x to 3.3"
date: 2023-05-23 22:00:00 -0500
categories:
tags:
#image:
#  path: /assets/img/headers/mirror-image-chess.jpg
---

## Before upgrade

Stop instrumented apps (stop all active apps that are sending telemetry)

### Step 1: Stop primary Deepfactor services

Set the version-build that you will be upgrading to.  Example: 3.3.0-2152
``` sh
export DF_RELEASE_NAME=df-stable
export DF_VERSION_BUILD=3.3.0-2152
```
``` sh
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-nginx -n deepfactor --replicas=0
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-apisvc -n deepfactor --replicas=0
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-alertstreamsvc -n deepfactor --replicas=0
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-eventsvc -n deepfactor --replicas=0
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-dfwebscan -n deepfactor --replicas=0
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-persistentsvc -n deepfactor --replicas=0
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-notificationsvc -n deepfactor --replicas=0
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-staticscaneventprocessor -n deepfactor --replicas=0
```
``` sh
curl -L https://staging-df-helm-charts.s3.us-west-2.amazonaws.com/scripts/public/df3.2_to_3.3_scripts_$DF_VERSION_BUILD.tar.gz -o df3.2_to_3.3_scripts.tar.gz
```
``` sh
kubectl cp df3.2_to_3.3_scripts.tar.gz deepfactor/$DF_RELEASE_NAME-deepfactor-clickhouse-0:var/lib/clickhouse/
kubectl cp df3.2_to_3.3_scripts.tar.gz deepfactor/$DF_RELEASE_NAME-deepfactor-postgres-0:var/lib/postgresql/
```
* Wait 3 minutes for pods to stop and for buffer tables to write to base tables

### Step 2: Backup

Backup the existing clickhouse and postgres databases. This is needed if the upgrade fails and needs to be rolled back.
Store the backup files to an external location (not in pod or pvc).

Clickhouse backup tool installation:

``` sh
curl -L https://github.com/AlexAkulov/clickhouse-backup/releases/download/v2.2.5/clickhouse-backup-linux-amd64.tar.gz -o clickhouse-backup-linux-amd64.tar.gz
```
``` sh
kubectl cp clickhouse-backup-linux-amd64.tar.gz deepfactor/$DF_RELEASE_NAME-deepfactor-clickhouse-0:var/lib/clickhouse/
kubectl exec -it $DF_RELEASE_NAME-deepfactor-clickhouse-0  -n deepfactor -- bash
```
``` sh
su - clickhouse
```
``` sh
tar -xf df3.2_to_3.3_scripts.tar.gz
tar -xf clickhouse-backup-linux-amd64.tar.gz
mv build/linux/amd64/clickhouse-backup .
rm -rf build/
CLICKHOUSE_PASSWORD=$(cat /etc/clickhouse-server/users.xml | grep '<password>' | sed -n 2p | awk -F "[<>]" '{print $3}')
cp scripts/3.2_to_3.3_manual/ch_3.2_to_3.3/config.yml config.yml
sed -i 's/your_ch_password/'"$CLICKHOUSE_PASSWORD"'/' config.yml
```
Clickhouse pre-backup verification:

Ensure the total_rows match for the pairs of Buffer/MergeTree tables.  If they don't match, probably some app is still running.  Identify and stop.
```sh

clickhouse-client --database=deepfactor --host=localhost --password=$CLICKHOUSE_PASSWORD
```
``` sql
select name, engine, total_rows, total_bytes from `system`.tables where engine = 'Buffer' or name in (
'dependency_info_events',
'hb_events',
'lsa_class_loaded_events',
'lsa_method_invoked_events',
'os_pkg_events',
'ot_events',
'resource_meta',
'rt_api_events',
'rt_env_events',
'rt_file_events',
'rt_id_change_events',
'rt_mw_events',
'rt_ne_events',
'rt_process_events',
'scan_metadata_events',
'vulnerabilities_meta',
'web_uri_events'
);

exit
```
Execute the backup:
``` sh
./clickhouse-backup create -c config.yml
./clickhouse-backup list -c config.yml
tar -czvf dfchbackup32.tar.gz backup
exit
```
``` sh
kubectl cp deepfactor/$DF_RELEASE_NAME-deepfactor-clickhouse-0:var/lib/clickhouse/dfchbackup32.tar.gz dfchbackup32.tar.gz
```
PostgreSQL backup:
``` sh
kubectl exec -it $DF_RELEASE_NAME-deepfactor-postgres-0 -n deepfactor -- bash
```
``` sh
su - postgres
pg_dump deepfactor > df32.sql
tar -czvf dfpgbackup32.tar.gz df32.sql
exit
```
``` sh
kubectl cp  deepfactor/$DF_RELEASE_NAME-deepfactor-postgres-0:var/lib/postgresql/dfpgbackup32.tar.gz dfpgbackup32.tar.gz
```
### Step 3: Pre-upgrade:
``` sh
kubectl exec -it $DF_RELEASE_NAME-deepfactor-postgres-0  -n deepfactor -- bash
```
``` sh
su - postgres
tar -xf df3.2_to_3.3_scripts.tar.gz
psql -v ON_ERROR_STOP=1 -U postgres -d deepfactor -f ~/scripts/3.2_to_3.3_manual/pg_3.2_to_3.3.sql
pg_dump deepfactor -n df --data-only > df32_33_data.sql
tar -czvf df32_33_data.tar.gz df32_33_data.sql
exit
exit
```
``` sh
kubectl cp deepfactor/$DF_RELEASE_NAME-deepfactor-postgres-0:var/lib/postgresql/df32_33_data.tar.gz backup/df32_33_data.tar.gz
```
``` sh
kubectl exec -it $DF_RELEASE_NAME-deepfactor-clickhouse-0 -n deepfactor -- bash
```
``` sh
su - clickhouse
tar -xf df3.2_to_3.3_scripts.tar.gz
CLICKHOUSE_PASSWORD=$(cat /etc/clickhouse-server/users.xml | grep '<password>' | sed -n 2p | awk -F "[<>]" '{print $3}')
cd ~/scripts/3.2_to_3.3_manual/ch_3.2_to_3.3
./ch_3.2_to_3.3.sh preupgrade
```
* Verify there are only 33 "_old" tables:
``` sh
clickhouse-client --database=deepfactor --host=localhost --password=$CLICKHOUSE_PASSWORD
```
``` sh
select name, total_rows from system.tables where database='deepfactor';
exit
```
* Create 3.3 schema:
``` sh
cd ~/scripts
sed -i 's/DROP DATABASE/--DROP DATABASE/' CH_Schema_v3_3.sql
sed -i 's/CREATE DATABASE/--CREATE DATABASE/' CH_Schema_v3_3.sql
sed -i 's/TTTTT/180/g' CH_Schema_v3_3.sql
cat CH_Schema_v3_3.sql | clickhouse-client --database=deepfactor --host=localhost --password=$CLICKHOUSE_PASSWORD -m -n
```
### Step 4: Upgrade (continue in same terminal window/session)
``` sh
cd ~/scripts/3.2_to_3.3_manual/ch_3.2_to_3.3
./ch_3.2_to_3.3.sh postupgrade
exit
exit
```
* PostgreSQL upgrade - delete PGSQL 11:
``` sh
kubectl scale sts $DF_RELEASE_NAME-deepfactor-postgres -n deepfactor --replicas=0
kubectl delete pvc postgres-pvc-$DF_RELEASE_NAME-deepfactor-postgres-0 -n deepfactor
```
### Step 5: Portal services upgrade (instructions omitted as it is the same process as usual - helm upgrade)

* Wait for    df-stable-deepfactor-postgres-0 to be "Running"
* df-stable-deepfactor-dfstartup-xxxxx, df-stable-deepfactor-migrationsvc-xxxxx, and df-stable-deepfactlr-policyctl-xxxxx to all be "Completed"
* Scale down the following services:
``` sh
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-nginx -n deepfactor --replicas=0
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-apisvc -n deepfactor --replicas=0
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-alertstreamsvc -n deepfactor --replicas=0
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-eventsvc -n deepfactor --replicas=0
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-authsvc -n deepfactor --replicas=0
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-dfwebscan -n deepfactor --replicas=0
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-persistentsvc -n deepfactor --replicas=0
kubectl scale deploy $DF_RELEASE_NAME-deepfactor-notificationsvc -n deepfactor --replicas=0
```
* Drop default PG14 DB and recreate:
``` sh
kubectl cp df3.2_to_3.3_scripts.tar.gz deepfactor/$DF_RELEASE_NAME-deepfactor-postgres-0:/var/lib/postgresql/
kubectl cp backup/df32_33_data.tar.gz deepfactor/$DF_RELEASE_NAME-deepfactor-postgres-0:var/lib/postgresql/
```
``` sh
kubectl exec -it $DF_RELEASE_NAME-deepfactor-postgres-0  -n deepfactor -- bash
```
``` sh
su - postgres
tar -xf df3.2_to_3.3_scripts.tar.gz
tar -xvf df32_33_data.tar.gz
sed -i 's/CCCCC/1/g' scripts/dfSchema_v3_3_partitioned_customerId.sql
psql -v ON_ERROR_STOP=1 -U postgres -f scripts/dfSchema_v3_3_partitioned.sql
psql -v ON_ERROR_STOP=1 -U postgres -d deepfactor -f scripts/dfSchema_v3_3_partitioned_customerId.sql
psql -v ON_ERROR_STOP=1 -U postgres -d deepfactor < df32_33_data.sql
exit
exit
```
### Step 6: Re-run upgrade (helm upgrade) again to run the policyctl jobs and other upgrade scripts)

## UPGRADE COMPLETE

### Restore
This section is provided only to help to restore the backup in the event of errors during upgrade.

Clickhouse

Copy the clickhouse backup file dfchbackup32.tar.gz to the clickhouse pod
``` sh
kubectl cp dfchbackup32.tar.gz deepfactor/$DF_RELEASE_NAME-deepfactor-clickhouse-0:var/lib/clickhouse/dfchbackup32.tar.gz
sh/bash into clickhouse pod
```
``` sh
kubectl exec -it $DF_RELEASE_NAME-deepfactor-clickhouse-0  -n deepfactor -- bash
```
``` sh
su - clickhouse
tar -xf dfchbackup32.tar.gz backup
```
Follow instructions in Backup section to install clickhouse-backup tool

Verify the backup is available
``` sh
./clickhouse-backup list -c config.yml
```
Restore
``` sh
./clickhouse-backup restore -c config.yml <backup name as shown in the list>
```