### Pgbackrest

- setup inventory file
- tuning sysctl on master host: `sysctl -w net.ipv4.ip_forward=1` and `sysctl -w net.ipv4.conf.all.forwarding=1`
- add route on replica1 replica2 192.168.56.0/24 via 192.168.57.10: `ip ro add 192.168.56.0/24 via 192.168.57.10`
- install this depend on all hosts: `ansible all -m yum -a "name=libselinux-python3 state=latest" -b` [problem](https://stackoverflow.com/questions/63173955/aborting-target-uses-selinux-but-python-bindings-libselinux-python-arent-ins)
- on all node add env for user postgres: `export+=":/usr/local/bin"`, need for ansible this PATH must be present in `/var/lib/pgsql/.bash_profile`
- 

###### [pgBackRest](https://pgbackrest.org)

Well, while vitabaks with some project iam trying setup in manual method:

- install dependencies pgbackrest on all nodes
- with patroni setup and provision config postgres.conf and recovery.conf (and may be restart cluster)
- init create repo pgbackrest
- 
- log in to master and make sure if you superuser (sudo -i) and enter command: `sudo su postgres`
- edit config cluster with: `patronictl edit-config`:

```bash
bash-4.2$ /usr/local/bin/patronictl edit-config
--- 
+++ 
@@ -3,8 +3,8 @@
 maximum_lag_on_failover: 1048576
 postgresql:
   parameters:
-    archive_command: cd .
+    archive_command: 'pgbackrest --stanza=postgres-otus-demo archive-push %p'
-    archive_mode: true
+    archive_mode: on
     archive_timeout: 1800s
     auto_explain.log_analyze: true
     auto_explain.log_buffers: true

Apply these changes? [y/N]: y
Configuration changed
bash-4.2$ /usr/local/bin/patronictl list
+ Cluster: postgres-cluster ---------+---------+----+-----------+
| Member   | Host          | Role    | State   | TL | Lag in MB |
+----------+---------------+---------+---------+----+-----------+
| master   | 192.168.57.10 | Replica | running |  4 |         0 |
| replica1 | 192.168.57.11 | Leader  | running |  4 |           |
| replica2 | 192.168.57.12 | Replica | running |  4 |         0 |
+----------+---------------+---------+---------+----+-----------+
```

- pgbasrest initialize repo1 and check:

```bash
bash-4.2$ pgbackrest --stanza=postgres-otus-demo --log-level-console=info stanza-create
2023-02-03 21:59:14.590 P00   INFO: stanza-create command begin 2.44: --exec-id=15635-7d35bb2d --log-level-console=info --pg1-path=/var/lib/pgsql/15/data --repo1-path=/var/lib/pgbackrest --stanza=postgres-otus-demo
2023-02-03 21:59:15.251 P00   INFO: stanza-create for stanza 'postgres-otus-demo' on repo1
2023-02-03 21:59:15.282 P00   INFO: stanza-create command end: completed successfully (694ms)
bash-4.2$ pgbackrest --stanza=postgres-otus-demo --log-level-console=info check
2023-02-03 22:00:22.172 P00   INFO: check command begin 2.44: --exec-id=15704-5ead42e0 --log-level-console=info --pg1-path=/var/lib/pgsql/15/data --repo1-path=/var/lib/pgbackrest --stanza=postgres-otus-demo
2023-02-03 22:00:22.788 P00   INFO: check repo1 configuration (primary)
2023-02-03 22:00:23.361 P00   INFO: check repo1 archive for WAL (primary)
2023-02-03 22:00:23.863 P00   INFO: WAL segment 000000040000000000000007 successfully archived to '/var/lib/pgbackrest/archive/postgres-otus-demo/15-1/0000000400000000/000000040000000000000007-ec89bded2dcecfbd3dcfcf2e88d4a7cdfe835259.gz' on repo1
2023-02-03 22:00:23.863 P00   INFO: check command end: completed successfully (1693ms)
```

- lets make backup:

```bash
bash-4.2$ pgbackrest --stanza=postgres-otus-demo --log-level-console=info backup
2023-02-03 22:06:17.443 P00   INFO: backup command begin 2.44: --exec-id=16080-56a8245e --log-level-console=info --pg1-path=/var/lib/pgsql/15/data --repo1-path=/var/lib/pgbackrest --stanza=postgres-otus-demo
WARN: option 'repo1-retention-full' is not set for 'repo1-retention-full-type=count', the repository may run out of space
      HINT: to retain full backups indefinitely (without warning), set option 'repo1-retention-full' to the maximum.
WARN: no prior backup exists, incr backup has been changed to full
2023-02-03 22:06:18.164 P00   INFO: execute non-exclusive backup start: backup begins after the next regular checkpoint completes
2023-02-03 22:06:19.318 P00   INFO: backup start archive = 000000040000000000000009, lsn = 0/9000060
2023-02-03 22:06:19.318 P00   INFO: check archive for prior segment 000000040000000000000008
2023-02-03 22:06:25.956 P00   INFO: execute non-exclusive backup stop and wait for all WAL segments to archive
2023-02-03 22:06:26.258 P00   INFO: backup stop archive = 000000040000000000000009, lsn = 0/9000138
2023-02-03 22:06:26.268 P00   INFO: check archive for segment(s) 000000040000000000000009:000000040000000000000009
2023-02-03 22:06:26.816 P00   INFO: new backup label = 20230203-220618F
2023-02-03 22:06:26.902 P00   INFO: full backup size = 22.5MB, file total = 971
2023-02-03 22:06:26.902 P00   INFO: backup command end: completed successfully (9462ms)
2023-02-03 22:06:26.902 P00   INFO: expire command begin 2.44: --exec-id=16080-56a8245e --log-level-console=info --repo1-path=/var/lib/pgbackrest --stanza=postgres-otus-demo
2023-02-03 22:06:26.915 P00   INFO: option 'repo1-retention-archive' is not set - archive logs will not be expired
2023-02-03 22:06:26.915 P00   INFO: expire command end: completed successfully (13ms)
```

- make a difference backup:

```bash
bash-4.2$ pgbackrest --stanza=postgres-otus-demo --type=diff --log-level-console=info backup
2023-02-03 22:10:20.712 P00   INFO: backup command begin 2.44: --exec-id=16328-b164b292 --log-level-console=info --pg1-path=/var/lib/pgsql/15/data --repo1-path=/var/lib/pgbackrest --stanza=postgres-otus-demo --type=diff
WARN: option 'repo1-retention-full' is not set for 'repo1-retention-full-type=count', the repository may run out of space
      HINT: to retain full backups indefinitely (without warning), set option 'repo1-retention-full' to the maximum.
2023-02-03 22:10:21.475 P00   INFO: last backup label = 20230203-220618F, version = 2.44
2023-02-03 22:10:21.475 P00   INFO: execute non-exclusive backup start: backup begins after the next regular checkpoint completes
2023-02-03 22:10:22.620 P00   INFO: backup start archive = 00000004000000000000000B, lsn = 0/B000028
2023-02-03 22:10:22.620 P00   INFO: check archive for prior segment 00000004000000000000000A
2023-02-03 22:10:23.574 P00   INFO: execute non-exclusive backup stop and wait for all WAL segments to archive
2023-02-03 22:10:23.880 P00   INFO: backup stop archive = 00000004000000000000000B, lsn = 0/B000100
2023-02-03 22:10:23.891 P00   INFO: check archive for segment(s) 00000004000000000000000B:00000004000000000000000B
2023-02-03 22:10:24.264 P00   INFO: new backup label = 20230203-220618F_20230203-221021D
2023-02-03 22:10:24.362 P00   INFO: diff backup size = 8.3KB, file total = 971
2023-02-03 22:10:24.362 P00   INFO: backup command end: completed successfully (3652ms)
2023-02-03 22:10:24.362 P00   INFO: expire command begin 2.44: --exec-id=16328-b164b292 --log-level-console=info --repo1-path=/var/lib/pgbackrest --stanza=postgres-otus-demo
2023-02-03 22:10:24.375 P00   INFO: option 'repo1-retention-archive' is not set - archive logs will not be expired
2023-02-03 22:10:24.375 P00   INFO: expire command end: completed successfully (13ms)
```

- backups information:

```bash
bash-4.2$ pgbackrest info
stanza: postgres-otus-demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (15): 000000040000000000000007/00000004000000000000000B

        full backup: 20230203-220618F
            timestamp start/stop: 2023-02-03 22:06:18 / 2023-02-03 22:06:26
            wal start/stop: 000000040000000000000009 / 000000040000000000000009
            database size: 22.5MB, database backup size: 22.5MB
            repo1: backup set size: 3MB, backup size: 3MB

        diff backup: 20230203-220618F_20230203-221021D
            timestamp start/stop: 2023-02-03 22:10:21 / 2023-02-03 22:10:23
            wal start/stop: 00000004000000000000000B / 00000004000000000000000B
            database size: 22.5MB, database backup size: 8.3KB
            repo1: backup set size: 3MB, backup size: 437B
            backup reference list: 20230203-220618F
```

### Demo backup and restore

- install pgbackrest: `yum install pgbackrest` now it v2.44 (ansible playbook if need:`ansible all -b -m yum -a "name=pgbackrest state=latest"`)
- configurable with script playbook

#### Backup



#### Restore