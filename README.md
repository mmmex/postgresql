## Postgresql

Задачи:

- [X] настроить hot_standby репликацию с использованием слотов
- [X] настроить правильное резервное копирование
- [X] Для сдачи работы присылаем ссылку на репозиторий, в котором должны обязательно быть
  - [Vagranfile (2 машины)](/Vagrantfile)
  - [плейбук Ansible](/ansible/provision.yml)
  - конфигурационные файлы [postgresql.conf](/ansible/roles/postgres-cluster/templates/postgresql.conf.j2), [pg_hba.conf](/ansible/roles/postgres-cluster/templates/pg_hba.conf.j2) и [recovery.conf](#примечания), конфиг barman, либо скрипт резервного копирования.
- [X] Команда "vagrant up" должна поднимать машины с настроенной репликацией и резервным копированием.
- [X] Рекомендуется в README.md файл вложить результаты (текст или скриншоты) проверки работы репликации и резервного копирования.

### Запуск проекта

1. Клонируем репозиторий: `git clone ...`
2. Выполняем вход в папку: `cd postgresql`
3. Запускаем проект: `vagrant up`

Будет поднято 3 ВМ с настроенной репликацией и резервным копированием. Резервные копии будут хранится на `backup` в каталоге `/var/lib/pgbackrest`. В качестве ПО резервного копирования выбрана `pgbackrest`.

Хост     | IP-адрес      | Роль                  | Файлы конфигурации
---------|---------------|-----------------------|--------------------
master   | 192.168.57.10 | postgresql-master     | [postgresql.conf](/ansible/roles/postgres-cluster/templates/postgresql.conf.j2), [pg_hba.conf](/ansible/roles/postgres-cluster/templates/pg_hba.conf.j2), [pgbackrest.conf](/ansible/roles/pgbackrest/templates/master.conf.j2)
replica1 | 192.168.57.11 | postgresql-hot_stanby | [postgresql.conf](/ansible/roles/postgres-cluster/templates/postgresql.conf.j2), [pg_hba.conf](/ansible/roles/postgres-cluster/templates/pg_hba.conf.j2), [pgbackrest.conf](/ansible/roles/pgbackrest/templates/replica1.conf.j2)
backup   | 192.168.57.12 | backup server         | [pgbackrest.conf](/ansible/roles/pgbackrest/templates/backup.conf.j2)

В списке `pgbackrest_cron` в файле [ansible/vars/main.yml](/ansible/vars/main.yml), задаются задачи по резервному копированию для cron.

### Резервное копирование, основные термины

**Резервная копия (Backup)** — это непротиворечивая копия кластера базы данных, которую можно восстановить для восстановления после сбоя оборудования, для выполнения восстановления на определенный момент времени или для запуска нового резервного.

**Полное резервное копирование (Full Backup)**: pgBackRest копирует все содержимое кластера базы данных в резервную копию. Первая резервная копия кластера базы данных всегда является полной резервной копией. pgBackRest всегда может напрямую восстановить полную резервную копию. Полная резервная копия не зависит от каких-либо файлов за пределами полной резервной копии для согласованности.

**Дифференциальное резервное копирование (Differential Backup)**: pgBackRest копирует только те файлы кластера базы данных, которые изменились с момента последнего полного резервного копирования. pgBackRest восстанавливает разностную резервную копию, копируя все файлы из выбранной разностной резервной копии и соответствующие неизмененные файлы из предыдущей полной резервной копии. Преимущество дифференциальной резервной копии заключается в том, что для нее требуется меньше места на диске, чем для полной резервной копии, однако и дифференциальная резервная копия, и полная резервная копия должны быть действительными для восстановления дифференциальной резервной копии.

**Инкрементное резервное копирование (Incremental Backup)**: pgBackRest копирует только те файлы кластера базы данных, которые изменились с момента последнего резервного копирования (которое может быть другим добавочным резервным копированием, дифференциальным резервным копированием или полным резервным копированием). Поскольку добавочное резервное копирование включает только те файлы, которые были изменены с момента предыдущего резервного копирования, они, как правило, намного меньше, чем полные или дифференциальные резервные копии. Как и в случае с дифференциальным резервным копированием, инкрементное резервное копирование зависит от того, будут ли другие резервные копии действительными для восстановления инкрементной резервной копии. Поскольку добавочная резервная копия включает только те файлы, которые были созданы с момента последней резервной копии, все предыдущие добавочные резервные копии до предыдущей разностной, предыдущей разностной резервной копии и предыдущей полной резервной копии должны быть допустимы для выполнения восстановления добавочной резервной копии. Если дифференциальной резервной копии не существует, то все предыдущие инкрементные резервные копии возвращаются к предыдущей полной резервной копии, которая должна существовать, а сама полная резервная копия должна быть действительна для восстановления инкрементной резервной копии.

**Восстановление (Restore)** — это действие по копированию резервной копии в систему, где она будет запущена как работающий кластер базы данных. Для правильной работы восстановления требуются файлы резервных копий и один или несколько сегментов WAL.

**WAL** — это механизм, который PostgreSQL использует для обеспечения того, чтобы никакие зафиксированные изменения не были потеряны. Транзакции последовательно записываются в WAL, и транзакция считается зафиксированной, когда эти записи сбрасываются на диск. После этого фоновый процесс записывает изменения в основные файлы кластера базы данных (также известные как куча). В случае сбоя WAL воспроизводится повторно, чтобы сделать базу данных согласованной.
WAL концептуально бесконечен, но на практике он разбит на отдельные 16-мегабайтные файлы, называемые сегментами. Сегменты WAL следуют соглашению об именах 0000000100000A1E000000FE, где первые 8 шестнадцатеричных цифр представляют временную шкалу, а следующие 16 цифр — логический порядковый номер (LSN).

[Несколько полезных ссылок.](#полезные-ссылки)

### Конфигурация стенда

Горячий резерв (replica1) выполняет репликацию с использованием архива WAL и разрешает запросы только для чтения. В конфигурацию pgbackrest также добавлен сервер горячего резерва, это позволяет выполнять резервное копирование на сервер backup с распределением нагрузки.

Для выполнения резервного копирования требуются как первичная, так и резервная базы данных, хотя подавляющее большинство файлов будет скопировано из резервной базы данных, чтобы снизить нагрузку на первичную. Хосты базы данных можно настроить в любом порядке. pgBackRest автоматически определит, какой из них является основным, а какой — резервным.

pgBackRest создает резервную копию, идентичную резервной копии, созданной на основном сервере. Для этого он запускает/останавливает резервное копирование на первичном хосте master, копирует только те файлы, которые реплицированы с хоста replica1, а затем копирует несколько оставшихся файлов с первичного хоста master. Это означает, что журналы и статистика из первичной базы данных будут включены в резервную копию.

### Проверки

##### Проверка репликации

1. Выполним вход на master: `vagrant ssh master`
2. Повысим привелегии до root: `sudo -i`
3. Проверим работу репликации, создадим таблицу testdb: `sudo -u postgres psql -c "create database testdb;"`
4. Переходим на сервер replica1: `sudo -u postgres ssh replica1`
5. Получим список БД: `psql -c "\l"`
6. Должны увидеть созданную БД на master: `testdb`

```bash
test@test-virtual-machine:~/Otus/postgresql$ vagrant ssh master
Last login: Tue Feb  7 22:35:39 2023 from 10.0.2.2
[vagrant@master ~]$ sudo -i
[root@master ~]# sudo -u postgres psql -c "create database testdb;"
could not change directory to "/root": Отказано в доступе
CREATE DATABASE
[root@master ~]# sudo -u postgres psql -c "create database testdb;"
could not change directory to "/root": Отказано в доступе
CREATE DATABASE
[root@master ~]# sudo -u postgres ssh replica1
Warning: Permanently added the ED25519 host key for IP address '192.168.57.11' to the list of known hosts.
-bash-4.2$ psql -c "\l"
                                                  Список баз данных
    Имя    | Владелец | Кодировка | LC_COLLATE  |  LC_CTYPE   | локаль ICU | Провайдер локали |     Права доступа     
-----------+----------+-----------+-------------+-------------+------------+------------------+-----------------------
 postgres  | postgres | UTF8      | en_US.UTF-8 | en_US.UTF-8 |            | libc             | 
 template0 | postgres | UTF8      | en_US.UTF-8 | en_US.UTF-8 |            | libc             | =c/postgres          +
           |          |           |             |             |            |                  | postgres=CTc/postgres
 template1 | postgres | UTF8      | en_US.UTF-8 | en_US.UTF-8 |            | libc             | =c/postgres          +
           |          |           |             |             |            |                  | postgres=CTc/postgres
 testdb    | postgres | UTF8      | en_US.UTF-8 | en_US.UTF-8 |            | libc             | 
(4 строки)
```

##### Проверка резервного копирования

Если на предыдущей проверке окно небыло закрыто, то следующие 3 шага можно пропустить, и приступить к шагу 4.

1. Выполним вход на master: `vagrant ssh master`
2. Повысим привелегии до root: `sudo -i`
3. Выполним вход под пользователем postgres: `sudo su postgres`
4. Выполним проверку правильности настройки pgBackRest: `pgbackrest --stanza=demo --log-level-console=info check`
5. Проверим наличие резервных копий в репозитории: `pgbackrest info`

```bash
test@test-virtual-machine:~/Otus/postgresql$ vagrant ssh master
Last login: Tue Feb  7 22:46:25 2023 from 10.0.2.2
[vagrant@master ~]$ sudo -i
[root@master ~]# sudo su postgres
bash-4.2$ pgbackrest --stanza=demo --log-level-console=info check
2023-02-07 23:12:39.303 P00   INFO: check command begin 2.44: --exec-id=17507-ad326689 --log-level-console=info --log-level-file=detail --pg1-path=/var/lib/pgsql/15/data --repo1-host=backup --repo1-host-user=postgres --stanza=demo
2023-02-07 23:12:39.920 P00   INFO: check repo1 configuration (primary)
2023-02-07 23:12:40.731 P00   INFO: check repo1 archive for WAL (primary)
2023-02-07 23:12:42.904 P00   INFO: WAL segment 000000010000000000000009 successfully archived to '/var/lib/pgbackrest/archive/demo/15-1/0000000100000000/000000010000000000000009-2761940112f28adcec64eeccc63e842632221b2b.gz' on repo1
2023-02-07 23:12:43.025 P00   INFO: check command end: completed successfully (3724ms)
bash-4.2$ pgbackrest info
stanza: demo
    status: ok
    cipher: none

    db (current)
        wal archive min/max (15): 000000010000000000000001/000000010000000000000009

        full backup: 20230207-223236F
            timestamp start/stop: 2023-02-07 22:32:36 / 2023-02-07 22:32:46
            wal start/stop: 000000010000000000000005 / 000000010000000000000005
            database size: 22.3MB, database backup size: 22.3MB
            repo1: backup set size: 3.0MB, backup size: 3.0MB
```

---

###### Примечания

Из [официальной документации](https://postgrespro.ru/docs/postgresql/15/recovery-config?lang=ru):

В PostgreSQL 11 и более ранних версиях существовал файл конфигурации recovery.conf, предназначенный для управления репликами и резервными серверами. Начиная с PostgreSQL 12, этот файл не поддерживается. Информация о данном изменении представлена в [Замечаниях к выпуску PostgreSQL 12](https://postgrespro.ru/docs/postgresql/15/release-prior).

В PostgreSQL версии 12 и выше [восстановление архивов, потоковая репликация и PITR](https://postgrespro.ru/docs/postgresql/15/continuous-archiving) настраиваются с использованием [обычных параметров конфигурации сервера](https://postgrespro.ru/docs/postgresql/15/runtime-config-replication#RUNTIME-CONFIG-REPLICATION-STANDBY). Соответствующие параметры, как и любые другие, можно задать командой [ALTER SYSTEM](https://postgrespro.ru/docs/postgresql/15/sql-altersystem) или внести в файл postgresql.conf.

Сервер не запустится, если обнаружит файл recovery.conf.

Параметр trigger_file переименован в [promote_trigger_file](https://postgrespro.ru/docs/postgresql/15/runtime-config-replication#GUC-PROMOTE-TRIGGER-FILE).

Параметр standby_mode удалён. Его роль теперь выполняет файл standby.signal в каталоге данных. Подробности в [Работа резервного сервера](https://postgrespro.ru/docs/postgresql/15/warm-standby#STANDBY-SERVER-OPERATION).

###### Полезные ссылки

[PostgreSQL in production](https://github.com/vitabaks/postgresql_cluster)

[pgBackRest](https://pgbackrest.org/)

[Непрерывное архивирование и восстановление на момент времени (Point-in-Time Recovery, PITR)](https://postgrespro.ru/docs/postgresql/15/continuous-archiving)

[SQL Dump](https://www.postgresql.org/docs/current/static/backup-dump.html)

[File System Level Backup](https://www.postgresql.org/docs/current/static/backup-file.html)

[Reliability and the Write-Ahead Log](https://www.postgresql.org/docs/current/static/wal.html)
