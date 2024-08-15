# Falcon Logscale Self Hosted Clusters Set Up Guide
## Doc Updated As of: 15/08/2024

## Requirements

- You need to be able to hold 48 hours of compressed data in 80% of your RAM.
- You want enough hyper-threads/vCPUs (each giving you 1GB/s search) to be able to search 24 hours of data in less than 10 seconds.
- You need disk space to hold your compressed data. Never fill your disk more than 80%.

Falcon Logscale recommends at least 16 CPU cores, 32 GB of memory, and a 1 GBit Network card for single server set up on production environment.
Disk space will depend on the amount of ingested data per day and the number of retention days.
This is calculated as follows: Retention Days x GB Injected / Compression Factor. That will determine the needed disk space for a single server.

**Set Up Used in this Doc:**

**Data:** 200GB partition at /data partition

**RAM:** 32GB

**CPU:** 8 Cores

**Kafka Version:** 2.13-3.7.0

**Humio Falcon Version:** 1.142.1





**Increase Open File Limit:**

For production usage, Humio needs to be able to keep a lot of files open for sockets and actual files from the file system.

You can verify the actual limits for the process using:

```
PID=`ps -ef | grep java | grep humio-assembly | head -n 1 | awk '{print $2}'`
cat /proc/$PID/limits | grep 'Max open files'
```

The minimum required settings depend on the number of open network connections and datasources. **There is no harm in setting these limits high for the falcon process. A value of at least 8192 is recommended**.

You can do that using a simple text editor to create a file named **99-falcon-limits.conf** in the **/etc/security/limits.d/** sub-directory. Copy these lines into that file:

```
#Raise limits for files:
falcon soft nofile 250000
falcon hard nofile 250000
```

Create another file with a text editor, this time in the **/etc/pam.d/** sub-directory, and name it **common-session**. Copy these lines into it:

```
#Apply limits:
session required pam_limits.so
```

**Check noexec on /tmp directory**

Check the filesystem options on /tmp. Falcon Logscale makes use of the Facebook Zstandard real-time compression algorithm, which requires the ability to execute files directly from the configured temporary directory.

The options for the filesystem can be checked using mount:

```
$ mount
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
udev on /dev type devtmpfs (rw,nosuid,noexec,relatime,size=1967912k,nr_inodes=491978,mode=755,inode64)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
tmpfs on /run type tmpfs (rw,nosuid,nodev,noexec,relatime,size=399508k,mode=755,inode64)
/dev/sda5 on / type ext4 (rw,relatime,errors=remount-ro)
tmpfs on /tmp type tmpfs (rw,nosuid,nodev,noexec,seclabel)

```

You can temporarily remove noexec using **mount** to 'remount' the directory:

```
mount -oremount,exec /tmp
```
 

To permanently remove the noexec flag, update **/etc/fstab** to remove the flag from the options:

```
tmpfs /tmp tmpfs mode=1777,nosuid,nodev 0 0
```

## Installing JDK 21

**Humio requires a Java version 17 or later JVM to function properly.** (Doc updated to use JDK 21 as Logscale will drop support for JDK version below 17 Soon)

```
wget https://cdn.azul.com/zulu/bin/zulu21.36.19-ca-crac-jdk21.0.4-linux.x86_64.rpm
rpm -i zulu21.36.19-ca-crac-jdk21.0.4-linux.x86_64.rpm
```

## Installing Kafka

**At first add Kafka system user like the following**

```
$ sudo adduser kafka --shell=/bin/false --no-create-home --system --group
```

Download kafka:

```
$ curl -o kafka_2.13-3.7.0.tgz https://downloads.apache.org/kafka/3.7.0/kafka_2.13-3.7.0.tgz
```

**Create the following directories**
```
$ mkdir /data /opt/kafka
```

**Untar kafka to /opt/kafka directory**

```
$ sudo tar -zxf kafka_2.13-3.7.0.tgz /opt/kafka/
```

**Create the following directory**

```
$ sudo mkdir -p /data/kafka/ /data/kafka/kafka-data /data/kafka/zookeeper-data /var/log/kafka /var/log/zookeeper
$ sudo chown -R kafka:kafka /data/kafka
$ sudo chown -R kafka:kafka /opt/kafka
```

Navigate to `/opt/kafka/config` and then edit the following details of the `server.properties` file.

```
broker.id=1
log.dirs=/data/kafka/log
delete.topic.enable = true
```


**zookeeper configuration.**
To configure the properties for ZooKeeper, edit the ``` kafka/config/zookeeper.properties ``` file with the following options:
```
dataDir=/data/kafka/zookeeper-data
clientPort=2181
maxClientCnxns=0
admin.enableServer=false
server.1=kafka1:2888:3888
server.2=kafka2:2888:3888
server.3=kafka3:2888:3888
4lw.commands.whitelist=*
tickTime=2000
initLimit=5
syncLimit=2
```



Create a `myid` file in the `data` sub-directory with just the number `1` as its contents. This number will be in order for each node. 1,2,3

```
$ echo 1 > /data/kafka/zookeeper-data/myid
$ chown -R kafka:kafka /data/kafka/zookeeper-data
```
**do this for each node.**
**The number in myid must be unique on each host, and match the broker.id configured for Kafka.**

Create a service file for ZooKeeper so that it will run as a system service and be automatically managed to keep running.

Create the file ```/etc/systemd/system/zookeeper.service ``` sub-directory, edit the file add the following lines:
```
[Unit]

[Service]
Type=simple
User=kafka
LimitNOFILE=800000
Environment="LOG_DIR=/var/log/zookeeper"
Environment="GC_LOG_ENABLED=true"
Environment="KAFKA_HEAP_OPTS=-Xms512M -Xmx4G"
ExecStart=/opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
Restart=on-failure
TimeoutSec=900

[Install]
WantedBy=multi-user.target
```


**Then you can start Zookeeper to verify that the configuration is working**

```
$ systemctl start zookeeper
```

**Check if the service is running by using the status command:**
```
$ systemctl status zookeeper
```
**Output similar to the following showing active (running) if the service is OK:**

```
zookeeper.service
     Loaded: loaded (/etc/systemd/system/zookeeper.service; disabled; vendor preset: enabled)
     Active: active (running) since Thu 2024-03-07 05:31:36 GMT; 1s ago
   Main PID: 4968 (java)
      Tasks: 16 (limit: 1083)
     Memory: 24.6M
        CPU: 1.756s
     CGroup: /system.slice/zookeeper.service
             ??4968 java -Xms512M -Xmx4G -server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -XX:MaxInlineLevel=15 -Djava.awt.headless=true "-Xlog:gc*:file=/var/log/zookeeper/zookeeper-gc.log:time,tags:filecount=10,filesize=100M" -Dcom.sun.management.>

Mar 07 05:31:36 kafka1 systemd[1]: Started zookeeper.service.
```

**This should report any issues which should be addressed before starting the service again. If everything is OK, enable the service so that it will always start on boot:**

```
$ systemctl enable zookeeper
```

**Now create a service for Kafka. The configuration file is slightly different because there is a dependency added so that the system will start ZooKeeper first if it is not running before trying to Kafka.**

Create the file /etc/systemd/system/kafka.service sub-directory, edit the file add the following lines:

```
[Unit]
Requires=zookeeper.service
After=zookeeper.service

[Service]
Type=simple
User=kafka
LimitNOFILE=800000
Environment="LOG_DIR=/var/log/kafka"
Environment="GC_LOG_ENABLED=true"
Environment="KAFKA_HEAP_OPTS=-Xms512M -Xmx4G"
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
Restart=on-failure
TimeoutSec=900

[Install]
WantedBy=multi-user.target
```

**Now start the Kafka service:**

```
$ systemctl start kafka
$ systemctl status kafka
$ systemctl enable kafka
```

## Setting up Falcon Logscale

**Create humio user**

```
$ sudo adduser humio --shell=/bin/false --no-create-home --system --group
```

**Creating falcon directories**

```
$ sudo mkdir -p /data/humio/ /data/humio/data /opt/humio /etc/humio/filebeat /var/log/humio
$ chown -R humio:humio /data/humio/ /data/humio/data /opt/humio /etc/humio/filebeat /var/log/humio

```

**We are now ready to download and install Falcon Logscale’s software. Download latest falcon logscale stable version from here:**

[**https://repo.humio.com/service/rest/repository/browse/maven-releases/com/humio/server/**](https://repo.humio.com/service/rest/repository/browse/maven-releases/com/humio/server/)

```
$ cd /opt/humio/
$ wget https://repo.humio.com/service/rest/repository/browse/maven-releases/com/humio/server/1.142.1/
$ tar xzf server-linux_x64-1.142.1.tar.gz
$ sudo chown -R humio:humio /opt/humio/
```

**Using a simple text editor, create the falcon logscale configuration file, `server.conf` in the `/etc/falcon` directory. You will need to enter a few environment variables in this configuration file to run Humio on a single server or instance. Below are those basic settings:**

```
$ sudo vi /etc/humio/server.conf
```

```
AUTHENTICATION_METHOD=single-user
DIRECTORY=/data/humio/data
HUMIO_AUDITLOG_DIR=/var/log/humio
HUMIO_DEBUGLOG_DIR=/var/log/humio
JVM_LOG_DIR=/var/log/humio
HUMIO_PORT=8080
ELASTIC_PORT=9200

KAFKA_SERVERS=kafka1:9092,kafka2:9092,kafka3:9092
EXTERNAL_URL=http://127.0.0.1:8080
PUBLIC_URL=http://127.0.0.1
```

**In the last create the `falcon.service` file inside `/etc/systemd/system/` directory and paste the below contents**

```
[Unit]
Description=LogScale service
After=network.service

[Service]
Type=notify
Restart=on-abnormal
User=humio
Group=humio
LimitNOFILE=250000:250000
EnvironmentFile=/etc/humio/server.conf
WorkingDirectory=/data/humio
ExecStart=/opt/humio/humio/bin/humio-server-start.sh
TimeoutSec=900

[Install]
WantedBy=default.target
```

**Start the falcon logscale server by executing the following command:**

```
$ sudo systemctl start humio
$ sudo systemctl enable humio

```

**Check for any errors in falcon end:**

```
$ journalctl -fu falcon
```

We can also check the logscale logs for errors here: `/var/humio/log`

If there is no error, you can access the falcon logscale site by visiting `http://<serverip>:8080` on your browser.
