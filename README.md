# Spark client running into YARN cluster in Docker

Apache Spark is an open-source, distributed processing system used for big data workloads.

In this demo, a Spark container uses a Hadoop YARN cluster as a resource management and job scheduling technology to perform distributed data processing.

This Docker image contains Spark binaries prebuilt and uploaded in Docker Hub.

## Steps to Build Spark image
```shell
$ git clone https://github.com/mkenjis/apache_binaries
$ wget https://archive.apache.org/dist/spark/spark-2.3.2/spark-2.3.2-bin-hadoop2.7.tgz
$ docker image build -t mkenjis/ubspkcluster1_img
$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: mkenjis
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
$ docker image push mkenjis/ubspkcluster1_img
```

## Shell Scripts Inside 

> run_spark.sh

Sets up the environment for Spark client by executing the following steps :
- sets environment variables for JAVA and SPARK
- starts the SSH service for passwordless SSHfiles on start-up

> create_conf_files.sh

Creates the following Hadoop files on $SPARK_HOME/conf directory :
- core-site.xml
- hdfs-site.xml
- hive-site.xml
- spark-env.sh

## Initial Steps on Docker Swarm

Start Swarm mode in Docker in node1
```shell
$ docker swarm init
Swarm initialized: current node (xv7mhbt8ncn6i9iwhy8ysemik) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token <token> <IP node1>:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

Add more workers in cluster hosts (node2, node3, ...) by joining them to manager.
```shell
$ docker swarm join --token <token> <IP node1>:2377
```

Change the workers as managers in node2, node3, ...
```shell
$ docker node promote node2
$ docker node promote node3
$ docker node promote ...
```

Start Docker stack using docker-compose.yml 
```shell
$ docker stack deploy -c docker-compose.yml yarn
```

Check the status of each service started
```shell
$ docker service ls
ID             NAME           MODE         REPLICAS   IMAGE                                 PORTS
io5i950qp0ac   yarn_hdp1      replicated   0/1        mkenjis/ubhdpclu_img:latest           
npmcnr3ihmb4   yarn_hdp2      replicated   0/1        mkenjis/ubhdpclu_img:latest           
uywev8oekd5h   yarn_hdp3      replicated   0/1        mkenjis/ubhdpclu_img:latest           
p2hkdqh39xd2   yarn_hdpmst    replicated   1/1        mkenjis/ubhdpclu_img:latest           
xf8qop5183mj   yarn_spk_cli   replicated   0/1        mkenjis/ubspkcluster1_img:latest
```

## Steps to Set up Spark client container

Identify which Docker container started as Hadoop master and logged into it
```shell
$ docker container ls   # run it in each node and check which <container ID> is running the Hadoop master constainer
CONTAINER ID   IMAGE                         COMMAND                  CREATED              STATUS              PORTS      NAMES
a8f16303d872   mkenjis/ubhdpclu_img:latest   "/usr/bin/supervisord"   About a minute ago   Up About a minute   9000/tcp   yarn_hdp2.1.kumbfub0cl20q3jhdyrcep4eb
77fae0c411ce   mkenjis/ubhdpclu_img:latest   "/usr/bin/supervisord"   About a minute ago   Up About a minute   9000/tcp   yarn_hdpmst.1.r81pn190785n1hdktvrnovw86

$ docker container exec -it <container ID> bash
```

Copy the setup_spark_files.sh into Hadoop master container.

Run it to copy the Hadoop conf files into Spark client container.
```shell
$ vi setup_spark_files.sh
$ chmod u+x setup_spark_files.sh
$ ping spk_cli          
PING spk_cli (10.0.2.11) 56(84) bytes of data.
64 bytes from 10.0.2.11 (10.0.2.11): icmp_seq=1 ttl=64 time=0.163 ms
64 bytes from 10.0.2.11 (10.0.2.11): icmp_seq=2 ttl=64 time=0.116 ms
^C
--- spk_cli ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.116/0.139/0.163/0.023 ms
$ ./setup_spark_files.sh
Warning: Permanently added 'spk_cli,10.0.2.11' (ECDSA) to the list of known hosts.
core-site.xml                                                      100%  137    75.8KB/s   00:00    
hdfs-site.xml                                                      100%  310   263.4KB/s   00:00    
yarn-site.xml                                                      100%  771   701.6KB/s   00:00
```

Identify which Docker container started as Spark client and logged into it
```shell
$ docker container ls   # run it in each node and check which <container ID> is running the Spark client constainer
CONTAINER ID   IMAGE                                 COMMAND                  CREATED         STATUS         PORTS                                          NAMES
8f0eeca49d0f   mkenjis/ubspkcluster1_img:latest   "/usr/bin/supervisord"   3 minutes ago   Up 3 minutes   4040/tcp, 7077/tcp, 8080-8082/tcp, 10000/tcp   yarn_spk_cli.1.npllgerwuixwnb9odb3z97tuh
e9ceb97de97a   mkenjis/ubhdpclu_img:latest           "/usr/bin/supervisord"   4 minutes ago   Up 4 minutes   9000/tcp                                       yarn_hdp1.1.58koqncyw79aaqhirapg502os

$ docker container exec -it <container ID> bash
```

Add the following parameters to $SPARK_HOME/conf/spark-defaults.conf
```shell
$ vi $SPARK_HOME/conf/spark-defaults.conf
spark.driver.memory  1024m
spark.yarn.am.memory 1024m
spark.executor.memory  1536m
```

Inside the Spark client container, start spark-shell
```shell
$ spark-shell --master yarn
2021-12-05 11:09:14 WARN  NativeCodeLoader:62 - Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
2021-12-05 11:09:40 WARN  Client:66 - Neither spark.yarn.jars nor spark.yarn.archive is set, falling back to uploading libraries under SPARK_HOME.
Spark context Web UI available at http://802636b4d2b4:4040
Spark context available as 'sc' (master = yarn, app id = application_1638723680963_0001).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.3.2
      /_/
         
Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_181)
Type in expressions to have them evaluated.
Type :help for more information.

scala> 
```


