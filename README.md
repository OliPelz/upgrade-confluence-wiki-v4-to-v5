# Upgrade Confluence Wiki to a new server (4.xx to 5.10.0) and migrating from mysql to postgres in the same process.
=========================================================
Here are my personal very detailed steps and the full complete process, please remember - mine migration process will be most likely different than yours, this is especially true for any mysql import errors while upgrading...
The exact version upgrade is from 4.0.3 to 5.10.0

One big step of my migration process is to convert the MySQL database to a Postgres system because this is just the better DBMS in my opinion (and also recommended by Atlassian)! Therefore we need to export the Confluence DB and data to XML using the confluence admin web application functionality (we cannot just dump the mysql database and import it to postgres).

Since our database is very huge and our production server does not have enough free RAM for the export process to run, we need to use a migration server (my own local workstation) first to export our database to xml using the confluence admin frontend (the migration server is also useful to completely test upgrading your Confluence wiki before you go to real production mode). Therefore I have to migrate the whole existing confluence installation to my system first. In our case the confluence upgrade takes about ~8GB Java Heap space because of the huge DB.

So the plan of action in my special case is:

* migrate the existing confluence server onto a migration server - which is my own workstation with big RAM (so I can export the database to xml)
* Backup the status quo: backup the existing database and data and export it also to XML  using the confluence frontend (all this before fixing database problems).
* fix all the database problems (otherwise upgrading the database will crash due to mysql schema problems)
* initate the upgrade process from v4 to v5 on the migration server so we can use the data in a new 5.x environment later.
* export again to xml, this will be used to import later into a different DB backend -> postgres
* set up a new production server with postgres backend
* import the xml from the migration server
* DONE!

## Prequesites  
* login credentials at hand for your atlassian account (https://my.atlassian.com) where you find your current v5 license key and also download the latest v5.10.x Linux Installer .run package.
* For simplicity folder name structure of the confluence installation and operating system on all three servers (```old_production_server```, ```migration_server``` and ```new_production_server```=(target) server) will be the same. In our example we will use CentOS 7 for all three server instances and  ```/opt/confluence``` for the confluence installation. If you need to adapt to a different folder on the production server start reading this document here: ```https://confluence.atlassian.com/doc/confluence-home-and-other-important-directories-590259707.html```

* Mysql server and Oracle Java 6 (Confluence 4.x needs Java 6! and does not work with latest 7 or 8) as well as JDK 8 installed on the migration server
* Postgres and Oracle Java 8 on the new-production (target) server for confluence 5.10

Are you ready? Fasten your seatbelt:

## Migrate confluence to migration server
In the code examples we will use the hostnames ```old_production_server```, ```migration_server```, ```new_production_server``` for labeling the right server.

on the migration server use the following commands to install MariaDB (which is a mysql clone):
```bash
migration_server$ yum install mariadb mariadb-server -y; systemctl enable mariadb;
``` 
Next we need to tune the MariaDB mysql server a bit in order it will be able to import and export the confluence database and web frontend.
Put the following lines somewhere in your ```/etc/my.cnf``` MariaDB config file within the ```[mysqld]``` section:
```bash 
wait_timeout = 10000
innodb_lock_wait_timeout = 10000
max_allowed_packet = 256M
transaction-isolation=READ-COMMITTED
```
Now start the server and init some security measuements:
```bash
migration_server$ systemctl start mariadb; mysql_secure_installation
```

On the migration server we need Java 6 for starting the original Confluence wiki! So we need to install it first (we will later switch to Java 8 on the migration server after upgrading confluence).
Please note that Java 6 is considered to be a security threat! Do not install it on a productive system.

Log in to the migration server and type following to install Java 6:
```bash
migration_server$ mkdir ~/Downloads; cd $_
migration_server$ curl -L -o jdk-6u45-linux-x64.bin -H "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/6u45-b06/jdk-6u45-linux-x64.bin
migration_server$ chmod +x ~/Downloads/jdk-6u45-linux-x64.bin
migration_server$ cd /opt
migration_server$ ~/Downloads/jdk-6u45-linux-x64.bin
```

Next configure the migration server to use Java 6:

```bash
migration_server$ alternatives --install /usr/bin/java java /opt/jdk1.6.0_45/bin/java 2
migration_server$ alternatives --set java /opt/jdk1.6.0_45/bin/java
migration_server$ alternatives --install /usr/bin/jar jar /opt/jdk1.6.0_45/bin/jar 2
migration_server$ alternatives --install /usr/bin/javac javac /opt/jdk1.6.0_45/bin/javac 2
migration_server$ alternatives --set jar /opt/jdk1.6.0_45/bin/jar
migration_server$ alternatives --set javac /opt/jdk1.6.0_45/bin/javac
```

setup environment for Java 6
```
migration_server$ echo "
export JAVA_HOME=/opt/jdk1.6.0_45
export JRE_HOME=/opt/jdk1.6.0_45/jre
export PATH=$PATH:/opt/jdk1.6.0_45/bin:/software/jdk1.6.0_45/jre/bin
"> /etc/profile.d/java.sh
migration_server$ source /etc/profile.d/java.sh
```

test (should now show Oracle java 6):
```
java -version
```

Warning: after import has been completed switch back to your normal java version 7 or 8 (using alternatives command).


(credits to http://tecadmin.net/steps-to-install-java-on-centos-5-6-or-rhel-5-6/)


#Migrating confluence installation to the migration server
First thing is to download the existing confluence installation from old production server to the migration server, in our example everything has been deployed to ```/opt/confluence``` (installation is in ```/opt/confluence/confluence```, data dir is in ```/opt/confluence/data```) on the current old production server (as said before we will keep the same directories on all other servers as well).

Login on the old production server which in our example has the hostname ```old_production_server``` and the migration server ```migration_server```. We will make a tar ball of the original wiki's state so we have a backup at hand when something goes wrong later!:
```bash
old_production_server$ CONFL_HOME=/opt/confluence
old_production_server$ CONFL_HOME_PROD=$CONFL_HOME/data
old_production_server$ now=`date +"%m_%d_%Y"`
old_production_server$ mkdir ~/confluence-backup-$now; cd $_
old_production_server$ tar zcvf ~/confluence-backup-$now/confluence.tar.gz $CONFL_HOME &> ~/confluence-backup-$now/confluence-tar.log
```
afterwards check ```~/confluence-backup-$now/confluence-tar.log``` for any errors.

Dump the confluence database (in our example the database is running on the old production server and is called ```confluencedb```) and restore it on the migration server 
To find out the correct database name on the production server, use the following command (change the root confluence folder accordingly if different on your machine):
```bash
old_production_server$ grep "hibernate.connection.url" $CONFL_HOME_PROD/confluence.cfg.xml
```

```bash  
old_production_server$ cd ~/confluence-backup-$now; mysqldump -u root -p<Enter SQL database Password here> \
confluencedb > ./confluencedb.sql;gzip ./confluencedb.sql
```

Now after both backups have been completed on the ```old_production_server``` login to your migration server (please note we change servers now!) and transfer our compressed tarballs and extract (this will extract to ```/opt```), extracting should be done by ```root``` user:
```bash
migration_server$ now=`date +"%m_%d_%Y"`
migration_server$ mkdir ~/confluence-backup-$now; cd $_
migration_server$ rsync -rav johndoe@old_production_server:~/confluence-backup-$now ~ --progress
migration_server$ cd /; tar xvf ~/confluence-backup-$now/confluence.tar.gz
```

For all our examples we will first set the following two env variables (on the ```migration_server``` and later on the ```new_production_server```):

```bash
migration_server$ export CONFL_HOME=/opt/confluence/data
migration_server$ export CONFL_WEB=/opt/confluence/confluence
```

Now install the database first (in our case the confluence database is called ```confluencedb```:
```bash
migration_server$ cd ~/confluence-backup-$now; gunzip confluencedb.sql.gz; 
migration_server$ echo "DROP DATABASE IF EXISTS confluencedb;CREATE DATABASE confluencedb CHARACTER SET utf8 COLLATE utf8_bin;" | mysql -u root -p
migration_server$ mysql -u root -p confluencedb < ~/confluence-backup-$now/confluencedb.sql
```
Lets create a database user for confluence, which will be exact the same as on the old-production server
read out the original database user to connect:
 
```bash
migration_server$ grep "hibernate.connection.username\|hibernate.connection.password" $CONFL_HOME/confluence.cfg.xml
```

Take this username and password from the output above and create the following database user:
```bash
migration_server$ echo "GRANT ALL ON confluencedb.* TO '<confluence db user name>'@'localhost' identified by '<confluence db user password>';" | mysql -u root -p
```
Now move on fixing some database problems. Before we fix these database problems make sure you keep your backups we made for copying the confluence wiki at a safe place first! Put it on your backup file server etc...

Here we need to fix some database problems before export / upgrade is possible, otherwise the exporter/upgrader will crash later. Please fix this on your migration system only which is a copy of the original database from the ```old_production_server```! Please note that most of the error fixes shown here are individual to my confluence database installation. You need to put some energy into resolving your own problems and research quite a bit on the Internet if errors occur.

Login to your mysql shell and first check and convert the confluence database to the right collation (utf-8 is mandantory for Confluence 5.x):
```bash
migration_server$ mysql -u root -p confluencedb
```
check collation to be utf-8 :
```mysql
MariaDB [confluencedb]> SELECT DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME FROM information_schema.SCHEMATA WHERE schema_name = 'confluencedb'
AND
(
    DEFAULT_CHARACTER_SET_NAME != 'utf8'
    OR
    DEFAULT_COLLATION_NAME != 'utf8_bin'
);
```
If the output of the last command is non empty! (```Empty set (0.00 sec)```) change it using the following command,(otherwise skip this step):
```bash
ALTER DATABASE confluencedb CHARACTER SET utf8 COLLATE utf8_bin
```

Now log out the mysql shell and run the following query in bash (insert your mysql password when prompted):

```bash
migration_server$ echo "
SELECT T.TABLE_NAME, C.CHARACTER_SET_NAME, C.COLLATION_NAME
FROM information_schema.TABLES AS T, information_schema.COLLATION_CHARACTER_SET_APPLICABILITY AS C
WHERE C.collation_name = T.table_collation
AND T.table_schema = 'confluencedb'
AND
(
    C.CHARACTER_SET_NAME != 'utf8'
    OR
    C.COLLATION_NAME != 'utf8_bin'
);
SELECT TABLE_NAME, COLUMN_NAME, CHARACTER_SET_NAME, COLLATION_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'confluencedb'
AND
(
    CHARACTER_SET_NAME != 'utf8'
    OR
    COLLATION_NAME != 'utf8_bin'
);
" | mysql -u root -p 

```
if the run script has any output lines AT ALL other than ONLY ```Empty set``` or NO output at all, run the following sql query to convert anything to UTF8 collation (you have to input your mysql password twice):

```bash
migration_server$ echo "
SELECT CONCAT('ALTER TABLE ',  table_name, ' CHARACTER SET utf8 COLLATE utf8_bin;') FROM information_schema.TABLES AS T, information_schema.\`COLLATION_CHARACTER_SET_APPLICABILITY\` AS C
WHERE C.collation_name = T.table_collation
AND T.table_schema = 'confluencedb'
AND
(
    C.CHARACTER_SET_NAME != 'utf8'
    OR
    C.COLLATION_NAME != 'utf8_bin'
);
SELECT CONCAT('ALTER TABLE \`', table_name, '\` MODIFY \`', column_name, '\` ', DATA_TYPE, '(', CHARACTER_MAXIMUM_LENGTH, ') CHARACTER SET UTF8 COLLATE utf8_bin', (CASE WHEN IS_NULLABLE = 'NO' THEN ' NOT NULL' ELSE '' END), ';')
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'confluencedb'
AND DATA_TYPE = 'varchar'
AND
(
    CHARACTER_SET_NAME != 'utf8'
    OR
    COLLATION_NAME != 'utf8_bin'
);
SELECT CONCAT('ALTER TABLE \`', table_name, '\` MODIFY \`', column_name, '\` ', DATA_TYPE, ' CHARACTER SET UTF8 COLLATE utf8_bin', (CASE WHEN IS_NULLABLE = 'NO' THEN ' NOT NULL' ELSE '' END), ';')
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'confluencedb'
AND DATA_TYPE != 'varchar'
AND
(
    CHARACTER_SET_NAME != 'utf8'
    OR
    COLLATION_NAME != 'utf8_bin'
);
" | mysql -u root -p | grep -v CONCAT >   /tmp/alter-script-collation.sql; mysql -u root -p confluencedb< /tmp/alter-script-collation.sql
```

Now run the first database query from above again to check that everything now is UTF-8, this should return a complete empty output if successful!:
```bash
migration_server$ echo "
SELECT T.TABLE_NAME, C.CHARACTER_SET_NAME, C.COLLATION_NAME
FROM information_schema.TABLES AS T, information_schema.COLLATION_CHARACTER_SET_APPLICABILITY AS C
WHERE C.collation_name = T.table_collation
AND T.table_schema = 'confluencedb'
AND
(
    C.CHARACTER_SET_NAME != 'utf8'
    OR
    C.COLLATION_NAME != 'utf8_bin'
);
SELECT TABLE_NAME, COLUMN_NAME, CHARACTER_SET_NAME, COLLATION_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'confluencedb'
AND
(
    CHARACTER_SET_NAME != 'utf8'
    OR
    COLLATION_NAME != 'utf8_bin'
);
" | mysql -u root -p 

```
Before we can start the web interface give the Confluence webapp more heap so we can export our v4 for backup purpose as XML before we upgrade finally.
increase ```-Xmx1024m``` to ```-Xmx8000m```
```bash
migration_server$ vi $CONFL_WEB/bin/setenv.sh
```
Next if you need change the listening port from 8280 to 8080 etc., disable ssl on local machine, change other settings etc in (otherwise skip):
```bash
migration_server$ vi $CONFL_WEB/conf/server.xml
```
e.g. the main http confluence listening port will be defined here with:
```bash
<Connector className="org.apache.coyote.tomcat4.CoyoteConnector" port="8280" ...
```
Next for accessing confluence in your network you need to open the firewall port 8280 etc. if you are running a local firewall etc., for me on CentOS 7 it was (this is optionally if you want to access your Confluence migration installation from another machine in the same network):
```bash
migration_server$ firewall-cmd --permanent --add-port=8280/tcp
migration_server$ firewall-cmd --reload
```
Before we proceed with starting our confluence server for the first time, we should first need to enable correct logging output for confluence optimized for exporting the data to XML (which we will do in the next step). This is very important so the XML export performance is best, because log level has direct output of Confluence overall performance and if you have a very verbose but unnecessay log output for all confluence messages it can take far too long. When I tried to export to XML with full log output everything it took me +8hours while with the log level seen below it takes me about 10 minutes. Talk about a difference! 
So what we need is a logging setup which hides all unneeded log outputs but displays us detailed export error log messages.
To show important export confluence messages we need to adjust the log level in ```log4j.properties```. First make a backup of the original log4j file so we can later easily revert back.
```bash
migration_server$ cp $CONFL_WEB/confluence/WEB-INF/classes/log4j.properties $CONFL_WEB/confluence/WEB-INF/classes/log4j.properties.BAK
``` 
Now replace the existing ```log4j.properties``` file with the following content (most important part are the ```log4j.logger.com.atlassian.confluence.importexport``` settings)

```bash
echo "
log4j.rootLogger=ERROR, confluencelog, errorlog
log4j.appender.confluencelog.layout=com.atlassian.confluence.util.PatternLayoutWithContext
log4j.appender.confluencelog.layout.ConversionPattern=%d %p [%t] [%c{4}] %M %m%n
log4j.appender.errorlog=com.atlassian.core.logging.ThreadLocalErrorLogAppender
log4j.appender.errorlog.Threshold=ERROR
log4j.appender.confluencelog=com.atlassian.confluence.logging.ConfluenceHomeLogAppender
log4j.appender.confluencelog.Threshold=ERROR
log4j.appender.confluencelog.MaxFileSize=99999999KB
log4j.appender.confluencelog.MaxBackupIndex=5
log4j.logger.net.sf.hibernate.type.CustomType=ERROR
log4j.logger.net.sf.hibernate.type=ERROR
log4j.logger.net.sf.hibernate.cache.ReadWriteCache=ERROR
log4j.logger.net.sf.hibernate.cache.EhCacheProvider=ERROR
log4j.logger.org.springframework.jmx.support.JmxUtils=ERROR
log4j.logger.com.atlassian.confluence.importexport.impl.XMLDatabinder=DEBUG, confluencelog
log4j.additivity.com.atlassian.confluence.importexport.impl.XMLDatabinder=false
" > $CONFL_WEB/confluence/WEB-INF/classes/log4j.properties

```

Now start the server in the FOREGROUND TO SEE WHATS GOING ON (there are likely to be some errors)
```bash
migration_server$ sh $CONFL_WEB/bin/catalina.sh run
```
Open another window and tail the log file using:
```bash
migration_server$ export CONFL_HOME=/opt/confluence/data
migration_server$ tail -f $CONFL_HOME/logs/atlassian-confluence.log
```
wait some time (several minutes) and watch log files and processes (e.g. using top), than start the app in the web browser by browsing to (in my case the document root of confluence has been set to ```/wiki``` which can be different for you, you can adjust it in ```$CONFL_WEB/conf/server.xml``` in the element ```<Context path=/wiki ....>```): ```http://migration_server:8280/wiki```, a login window should appear within 1-5 minutes. You can also try to wait until the message appears before browsing to the page. Please note: the context path has to be changed on several different places, please read: https://confluence.atlassian.com/confkb/how-to-change-the-confluence-context-path-707985922.html
```bash
xxxx INFO [main] [com.atlassian.confluence.lifecycle] init Confluence is ready to serve
```
or
```bash
INFO: Starting service Tomcat-Standalone
```

Next as we already have the mysql dump of the confluence web site we also want a backup of confluence in its original state as XML before we initate the upgrading process (this step is optional but also nice if you ever want to go back to your 4.x version)

## Backup original production confluence wiki as XML
When browsing to your migration server wiki startpage you should be presented with a login screen, login in.
* Go to the admin page at ```http://migration_server:8280/wiki/admin```
* Here navigate to: Administration->Backup & Restores
* Tick  "Archive to backups folder" so that both options:  "Archive to backups folder" and "Back up attachments" are enabled.
* Click on "Back Up" button and wait.... (follow progress and error messages in your server's log file and  ```tail -f $CONFL_HOME/logs/atlassian-confluence.log```)
* If there are any errors during backup you will see them in your two terminal windows. Most likely they are database problems or memory related. If there are any errors and you are stuck, try to make an Internet search for the exact error or Exception message and start by reading offical atlassian or confluence website / message board entries. 
* Depending on the size of your confluence wiki installation, the backup process can take about 10-30 minutes. If it is done, you will be presented by a success message with the exact location of your confluence XML backup file, for me it was:

```bash
The backup was successfully created at /data/opt/confluence/data/temp/xmlexport-20160321-112943-1.zip. This file will be deleted in 24 hours.
```
Oftentimes the XML backup can not be generated because of SQL errors. If this happens to you XML generation will abort and the exact SQL error/problem reason will be printed into our terminal log files. In [Appendix a](#appendix-a-fixing-xml-backup-issues-with-the-database-for-v403). you will find all the issues I had with my 4.x to 5.9.x XML backup procedure.

You should now backup this XML zipfile which is your full confluence backup in a safe place e.g. on a backup drive 

Now after we made a backup of the original 4.x confluence state as XML backup, lets now upgrade the migration server before we can create the XML backup once again (most likely the database needs to be modified for this to work), which we will then use on our new production server.
Please stop the server first (ctrl+c in the window where it is running

# Upgrading the ```migration_server``` to 5.x.

For converting/migrating the database structure and data to the new 5.x scheme, we need to run the confluence upgrader and web frontend which needs Java 8 on your ```migration_server```. So first thing we need to do is install Java 8 and enable it on your migration server (switch from Java6 to Java8):

```
cd ~/Downloads
wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u91-b14/jdk-8u91-linux-x64.tar.gz"
cd /opt
tar xzvf ~/Downloads/jdk-8u91-linux-x64.tar.gz
```

now switch from Java6 to Java8 on ```migration_server```:
```bash
alternatives --install /usr/bin/java java /opt/jdk1.8.0_91/bin/java 2
alternatives --set java /opt/jdk1.8.0_91/bin/java
alternatives --install /usr/bin/jar jar /opt/jdk1.8.0_91/bin/jar 2
alternatives --install /usr/bin/javac javac /opt/jdk1.8.0_91/bin/javac 2
alternatives --set jar /opt/jdk1.8.0_91/bin/jar
alternatives --set javac /opt/jdk1.8.0_91/bin/javac 
mv /etc/profile.d/java.sh /etc/profile.d/java6.sh.OLD
echo "
export JAVA_HOME=/opt/jdk1.8.0_91
export JRE_HOME=/opt/jdk1.8.0_91/jre
export PATH=$PATH:/opt/jdk1.8.0_91/bin:/software/jdk1.8.0_91/jre/bin
"> /etc/profile.d/java.sh
source /etc/profile.d/java.sh
```
validate we are now using java 8
```
java -version
```


Now download confluence 5.10 EXECUTABLE from your 'my atlassian' web page (login to your account using https://my.atlassian.com) and transfer it to the server. In our example we will use: 
```atlassian-confluence-5.10.0-x64.bin``` (Download Confluence 5.10.0 - Linux Installer (64 bit)  

## Create a quick backup of your database/confluence app
Just quick before you start the actual upgrading process you MUST make a quick backup of the current myqsl database (you dont want to start all over - you already most likely modified it when you created the XML backup and/or converted to UTF8), datadir and confluence home dir BECAUSE its likely you run into some upgrading errors.
EVERYTIME YOU RUN INTO SUCH A PROBLEM YOU NEED TO 
* ROLLBACK TO STATUS QUO (as confluence modifies the database and app while upgrading and if it gets interrupted by an upgrade error the database and installation dir will be garbage)
* fix the error (do some Internet research, see later) - apply all the other fixes from former rounds if you need to iterate several times over several issues 
* try upgrading again. 
So do a quick backup the following way NOW! NEVER SKIP THIS SECTION
```
migration_server$ TMP_CNF_BCK_DIR=~/confluence-tmp-backup-for-upgrade-`date +"%m_%d_%Y"`
migration_server$ mkdir -p $TMP_CNF_BCK_DIR
migration_server$ mysqldump -u root -p --default-character-set=utf8 confluencedb > $TMP_CNF_BCK_DIR/database-dump.sql
```
Also make a full quick backup of the complete confluence home directory so we can roll back everything in case of any error
```
migration_server$ TMP_CNF_HOME_DIR=$TMP_CNF_BCK_DIR/confluence-home-backup
migration_server$ TMP_CNF_WEB_DIR=$TMP_CNF_BCK_DIR/confluence-web-backup_
migration_server$ rsync -rav $CONFL_HOME/ $TMP_CNF_HOME_DIR --exclude=data/temp --exclude=data/index --exclude=data/backups
migration_server$ rsync -rav $CONFL_WEB/  $TMP_CNF_WEB_DIR
```
## Start the v4 to v5 upgrading process

Now run the 4.x to 5.x upgrade (start with admin rights), to do so start the confluence server manually and keep log files open in another window (to easily spot upgrading errors):
```
migration_server$ chmod +x  ~/Downloads/atlassian-confluence-5.10.0-x64.bin
migration_server$  ~/Downloads/atlassian-confluence-5.10.0-x64.bin
```

Navigate through the text installer and choose option 
* "Upgrade an existing Confluence installation".
*  As a "Existing installation directory:" use your CONFL_HOME dir which is for me: /opt/confluence/confluence
*  Say NO to backing up home dir as we did this in a previous step and you dont want to double the work
*  Press Enter some times to browse through all the changes
*  Start the actual upgrade process by typing in Enter or "u"
*  after upgrade has been done and finished by the installer, Confluence has been automatically restarted in the background but as user root, which will produce errors.

First we want to kill the Confluence java process, which has been started in the background from the Confluence upgrade installer as user root. To do so ```grep``` for confluence processes and kill the right one:
```
ps aux | grep -i confluence
kill -9 <PID of correct confluence process>
```

## Adjust confluence after upgrading via the installer

Next since we are using mysql for the upgrade process and confluence 5.x is not shipped with the mysql driver (https://confluence.atlassian.com/doc/database-jdbc-drivers-171742.html) we need to install it manually:
```
# Download the Mysql Connector/j for your operating system using your browser and: http://dev.mysql.com/downloads/connector/j/
migration_server$ cd /tmp; wget http://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.38.tar.gz
migration_server$ tar xvf mysql-connector-java-5.1.38.tar.gz
migration_server$ cp ./mysql-connector-java-5.1.38/mysql-connector-java-5.1.38-bin.jar $CONFL_WEB/confluence/WEB-INF/lib
```
Also the Installer modified our heap memory settings during upgrade, set it again to 8G by increasing the value ```-Xmx1024``` to ```-Xmx8000```
```
migration_server$ vi $CONFL_WEB/bin/setenv.sh
```
When upgrading to v5.10.x I found out that some error messages can be avoided by deleting old indexed files.
```
rm -rf $CONFL_HOME/index/* $CONFL_HOME/journal/*
```
Also the log4j settings have been resetted to verbose output but we will keep it because it is useful for tracking down upgrade errors.
## Try to start confluence v5 after upgrading for the first time

Now logout root and start confluence with a normal system user so:
```
migration_server$ sh $CONFL_WEB/bin/catalina.sh run
```
Next open a new windows to show the output of the atlassian log file so we can trace down any upgrade errors which will occur
```bash
migration_server$ tail -f $CONFL_HOME/logs/atlassian-confluence.log 	     
```
Important: never interrupt upgrading. If you shutdown the server while upgrading you need to restart all over from restoring the upgrade

* Wait some 5 minutes inspecting both log output windows. It is likely that there will some plugin exceptions because of incompatible plugins after upgrade. Ignore them at the moment. After some time
the following message will appear:
```
Initialized Starting Confluence 5.10.0
```

* Now after this message popped up with your two log file output windows open start your browser and browse to ```http://migration_server:8280```  (port 8280 is the standard port for confluence after installation/upgrade and no web context dir such as /wiki is set up after upgrading). Now upgrade will continue, watch the log output and spinning wheel of the browser page (this will take another ~5-10 minutes).

* After some time you get an error message that your license has expired, put in the new license key in the form, click "Save" and restart the server (ctrl+c the catalina.sh process in the first window, then start it again, you can leave the live view on ```atlassian-confluence.log``` in the second window open).
Now the upgrade process will continue, again you have to wait some minutes and watch the output. Don't start the web application yet, wait for the success message, for me some "Upgrade failed messages appeared" so I stopped the server and resolved these issues than started all over by restoring the backup, fixing the issues and starting upgrade process again, see my [Appendix B.]

 At the end if everything has been successful, again the success message will be appear:

```
Initialized Starting Confluence 5.10.0

```

* After all those upgrade database errors have been resolved, you now can open the browser using ```http://migration_server:8280```  and wait again some time (for me it was 20 minutes - this is because of plugin upgrade processes in the background e.g. waitForBooleanValue Still waiting to find if plugin dependent upgrades are complete e.g. ```waitForBooleanValue Waiting to find if plugin dependent upgrades are complete. Maximum wait time will be 1800 seconds``` etc.). Just give it its time

If there are any upgrade errors you have to research some solutions from doing google search and afterwards start all over:
* you have to restore your backuped confluence database
* restore the original confluence dir
* fix the database errors with the information you got from researching, for a list of all the errors I need to fix see Appendix B.
* start the upgrade process again

To restore original settings do
```
echo "drop database confluencedb;create database confluencedb DEFAULT CHARACTER SET utf8  DEFAULT COLLATE utf8_bin;" | mysql -u root -p
mysql -u root -p confluencedb < $TMP_CNF_BCK_DIR/database-dump.sql
rm -rf $CONFL_HOME/*
rm -rf $CONFL_WEB/* 
migration_server$ rsync -rav $TMP_CNF_HOME_DIR/ $CONFL_HOME/
migration_server$ rsync -rav $TMP_CNF_WEB_DIR/ $CONFL_WEB/

```
Now fix your problems on the database and afterwards restart the upgrade process - continue at Start the v4 to v5 upgrading process afterwards. Add all your fixes to a list, because each time you restored the original database you need to apply ALL your fixes. (For me in my special case in summary here are all fixes I need to do in order to pass the upgrading process -  Appendix B.), do not apply these fixes on your installation, YOU have been warned.

You have to fix all the things until your confluence server starts with its typical login screen and the log file message
```
Starting Confluence 5.10.0
```

## XML backup from v.5.10
Now you can create the XML backup from the upgraded v5. Before you make the XML backup make sure you have 8G heap in the ```$CONFL_WEB/bin/setenv.sh``` file and also reduced logging output.
The standard logging log4j settings after upgrading to v5 is sufficient, you dont have to change it. So you can just create the backup using the url ```http://migration_server:8280/admin/backup.action```. Make sure to tick on "Archive to backups folder" also. After creating the backup copy the xml zip file to a safe place as we will need it on the ```new_production_server``` soon. 

While creating the XML backup for my new v.5.10.0 confluence installation I again run into some errors. See Appendix C. for detailed information. Here you can fix these errors without having to restore the database or web application. Just fix the database and retry XML backup. You can now continue with the new production server. Optionally you can now create another mysql and web application backup to have a working status quo and store it in a safe place. 

Part II Migrating confluence to a new production server
=======
In the second part we will do
* Install new ```new_production_server``` with CentOS 7 

* Prepare new ```new_production_server``` with Oracle Java 8 and Postgres and configure the system

* Install confluence

* Migrate the XML zip file backup confluence backup to the new server (use a different DBMS namely postgres and restore the xml zip backup we did in the previous step)


We start with a vanilla CentOS 7 server system. Confluence 5.10 needs Oracle Java 8, so lets start installing/configuring it, therefore login as root to ```new_production_server``` and type:
```
new_production_server$ mkdir ~/Downloads
new_production_server$ cd $_
new_production_server$ wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u72-b15/jdk-8u91-linux-x64.tar.gz"
new_production_server$ cd /opt
new_production_server$ tar xzvf ~/Downloads/jdk-8u91-linux-x64.tar.gz
```

```bash
new_production_server$ alternatives --install /usr/bin/java java /opt/jdk1.8.0_91/bin/java 2
new_production_server$ alternatives --set java /opt/jdk1.8.0_91/bin/java
new_production_server$ alternatives --install /usr/bin/jar jar /opt/jdk1.8.0_91/bin/jar 2
new_production_server$ alternatives --install /usr/bin/javac javac /opt/jdk1.8.0_91/bin/javac 2
new_production_server$ alternatives --set jar /opt/jdk1.8.0_91/bin/jar
new_production_server$ alternatives --set javac /opt/jdk1.8.0_91/bin/javac 

new_production_server$ echo "
export JAVA_HOME=/opt/jdk1.8.0_91
export JRE_HOME=/opt/jdk1.8.0_91/jre
export PATH=$PATH:/opt/jdk1.8.0_91/bin:/software/jdk1.8.0_91/jre/bin
"> /etc/profile.d/java.sh
new_production_server$ source /etc/profile.d/java.sh
```

Now we need to install postgresql server like on the new_production_server
```bash
new_production_server$ yum install postgresql postgresql-server -y; systemctl enable postgresql;
```

init postgres db
```
new_production_server$ postgresql-setup initdb
new_production_server$ systemctl start postgresql
```

set postgresql password (admin user postgres):

```
new_production_server$ su - postgres -c "psql --command '\password postgres'"
```

change postgres hba settings so we have a normal md5 based postgres login authentication

```
new_production_server$ cp /var/lib/pgsql/data/pg_hba.conf /var/lib/pgsql/data/pg_hba.conf.BAK
new_production_server$ sed -i 's/^\(local.*\)peer$/\1md5/g' /var/lib/pgsql/data/pg_hba.conf
new_production_server$ systemctl restart postgresql
```

test if you can login to postgres using shell (type ```\q```) to exit:
```
new_production_server$ psql -U postgres
```

now create the confluence db and user (replace ```<username>``` with something appropriate such as ```confluenceuser``` and ```<database-name>``` with e.g. ```confluenceprod```), give a very secure pasword.
Write down your new postgres user, db and password because we will need it in a minute:

```bash
new_production_server$ createuser -U postgres -P <username>
new_production_server$ createdb -U postgres <database-name> -O <username> --encoding utf-8
```

test if we can login locally with our new confluence user name
```
new_production_server$ psql -U <username> -l
```

Next change all local user authentication from ident to md5 (which is mandatory for confluence):
```
new_production_server$ sed  's!^\(host[[:space:]]*all[[:space:]]*all[[:space:]]*127.0.0.1/32[[:space:]]*\)ident$!\1md5!g' /var/lib/pgsql/data/pg_hba.conf
new_production_server$ systemctl restart postgresql

```

## Install confluence 5.10.0

First have your Confluence license key ready, you find it if your login to the 'my atlassian' webpage. Now download confluence 5.9 from your 'my atlassian' web page (login to your account using https://my.atlassian.com) and transfer it to the server. In our example we will use: 
```atlassian-confluence-5.10.0.tar.gz``` (Download confluence server standalone->Confluence 5.10.0 - Standalone (TAR.GZ Archive) - it should be the same version as the XML zip file you generated on the ```migration_server``` (but notice: we used the ```.bin``` installer on the migration_server instead)
Please note: You can also modify your confluence installation directory structure. Here I just show you where I normally install confluence to.

```
new_production_server$ mkdir -p /opt/confluence
new_production_server$ cd /opt/confluence
new_production_server$ tar xvf ~/Downloads/atlassian-confluence-5.10.0.tar.gz
```
create a symbolic link to ```/opt/confluence/confluence```

```
new_production_server$ ln -s /opt/confluence/atlassian-confluence-5.10.0/ /opt/confluence/confluence
```

next create a confluence data dir:

```
new_production_server$ mkdir /opt/confluence/data
```

now define the following env variables:

```bash
new_production_server$ export CONFL_HOME=/opt/confluence/data
new_production_server$ export CONFL_WEB=/opt/confluence/confluence
```

Define your Confluence Home Directory
```
new_production_server$ vi $CONFL_WEB/confluence/WEB-INF/classes/confluence-init.properties
```

here in this file change the line ```# confluence.home=c:/confluence/data``` to:

```
new_production_server$ confluence.home=/opt/confluence/data
```

now you can make some settings to the applications Java heap space, listeing port, firewall settings, see the TODO: create link migration_server section of this document for more infos.
``` 
new_production_server$ vi $CONFL_WEB/bin/setenv.sh
```
increase ```-Xmx1024m``` to e.g. ```-Xmx3000m```  (in env variable CATALINA_OPTS)
Next if you need change the listening port, enable ssl on local machine or change other settings etc in (otherwise skip):
```bash
new_production_server$ vi $CONFL_WEB/conf/server.xml
```
Next for accessing confluence in your network you need to open the firewall port 8090 etc. if you are running a local firewall etc., for me on CentOS 7 it was (this is optionally if you want to access your Confluence migration installation from another machine in the same network):
```bash
new_production_server$ firewall-cmd --permanent --add-port=8090/tcp
new_production_server$ firewall-cmd --reload
```
Now start the server in the FOREGROUND TO SEE WHATS GOING ON 
```bash
new_production_server$ sh $CONFL_WEB/bin/catalina.sh run
```
Open another window and tail the log file using:
```bash
new_production_server$ tail -f $CONFL_HOME/logs/atlassian-confluence.log
```
After confluence has been loaded (wait for log file output ```Server startup in xxx ms```), open a webbrowser and  browse to the webpager
```
http://new_production_server:8090/.
```

There click on ```[] Production Installation``` then paste your license key you got from your "my atlassian website" 

Next configure your Postgres external database, use "Direct JDBC Connection" , then type in your database and user credentials you created in an earlier step.
Next click on "[] Restore From Backup".

Keeping this window open now transfer the XML zip backup we made in an earlier step from the ```migration_server``` to this new ```new_production_server``` e.g.:
```
rsync -rav user@migration_server:/data/opt/confluence/data/temp/xmlexport-20160321-112943-1.zip /opt/confluence/data/restore --progress
```
Refresh this Restore data page in the browser, then the uploaded ```.xml.zip``` file will appear, select the uploaded xml.zip backup file from the file select menu and start the restoring process by clicking the Restore button!

Wait until the restoring process has completed (took me about 40 minutes). It should print out the message ```Complete!```. Do not interrupt the restoring process or you have to restart from scratch.

## The Aftermath of restoring the confluence wiki
First stop confluence running in your shell in foreground mode (ctrl+c).

Next we will set confluence to listen to port 80 using Apache httpd modproxy module
```
new_production_server$ yum install httpd -y; systemctl enable httpd; systemctl start httpd;
new_production_server$ firewall-cmd --permanent --add-service=http;firewall-cmd --reload
```
now we need to define a context path for our application, see [here](https://confluence.atlassian.com/doc/using-apache-with-mod_proxy-173669.html) for more info
open $CONFL_WEB/conf/server.xml
```
<Context path="/wiki" docBase="../confluence" debug="0" reloadable="true">
```
Next we need to configure mod_proxy
new_production_server$ echo "
LoadModule proxy_module /usr/lib/apache2/modules/mod_proxy.so
LoadModule proxy_http_module /usr/lib/apache2/modules/mod_proxy_http.so
<VirtualHost *>
ProxyRequests Off
ProxyPreserveHost On
<Proxy *>
        # Auth changes in 2.4 - see http://httpd.apache.org/docs/2.4/upgrading.html#run-time
    Require all granted
</Proxy>
ProxyPass /wiki http://127.0.0.1:8090/wiki
ProxyPassReverse /wiki http://127.0.0.1:8090/wiki
<Location /confluence>
        # Auth changes in 2.4 - see http://httpd.apache.org/docs/2.4/upgrading.html#run-time
    Require all granted
</Location>
</VirtualHost>
" > /etc/httpd/conf.d/confluence.conf 
new_production_server$ systemctl reload httpd
```

Finally on CentOS 7 to make our httpd port rewrite work you must dig a hole in SELinux (see [here] (http://sysadminsjourney.com/content/2010/02/01/apache-modproxy-error-13permission-denied-error-rhel/))

```
setsebool -P httpd_can_network_connect 1
```
Also run confluence on boot using a systemd service file under a non-privileged user account.
```
useradd confluence
chown confluence:confluence /opt/confluence -R 

echo "
[Unit]
Description=Atlassian Confluence Service
After=syslog.target network.target

[Service]
User=confluence
Type=forking
ExecStart=/opt/confluence/confluence/bin/catalina.sh start
ExecStop=/opt/confluence/confluence/bin/catalina.sh stop

[Install]
WantedBy=multi-user.target
" > /usr/lib/systemd/system/confluence.service 

systemctl enable confluence
systemctl start confluence
systemctl status confluence
```


finally you are able to use port 80 e.g. http://new_production_server to browse to the wiki

Next open new confluence wiki admin area and do the following things:

- change base url -> this is very important in order that plugin installation e.g. UPM etc. will work!!!!
- execute Add-ons requiring action, update all plugins, disable all incompatible or paid ones
- rebuild index
- reset Look and feel

Finally install a systemd startup script to enable confluence wiki on boot and run it as user ```confluence```

```
useradd confluence
passwd confluence
chown -R confluence:confluence /opt/confluence
```

```
echo "
[Unit]
Description=Atlassian Confluence Wiki service
After=syslog.target network.target
 
[Service]
Type=forking
User=confluence
ExecStart=/opt/confluence/confluence/bin/catalina.sh start
ExecStop=/opt/confluence/confluence/bin/catalina.sh stop
 
[Install]
WantedBy=multi-user.target
" >  /usr/lib/systemd/system/confluence-wiki.service
systemctl enable confluence-wiki.service
systemctl start confluence-wiki 
```

Now you are done! 

Happy Confluencing!...


## Appendix a. fixing XML backup issues with the database for v.4.0.3
### Fixing database connection closures
First error while doing the v4 XML backup was
```bash
 org.springframework.transaction.TransactionSystemException: Could not commit Hibernate transaction; nested exception is net.sf.hibernate.TransactionException: Commit failed with SQL exception:
    at org.springframework.orm.hibernate.HibernateTransactionManager.doCommit(HibernateTransactionManager.java:514)

caused by: net.sf.hibernate.TransactionException: Commit failed with SQL exception:
    at net.sf.hibernate.transaction.JDBCTransaction.commit(JDBCTransaction.java:71)

caused by: com.mysql.jdbc.exceptions.jdbc4.MySQLNonTransientConnectionException: Communications link failure during commit(). Transaction resolution unknown.
    at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method) 
```
this was caused by some connections which took too long and died, more info and resolution can be found [here](https://confluence.atlassian.com/doc/surviving-database-connection-closures-297665664.html)

The recommendations range enabling validation query, adding ```?autoReconnect=true``` to your JDBC string and increasing the lock wait_timeout, see [here](https://answers.atlassian.com/questions/81339/cannot-rebuild-indexes-hibernate-error)
I did all that but for me it was exactly the lock wait timeout which caused the problems, see [https://answers.atlassian.com/questions/81339/cannot-rebuild-indexes-hibernate-error](here). To fix this I increased the following values in the Mariadb config file ```/etc/my.cnf```
```
innodb_lock_wait_timeout = 10000
wait_timeout = 10000
max_allowed_packet = 256M
```
Afterwards I restarted the MariaDB service ```systemctl restart mariadb```
I also did the other suggestion, but which did not have any impact but anyways add them to ```$CONFL_HOME/confluence.cfg.xml``` 
```
<property name="hibernate.connection.url">jdbc:mysql://localhost/confluencedb?autoReconnect=true</property>
...
<property name="hibernate.c3p0.validate">true</property>
<property name="hibernate.c3p0.preferredTestQuery">select 1</property>
...
```
Now restart confluence and rerun the upgrade process. Next error was:

### Fixing no row with given identifier error
After fixing the above the next error which killed the XML generation process could be found in ```$CONFL_HOME/logs/atlassian-confluence.log```

```bash
2016-04-12 12:37:31,779 ERROR [http-8280-1] [confluence.importexport.impl.BackupExporter] backupEntities Couldn't backup database data.
 -- referer: http://127.0.0.1:8280/wiki/admin/backup.action | url: /wiki/admin/dobackup.action | userName: johndoe | action: dobackup
net.sf.hibernate.LazyInitializationException: Exception initializing proxy: [com.atlassian.confluence.core.ContentEntityObject#23855986]
        at net.sf.hibernate.proxy.LazyInitializer.initializeWrapExceptions(LazyInitializer.java:64)
        at net.sf.hibernate.proxy.LazyInitializer.getImplementation(LazyInitializer.java:164)
        at com.atlassian.confluence.importexport.impl.XMLDatabinder.maybeInitializeIfProxy(XMLDatabinder.java:266)
        at com.atlassian.confluence.importexport.impl.XMLDatabinder.associatedObjectFound(XMLDatabinder.java:743)
        at com.atlassian.confluence.importexport.impl.XMLDatabinder.expandEntity(XMLDatabinder.java:686)
        at com.atlassian.confluence.importexport.impl.XMLDatabinder.collectObjectGraph(XMLDatabinder.java:633)
        at com.atlassian.confluence.importexport.impl.XMLDatabinder.toGenericXML(XMLDatabinder.java:144)
        at com.atlassian.confluence.importexport.impl.AbstractXmlExporter.backupEntities(AbstractXmlExporter.java:191)
        at com.atlassian.confluence.importexport.impl.AbstractXmlExporter.backupEverything(AbstractXmlExporter.java:99)
        at com.atlassian.confluence.importexport.impl.FileXmlExporter.backupEverything(FileXmlExporter.java:83)
        at com.atlassian.confluence.importexport.impl.AbstractXmlExporter.doExport(AbstractXmlExporter.java:92)
        at com.atlassian.confluence.importexport.impl.FileXmlExporter.doExport(FileXmlExporter.java:42)
        at com.atlassian.confluence.importexport.DefaultImportExportManager.exportAs(DefaultImportExportManager.java:99)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
```
going a bit above this error message the following debug output could be found
```bash
2016-04-12 12:37:31,768 TRACE [http-8280-1] [sf.hibernate.impl.SessionImpl] doLoad object not resolved in any cache [com.atlassian.confluence.core.ContentEntityObject#23855986]
2016-04-12 12:37:31,768 DEBUG [http-8280-1] [net.sf.hibernate.SQL] log select contentent0_.CONTENTID as CONTENTID0_, contentent0_.CONTENTTYPE as CONTENTT2_0_, contentent0_.TITLE as TITLE0_, 
contentent0_.VERSION as VERSION0_, contentent0_.CREATOR as CREATOR0_, contentent0_.CREATIONDATE as CREATION6_0_, contentent0_.LASTMODIFIER as LASTMODI7_0_, contentent0_.LASTMODDATE as LASTMO
DD8_0_, contentent0_.VERSIONCOMMENT as VERSIONC9_0_, contentent0_.PREVVER as PREVVER0_, contentent0_.CONTENT_STATUS as CONTENT11_0_, contentent0_.SPACEID as SPACEID0_, contentent0_.CHILD_POS
ITION as CHILD_P13_0_, contentent0_.PARENTID as PARENTID0_, contentent0_.MESSAGEID as MESSAGEID0_, contentent0_.DRAFTPAGEID as DRAFTPA16_0_, contentent0_.DRAFTSPACEKEY as DRAFTSP17_0_, conte
ntent0_.DRAFTTYPE as DRAFTTYPE0_, contentent0_.DRAFTPAGEVERSION as DRAFTPA19_0_, contentent0_.PAGEID as PAGEID0_, contentent0_.PARENTCOMMENTID as PARENTC21_0_, contentent0_.USERNAME as USERN
AME0_ from CONTENT contentent0_ where contentent0_.CONTENTID=?
2016-04-12 12:37:31,768 TRACE [http-8280-1] [sf.hibernate.type.LongType] nullSafeSet binding '23855986' to parameter: 1
2016-04-12 12:37:31,769 DEBUG [http-8280-1] [sf.hibernate.impl.SessionImpl] initializeNonLazyCollections initializing non-lazy collections
2016-04-12 12:37:31,769 ERROR [http-8280-1] [sf.hibernate.proxy.LazyInitializer] initializeWrapExceptions Exception initializing proxy
 -- referer: http://127.0.0.1:8280/wiki/admin/backup.action | url: /wiki/admin/dobackup.action | userName: johndoe | action: dobackup
net.sf.hibernate.ObjectNotFoundException: No row with the given identifier exists: 23855986, of class: com.atlassian.confluence.core.ContentEntityObject

```
googling for the error message: ```net.sf.hibernate.ObjectNotFoundException: No row with the given identifier exists: , of class: com.atlassian.confluence.core.ContentEntityObject``` let me to [https://confluence.atlassian.com/confkb/cannot-create-xml-backup-due-to-objectnotfoundexception-no-row-with-given-identifier-180289731.html](this here).

after applying the material from the document found through google I could found out the exact culprit and delete it:
verify we have the bad ID from the exception output ```23855986```
```
echo "select CONTENTID  from BODYCONTENT WHERE CONTENTID NOT IN (SELECT CONTENTID FROM CONTENT);" | mysql -u root -p confluencedb
```
Now apply
```
echo "delete from BODYCONTENT WHERE CONTENTID NOT IN (SELECT CONTENTID FROM CONTENT);" | mysql -u root -p confluencedb
```
Recheck with
```
echo "select CONTENTID  from BODYCONTENT WHERE CONTENTID NOT IN (SELECT CONTENTID FROM CONTENT);" | mysql -u root -p confluencedb
```

Now restart confluence and rerun the upgrade process. For me no further errors occured and I could restart and finally generate xml backup. 


## Appendix B. Fixing errors during upgrading v4.x to v5.x
=======================
After upgrading using the atlassian installer and starting the confluence server after upgrading for the first time I got the following upgrade error (even before loading the web app in the browser)

## Fixing IMAGEDETAILS constraint FKA768048734A4917E error
```bash
2016-04-13 13:39:44,456 ERROR [localhost-startStop-1] [atlassian.confluence.plugin.PluginFrameworkContextListener] launchUpgrades Upgrade failed, application will not start: com.atlassian.config.ConfigurationException: Cannot update schema
com.atlassian.confluence.upgrade.UpgradeException: com.atlassian.config.ConfigurationException: Cannot update schema
	at com.atlassian.confluence.upgrade.AbstractUpgradeManager.upgrade(AbstractUpgradeManager.java:155)
	at com.atlassian.confluence.plugin.PluginFrameworkContextListener.launchUpgrades(PluginFrameworkContextListener.java:118)
	at com.atlassian.confluence.plugin.PluginFrameworkContextListener.contextInitialized(PluginFrameworkContextListener.java:77)
	at org.apache.catalina.core.StandardContext.listenerStart(StandardContext.java:4811)
	at org.apache.catalina.core.StandardContext.startInternal(StandardContext.java:5251)
	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:147)
	at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1408)
	at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1398)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
Caused by: com.atlassian.config.ConfigurationException: Cannot update schema
	at bucket.core.persistence.hibernate.schema.SchemaHelper.updateSchemaIfNeeded(SchemaHelper.java:174)
	at bucket.core.persistence.hibernate.schema.SchemaHelper.updateSchemaIfNeeded(SchemaHelper.java:153)
	at com.atlassian.confluence.upgrade.AbstractUpgradeManager.upgrade(AbstractUpgradeManager.java:139)
	... 11 more
Caused by: com.mysql.jdbc.exceptions.jdbc4.MySQLIntegrityConstraintViolationException: Cannot add or update a child row: a foreign key constraint fails (`confluencedb`.`#sql-2fd3_c9`, CONSTRAINT `FKA768048734A4917E` FOREIGN KEY (`ATTACHMENTID`) REFERENCES `CONTENT` (`CONTENTID`))
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at com.mysql.jdbc.Util.handleNewInstance(Util.java:406)
	at com.mysql.jdbc.Util.getInstance(Util.java:381)
	at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:1038)
	at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3563)
	at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3495)
	at com.mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:1959)
	at com.mysql.jdbc.MysqlIO.sqlQueryDirect(MysqlIO.java:2113)
	at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2687)
	at com.mysql.jdbc.StatementImpl.executeUpdate(StatementImpl.java:1647)
	at com.mysql.jdbc.StatementImpl.executeUpdate(StatementImpl.java:1566)
	at com.mchange.v2.c3p0.impl.NewProxyStatement.executeUpdate(NewProxyStatement.java:410)
	at net.sf.hibernate.tool.hbm2ddl.SchemaUpdate.execute(SchemaUpdate.java:167)
	at bucket.core.persistence.hibernate.schema.SchemaHelper.updateSchemaIfNeeded(SchemaHelper.java:172)
```
The resolution to this FKA768048734A4917E problem can be found [here](https://confluence.atlassian.com/confkb/upgrade-to-5-7-x-fails-due-to-imagedetails-constraint-fka768048734a4917e-732268639.html). 
What I did was using mysql
```
ALTER TABLE IMAGEDETAILS ADD CONSTRAINT FKA768048734A4917E FOREIGN KEY (ATTACHMENTID) REFERENCES ATTACHMENTS(ATTACHMENTID);
```
In order you can do this, restore your original mysql database and confluence installation, then right before starting the upgrade process execute this line (and all the other fixes you collect in this appendix)

## Fixing NOTIFICATIONS.PAGEID column error
Next error was
```bash
2016-04-13 14:17:03,552 INFO [localhost-startStop-1] [confluence.upgrade.upgradetask.NotificationPageColumnUpgradeTask] dropFKConstraints Dropping FK's on NOTIFICATIONS.PAGEID column ...
2016-04-13 14:17:03,570 ERROR [localhost-startStop-1] [atlassian.confluence.plugin.PluginFrameworkContextListener] launchUpgrades Upgrade failed, application will not start: Could not determ
ine foreign keys for NOTIFICATIONS.PAGEID column
com.atlassian.confluence.upgrade.UpgradeException: Could not determine foreign keys for NOTIFICATIONS.PAGEID column
        at com.atlassian.confluence.upgrade.upgradetask.NotificationPageColumnUpgradeTask.dropFKConstraints(NotificationPageColumnUpgradeTask.java:89)
        at com.atlassian.confluence.upgrade.upgradetask.NotificationPageColumnUpgradeTask.doUpgrade(NotificationPageColumnUpgradeTask.java:58)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.springframework.aop.support.AopUtils.invokeJoinpointUsingReflection(AopUtils.java:307)
        at org.springframework.aop.framework.ReflectiveMethodInvocation.invokeJoinpoint(ReflectiveMethodInvocation.java:182)
        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:149)
        at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:106)
        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:171)
        at org.springframework.aop.framework.JdkDynamicAopProxy.invoke(JdkDynamicAopProxy.java:204)
        at com.sun.proxy.$Proxy33.doUpgrade(Unknown Source)
        at com.atlassian.confluence.upgrade.AbstractUpgradeManager$UpgradeStep$3.execute(AbstractUpgradeManager.java:636)
        at com.atlassian.confluence.upgrade.AbstractUpgradeManager.executeUpgradeTask(AbstractUpgradeManager.java:243)
        at com.atlassian.confluence.upgrade.AbstractUpgradeManager.executeUpgradeStep(AbstractUpgradeManager.java:223)
        at com.atlassian.confluence.upgrade.AbstractUpgradeManager.runSchemaUpgradeTasks(AbstractUpgradeManager.java:185)
        at com.atlassian.confluence.upgrade.AbstractUpgradeManager.upgrade(AbstractUpgradeManager.java:140)
        at com.atlassian.confluence.plugin.PluginFrameworkContextListener.launchUpgrades(PluginFrameworkContextListener.java:118)
        at com.atlassian.confluence.plugin.PluginFrameworkContextListener.contextInitialized(PluginFrameworkContextListener.java:77)
        at org.apache.catalina.core.StandardContext.listenerStart(StandardContext.java:4811)
        at org.apache.catalina.core.StandardContext.startInternal(StandardContext.java:5251)
        at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:147)
        at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1408)
        at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1398)
        at java.util.concurrent.FutureTask.run(FutureTask.java:266)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)
```
Reason for this can be found [here](https://confluence.atlassian.com/confkb/confluence-upgrade-fails-with-could-not-determine-foreign-keys-for-notifications-pageid-column-error-on-mysql-598840995.html)
I fixed this using
```
ALTER TABLE NOTIFICATIONS ADD CONSTRAINT FK594ACC88C38FBEA FOREIGN KEY (PAGEID) REFERENCES CONTENT (CONTENTID);
```
As usual: Fix this error on the rollbacked mysql 4 backup right before restarting upgrade.

## Fixing error SCHEMA_UPGRADE phase due to: null
Next error was

```
2016-04-13 14:47:05,775 ERROR [localhost-startStop-1] [atlassian.confluence.plugin.PluginFrameworkContextListener] launchUpgrades Upgrade failed, application will not start: Upgrade task com
.atlassian.confluence.upgrade.upgradetask.DropSpaceGroupTablesUpgradeTask@5c841823 failed during the SCHEMA_UPGRADE phase due to: null
com.atlassian.confluence.upgrade.UpgradeException: Upgrade task com.atlassian.confluence.upgrade.upgradetask.DropSpaceGroupTablesUpgradeTask@5c841823 failed during the SCHEMA_UPGRADE phase d
ue to: null
        at com.atlassian.confluence.upgrade.AbstractUpgradeManager.executeUpgradeStep(AbstractUpgradeManager.java:229)
        at com.atlassian.confluence.upgrade.AbstractUpgradeManager.runSchemaUpgradeTasks(AbstractUpgradeManager.java:185)
        at com.atlassian.confluence.upgrade.AbstractUpgradeManager.upgrade(AbstractUpgradeManager.java:140)
        at com.atlassian.confluence.plugin.PluginFrameworkContextListener.launchUpgrades(PluginFrameworkContextListener.java:118)
        at com.atlassian.confluence.plugin.PluginFrameworkContextListener.contextInitialized(PluginFrameworkContextListener.java:77)
        at org.apache.catalina.core.StandardContext.listenerStart(StandardContext.java:4811)
        at org.apache.catalina.core.StandardContext.startInternal(StandardContext.java:5251)
        at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:147)
        at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1408)
        at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1398)
```
Reason for this error can be found [here](https://confluence.atlassian.com/confkb/upgrading-to-confluence-5-9-x-fails-with-error-failed-during-the-schema_upgrade-phase-due-to-null-794212749.html)
Applying the fix for me was
```
show create table SPACES;

```
As there was no foreign kea available/ present we have to apply this patch
```
ALTER TABLE `SPACES` ADD CONSTRAINT `FK9228242D16994414` FOREIGN KEY (`SPACEGROUPID`) REFERENCES `SPACEGROUPS` (`SPACEGROUPID`);
```

## Fixing plugin errors

This was the last upgrading error. What followed were some plugin errors I checked and resolved (this is not mandantory).
For plugin errors we need to make log4j more verbose. Replace the following line in $CONFL_WEB/confluence/WEB-INF/classes/log4j.properties to 

```
log4j.logger.com.atlassian.plugin=DEBUG
```
More information about plugin errors of this kind can be found [here](https://confluence.atlassian.com/confkb/org-osgi-framework-bundleexception-problems-when-loading-plugins-due-to-class-loading-191005907.html)


After fixing plugin errors restart the server (you dont need to restore confluence or the database!)

## Appendix C.

Errors while creating the XML backup for confluence 5.10.0

```
com.atlassian.confluence.importexport.ImportExportException: java.lang.RuntimeException: Could not read fields for table AO_187CCC_SIDEBAR_LINK at com.atlassian.activeobjects.confluence.backup.ActiveObjectsBackupRestoreProvider.backup(ActiveObjectsBackupRestoreProvider.java:30) at com.atlassian.confluence.importexport.impl.FileXmlExporter.backupPluginData(FileXmlExporter.java:217) at com.atlassian.confluence.importexport.impl.FileXmlExporter.backupEverything(FileXmlExporter.java:110) at com.atlassian.confluence.importexport.impl.AbstractXmlExporter.doExport(AbstractXmlExporter.java:88) at com.atlassian.confluence.importexport.impl.FileXmlExporter.doExportInternal(FileXmlExporter.java:61) at com.atlassian.confluence.importexport.impl.FileXmlExporter.doExport(FileXmlExporter.java:53) at com.atlassian.confluence.importexport.DefaultImportExportManager.doExport(DefaultImportExportManager.java:124) at com.atlassian.confluence.importexport.DefaultImportExportManager.exportAs(DefaultImportExportManager.java:94) at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) at 
```

Solution could be found [here](https://confluence.atlassian.com/confkb/cannot-create-xml-backup-due-to-could-not-read-fields-for-table-unknown-activeobjects-table-306348859.html)

What did I do to resolve it
```
drop table AO_187CCC_SIDEBAR_LINK;
```

Next error while creating XML backup was
```
org.springframework.orm.hibernate.HibernateSystemException: could not update: [com.atlassian.confluence.pages.Attachment#1081420]; nested exception is net.sf.hibernate.HibernateException: could not update: [com.atlassian.confluence.pages.Attachment#1081420]
	at org.springframework.orm.hibernate.SessionFactoryUtils.convertHibernateAccessException(SessionFactoryUtils.java:597)
	at org.springframework.orm.hibernate.SpringSessionSynchronization.beforeCommit(SpringSessionSynchronization.java:137)
	at org.springframework.transaction.support.TransactionSynchronizationUtils.triggerBeforeCommit(TransactionSynchronizationUtils.java:95)
	at org.springframework.transaction.support.AbstractPlatformTransactionManager.triggerBeforeCommit(AbstractPlatformTransactionManager.java:932)
	at org.springframework.transaction.support.AbstractPlatformTransactionManager.processCommit(AbstractPlatformTransactionManager.java:744)
	at org.springframework.transaction.support.AbstractPlatformTransactionManager.commit(AbstractPlatformTransactionManager.java:730)
	at sun.reflect.GeneratedMethodAccessor59.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.aop.support.AopUtils.invokeJoinpointUsingReflection(AopUtils.java:302)
	at org.springframework.aop.framework.JdkDynamicAopProxy.invoke(JdkDynamicAopProxy.java:202)
	at com.sun.proxy.$Proxy32.commit(Unknown Source)
	at com.atlassian.xwork.interceptors.TransactionalInvocation.commitOrRollbackTransaction(TransactionalInvocation.java:107)
.....
```
Solution can be found here 
https://confluence.atlassian.com/doc/troubleshooting-failed-xml-site-backups-149212.html

