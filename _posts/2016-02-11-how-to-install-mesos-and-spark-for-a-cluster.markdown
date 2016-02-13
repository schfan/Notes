---
layout: post
comments: true
title:  "How to install Mesos and Spark for a cluster"
date:   2016-02-11 16:22:20 -0500
categories: notes
---
This post is a tutorial on installing Mesos and Spark frameworks on a cluster of servers.

## 1. Install Mesos [[1]](https://open.mesosphere.com/getting-started/install/)

> Note: In this example we use `master.com` as the IP address of the master server. Remember to replace it to your own master server IP address. The bash scripts here require you to have `sudo` privilege for package installation and configuration. It is also assumed that hdfs is already setup in the cluster.

### 1.1 Set up Master 
```bash
# mesos_master_setup.sh
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv E56151BF
DISTRO=$(lsb_release -is | tr '[:upper:]' '[:lower:]')
CODENAME=$(lsb_release -cs)
echo "deb http://repos.mesosphere.com/${DISTRO} ${CODENAME} main" | sudo tee /etc/apt/sources.list.d/mesosphere.list
sudo apt-get -y update
sudo apt-get -y install mesos marathon
sudo sh -c "echo 1 > /etc/zookeeper/conf/myid"
sudo sh -c "echo server.1=master.com:2888:3888 >> /etc/zookeeper/conf/zoo.cfg"
sudo service zookeeper restart
sudo sh -c "echo zk://master.com:2181/mesos > /etc/mesos/zk"
sudo service mesos-slave stop
sudo sh -c "echo manual > /etc/init/mesos-slave.override"
sudo service mesos-master restart
``` 

### 1.2 Set up Slave(s)
```bash
# mesos_slave_setup.sh
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv E56151BF
DISTRO=$(lsb_release -is | tr '[:upper:]' '[:lower:]')
CODENAME=$(lsb_release -cs)
echo "deb http://repos.mesosphere.com/${DISTRO} ${CODENAME} main" | sudo tee /etc/apt/sources.list.d/mesosphere.list
sudo apt-get -y update
sudo apt-get -y install mesos
sudo service zookeeper stop
sudo sh -c "echo manual > /etc/init/zookeeper.override"
sudo sh -c "echo zk://master.com:2181/mesos > /etc/mesos/zk"
sudo service mesos-master stop
sudo sh -c "echo manual > /etc/init/mesos-master.override"
```

### 1.3 Launch a slave
```bash
# mesos_slave_start.sh
sudo rm -f /tmp/mesos/meta/slaves/latest
sudo mesos-slave --master=master.com:5050 --hadoop_home=/usr/local/hadoop --hostname=$(hostname)
```

> At this point, you are able to view the Mesos Web UI at `master.com:5050`.

## 2. Install Spark on top of Mesos [[2]](https://spark.apache.org/docs/latest/running-on-mesos.html)

### 2.1 Configuration
> This step can be done in any server inside the cluster. You might need to log in as the hadoop user in order to upload Spark binary files onto hdfs. 

```bash
# mesos_spark_setup.sh
su hduser1
wget http://www.apache.org/dyn/closer.lua/spark/spark-1.6.0/spark-1.6.0-bin-hadoop2.6.tgz
hadoop fs -put spark-1.6.0-bin-hadoop2.6.tgz / 
tar xvf spark-1.6.0-bin-hadoop2.6.tgz
cd spark-1.6.0-bin-hadoop2.6
hadoop fs -put ./lib/spark-examples-1.6.0-hadoop2.6.0.jar /
```

### 2.2 Launch Spark Dispatcher
```bash
./sbin/start-mesos-dispatcher.sh --master mesos://zk://master.com:2181/mesos
```

### 2.3 Submit a Spark Job to Mesos
```bash
./bin/spark-submit --class org.apache.spark.examples.SparkPi --master mesos://master.com:7078 --deploy-mode cluster  --supervise --executor-memory 20G --total-executor-cores 20 hdfs://master:54310/spark-examples-1.6.0-hadoop2.6.0.jar 50000
```

> The specific port numbers might vary according to different system settings. If the job submission fails, check the error logs. Make sure that the firewalls do not block required ports. 

### 2.4 Stop Spark Dispatcher
> After your job finishes (which can be verified on the Web UI), you can stop the Spark job dispatcher.
```bash
./sbin/stop-mesos-dispatcher.sh
```
