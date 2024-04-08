# Задание: 
Базу развернуть на мастере и настроить так, чтобы реплицировались таблицы:

| bookmaker |
| competition |
| market |
| odds |
| outcome
Настроить GTID репликацию
x
варианты которые принимаются к сдаче
рабочий вагрантафайл
скрины или логи SHOW TABLES
конфиги*

пример в логе изменения строки и появления строки на реплике*

---
# Решение:

Запускаем стенд vagrant из следущего vagrantfile:

```
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :master => {
        :box_name => "centos/7",
        :ip_addr => '192.168.56.150'
  },
  :slave => {
        :box_name => "centos/7",
        :ip_addr => '192.168.56.151'
  }
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

      config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s

          #box.vm.network "forwarded_port", guest: 3260, host: 3260+offset

          box.vm.network "private_network", ip: boxconfig[:ip_addr]

          box.vm.provider :virtualbox do |vb|
            vb.customize ["modifyvm", :id, "--memory", "512"]
          end

          box.vm.provision :shell do |s|          
             s.inline = <<-SHELL
             mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh
             sed -i '65s/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
             systemctl restart sshd
           SHELL

          end

      end
  end
end

```

С помощью Ansible устанавливаем Percona-server, вот вывод:
```
[andrej@home-srv vagrant]$ ansible-playbook ../ansible/playbooks/mysql.yaml -i ../ansible/inventory/hosts 

PLAY [all] ***********************************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************************
ok: [slave]
ok: [master]

TASK [mysql : Add Percona repo from remote repo] *********************************************************************************************************************************************
changed: [slave]
changed: [master]

TASK [mysql : Add Percona-server] ************************************************************************************************************************************************************
changed: [slave]
changed: [master]

TASK [mysql : copy conf files] ***************************************************************************************************************************************************************
changed: [slave] => (item=01-base.cnf)
changed: [master] => (item=01-base.cnf)
changed: [slave] => (item=02-max-connections.cnf)
changed: [master] => (item=02-max-connections.cnf)
changed: [slave] => (item=03-performance.cnf)
changed: [master] => (item=03-performance.cnf)
changed: [master] => (item=04-slow-query.cnf)
changed: [slave] => (item=04-slow-query.cnf)
changed: [master] => (item=05-binlog.cnf)
changed: [slave] => (item=05-binlog.cnf)

TASK [mysql : Make sure group wheel is not in the sudoers configuration] *********************************************************************************************************************
ok: [master]
ok: [slave]

TASK [mysql : change my.conf для смены пароля] ***********************************************************************************************************************************************
changed: [master]
changed: [slave]

TASK [mysql : start mysql] *******************************************************************************************************************************************************************
changed: [master]
changed: [slave]

PLAY RECAP ***********************************************************************************************************************************************************************************
master                     : ok=7    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
slave                      : ok=7    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Меняем пароли, @@server_id на slave и убеждаемся, что @@server_id разные на master и slave:

```

mysql> SHOW VARIABLES LIKE 'gtid_mode';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | ON    |
+---------------+-------+
1 row in set (0.00 sec)

mysql> SET GLOBAL server_id=2;
Query OK, 0 rows affected (0.00 sec)

```

```

mysql> SELECT @@server_id;
+-------------+
| @@server_id |
+-------------+
|           2 |
+-------------+
1 row in set (0.00 sec)

```

На `master` создаём базу `bet`, загружаем в неё дамп и смотрим результат:

```
mysql> USE bet;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> SHOW TABLES;
+------------------+
| Tables_in_bet    |
+------------------+
| bookmaker        |
| competition      |
| events_on_demand |
| market           |
| odds             |
| outcome          |
| v_same_event     |
+------------------+
7 rows in set (0.00 sec)
```

Создаём пользователя для репликации, даём ему права и дампим базу:
```
mysql> CREATE USER 'repl'@'%' IDENTIFIED BY '!OtusLinux2024';
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT user,host FROM mysql.user where user='repl';
+------+------+
| user | host |
+------+------+
| repl | %    |
+------+------+
1 row in set (0.00 sec)

mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY '!OtusLinux2024';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> \q
Bye
[root@master ~]# mysqldump --all-databases --triggers --routines --master-data --ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > master.sql
Enter password: 
Warning: A partial dump from a server that has GTIDs will by default include the GTIDs of all transactions, even those that changed suppressed parts of the database. If you don't want to restore GTIDs, pass --set-gtid-purged=OFF. To make a complete dump, pass --all-databases --triggers --routines --events. 
[root@master ~]# ls
anaconda-ks.cfg  master.sql  original-ks.cfg
```

На `Slave `После загрузки базы из дампа смотрим результат:
```
mysql> USE bet;
Database changed
mysql> SHOW TABLES;
+---------------+
| Tables_in_bet |
+---------------+
| bookmaker     |
| competition   |
| market        |
| odds          |
| outcome       |
+---------------+
5 rows in set (0.00 sec)

mysql> 
```

Включаем `Slave`:

```
mysql> CHANGE MASTER TO MASTER_HOST = "192.168.56.150", MASTER_PORT = 3306, MASTER_USER = "repl", MASTER_PASSWORD = "!OtusLinux2024", MASTER_AUTO_POSITION = 1;
Query OK, 0 rows affected, 2 warnings (0.01 sec)

mysql> START SLAVE;
Query OK, 0 rows affected (0.01 sec)

mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.56.150
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 120769
               Relay_Log_File: slave-relay-bin.000003
                Relay_Log_Pos: 454
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: bet.events_on_demand,bet.v_same_event
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 120769
              Relay_Log_Space: 2616
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: f70e9e0d-f5d2-11ee-8ebe-5254004d77d3
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: f70e9e0d-f5d2-11ee-8ebe-5254004d77d3:39-44
            Executed_Gtid_Set: f70e9e0d-f5d2-11ee-8ebe-5254004d77d3:1-44,
f7115ecb-f5d2-11ee-9003-5254004d77d3:1
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)
```

На `Master` меняем `bet`:
```
mysql> USE bet;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> INSERT INTO bookmaker (id,bookmaker_name) VALUES(1,'1xbet');
Query OK, 1 row affected (0.00 sec)

mysql> SELECT * FROM bookmaker;
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |
|  2 | 2xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
6 rows in set (0.00 sec)
```

И проверяем на `Slave`:

```
mysql> SELECT * FROM bookmaker;
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |
|  2 | 2xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
6 rows in set (0.00 sec)

```

