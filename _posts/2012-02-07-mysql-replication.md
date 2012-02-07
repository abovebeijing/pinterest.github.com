---
layout: post
title: "MySQL Replication Tutorial"
categories: 
  - python
  - monitoring
date: Feb 06, 2012
time: "10:45"
date_time: "6 Feb, 2012 - <a href='http://pinterest.com/martaaay/'>Marty Weiner</a>"
---

I thought I'd write a quick tutorial on how to get MySQL replication started.  We use Percona MySQL for a variety of
reasons, one being that it comes with InnoBackup, a tool for taking a backup of your MySQL instance without locking up 
database.  Even cooler, recovery from an InnoBackup generated backup is instant.  

Here's how it works:

### Replication With Percona InnoBackup

Note: We're using Percona's MySQL 5.5.15-55 with XtraBackup 1.6.3 an Amazon EC2 m2.xlarge.  All of our tables are InnoDB.

On the master DB, run InnoBackup:

    master$ innobackupex --user=root --password="<PASSWORD>" \
         --defaults-file /etc/mysql/my.cnf /path/to/backup/

This will take a snapshot of the current dataset and can take an hour or two for datasets around 200GB to 300GB.  It does eat up
a fair amount of I/O bandwidth, so make sure your master database can handle the extra load.

During the backup, concurrent writes are written to a file called ibbackup_logfile.  These write logs need to be applied against 
the backup files once the snapshot is complete.  Run:

    master$ innobackupex --user=root --password="<PASSWORD>" \
         --defaults-file /etc/mysql/my.cnf \
         --apply-log /path/to/backup/2011-10-06_00-35-42/

Replace the date at the end with the one innobackup just created in your backup dir.  Once finished you'll
have a clean replica of your /var/lib/mysql directory in the backup dir.  Copy these files straight
into the /var/lib/mysql dir on your slave machine.

Give REPLICATION permission for the slave: 

    master$ mysql -u root -p"<PASSWORD>"
    > GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* \
         TO 'replicant'@'<SLAVEIP>' identified by '<REPLICATIONSECRET>';

Where SLAVEIP is the ip address of your slave machine and REPLICATIONSECRET is a password the slave will use to authenticate itself to the master.  Just make
something up.

Switch over to the slave DB.  At this point, we have a snapshot of the master's database.  Replication works by using
this initial dataset and asking the master for each subsequent write after that point, so we'll need to tell the slave
what log position the master was in at the moment the backup began:

    slave$ cat /var/lib/mysql/xtrabackup_binlog_info
    mysql-bin.000313	36518684
    
This says that the master was currently writing to log file 'mysql-bin.000313' at position 36518684 when the backup was initiated.

Start MySQL on the slave if it isn't already running, and enter the CLI with full permissions.

    slave$ mysql -u root -p"<PASSWORD>"
    > STOP SLAVE;
    > RESET SLAVE;
    > CHANGE MASTER TO MASTER_HOST='<MASTERIP>', MASTER_PORT=3306, \
         MASTER_USER='replicant', MASTER_PASSWORD='<REPLICATIONSECRET>', \
         MASTER_LOG_FILE='mysql-bin.000313', MASTER_LOG_POS=36518684;
    > START SLAVE;
    > SHOW SLAVE STATUS\G

Of course, replace the log file and log position with what you just pulled out of xtrabackup_binlog_info.  If all goes well, 
*SHOW SLAVE STATUS* should show *Slave_IO_Running* and *Slave_SQL_Running* as Yes.  Seconds_Behind_Master will show how behind the slave
is.  It's not uncommon for bigger datasets to take 8 to 24 hours (or more!) to get transferred over.  If there are not too many writes
coming in (and no reads on the slave), the slave should catch up within a few hours.

If Slave_IO_Running or Slave_SQL_Running is reporting 'No', SHOW SLAVE STATUS will list the reason.  A common mistake is incorrect
firewall settings (try **telnet &lt;MASTERIP> 3306** from the slave and see if you can connect).

Note: You should treat this slave as a readonly slave.  If you insert into it and the insert races with an incoming replicated insert, then
you may create a primary key collision which halts replication.

