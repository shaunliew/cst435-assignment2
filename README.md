# CST435 Assignment 2 - Distributed Hadoop cluster with Docker Swarm

## Team Members

```
Shaun Liew Xin Hong
Liew Hui Lek
Teh Zhen Jun
Poh Chyi Yi
Eng Jia Ying
```

## How to run this project

### Software Installation needed

1. Docker
2. VMWare/Multipass (for creating VM)
3. WSL2 (if using Windows)

### Clone the Project

1. Git clone the project and go to the directory

```
git clone https://github.com/shaunliew/cst435-assignment2

cd cst435-assignment2
```

## Distribution Hadoop Cluster

### Dockerfile used

For this project, we are using public Docker Image for Hadoop. Below is the details of the docker image that we used. 

Docker Image: [newnius/hadoop](https://hub.docker.com/r/newnius/hadoop)

GitHub Repo for Docker Image: [newnius/Dockerfiles/hadoop/2.7.4](https://github.com/newnius/Dockerfiles/tree/master/hadoop/2.7.4)

### Hadoop MapReduce use case

The Hadoop MapReduce use case that we chosen is `π Calculation`.

Description from the Hadoop Official Website:

```
This package consists of a map/reduce application, distbbp, which computes exact binary digits of the mathematical constant π. distbbp is designed for computing the nth bit of π, for large n, say n > 100,000,000. For computing the lower bits of π, consider using bbp.
```

Command used for `π Calculation`:
```
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.4.jar pi <number of map> <samples per map>
```

### prerequisites

#### Install Docker

```
sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install -y docker.io
sudo usermod -aG docker $USER
newgrp docker
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Case 1: Single Computer Multi Nodes

For Case 1: we are using `1 Master Node` and `3 Slave Nodes`

#### Initialize Docker Swarm and listens on localhost and create an overlay network

```
docker swarm init --advertise-addr 127.0.0.1

docker network create --driver overlay swarm-net
```

#### Start Hadoop Cluster using the Public Hadoop Image

##### Start Master Node

```
docker service create \
	--name hadoop-master \
	--network swarm-net \
	--hostname hadoop-master \
	--constraint node.role==manager \
	--replicas 1 \
	--endpoint-mode dnsrr \
	newnius/hadoop:2.7.4
```

##### Start 3 Slave Nodes

```
docker service create \
	--name hadoop-slave1 \
	--network swarm-net \
	--hostname hadoop-slave1 \
	--replicas 1 \
	--endpoint-mode dnsrr \
	newnius/hadoop:2.7.4
```

```
docker service create \
	--name hadoop-slave2 \
	--network swarm-net \
	--hostname hadoop-slave2 \
	--replicas 1 \
	--endpoint-mode dnsrr \
	newnius/hadoop:2.7.4
```

```
docker service create \
	--name hadoop-slave3 \
	--network swarm-net \
	--hostname hadoop-slave3 \
	--replicas 1 \
	--endpoint-mode dnsrr \
	newnius/hadoop:2.7.4
```

#### Start a proxy to access Hadoop web UI

```
docker service create \
	--replicas 1 \
	--name proxy_docker \
	--network swarm-net \
	-p 7001:7001 \
	newnius/docker-proxy
```

#### Run the MapReduce Program - π Calculation

Execute following command to enter the `master node`

```
docker exec -it hadoop-master.1.$(docker service ps \
hadoop-master --no-trunc | tail -n 1 | awk '{print $1}' ) bash
```

Now we will execute the following commands inside the master node

##### Format master node for first time running

```
# stop all Hadoop processes
sbin/stop-yarn.sh
sbin/stop-dfs.sh

# format namenode
bin/hadoop namenode -format

# start yarn and dfs nodes
sbin/start-dfs.sh
sbin/start-yarn.sh
```

##### Run the π Calculation

```
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.4.jar pi 100 150
```

You should see the result after executing the command above.

### Case 2: Multi Computer Multi Nodes

#### Environment used
We use 3 nodes(Virtual Machine) to deploy our Hadoop Cluster.

| Hostname      | Alias | IP address      | Roles |
| ----------- | ----------- |----------- | -----------|
| tehtehv2-virtual-machine      | hadoop-node021| 192.168.100.90      | Name Node, Resource Manager       |
| tehteh22v3-virtual-machine   | hadoop-node022 | 192.168.100.92 | Data Node, Secondary NameNode, Node Manager |
| tehteh22v4-virtual-machine   | hadoop-node023 | 192.168.100.93  | Data Node, Node Manager|

Notes: The alias means the hostname in the Hadoop cluster Due the limitation that we cannot have same (host)name in docker swarm and we may want to deploy other services on the same node, so we’d better choose another name.

Make sure that all of these VM have install `Docker` as shown below.

#### Create a Docker Swarm 

We choose `tehtehv2-virtual-machine` as the `manager` for Docker Swarm. Initialize the Docker Swarm on `tehtehv2-virtual-machine` and create an overlay network.

```
docker swarm init --listen-addr 192.168.100.90
docker network create --driver overlay hadoop-net
```

output
```
docker swarm join --token SWMTKN-1-5f4g4cbo8fdldw98a3hmhf8s12qe6ffx6s2sbluyyihpp5m3bg-eaia06w7v6w1z0ezon5dg61ac 192.168.100.90:2377
```

##### add other VM as worker nodes

use the output from docker swarm join to join as work nodes on other VMs. 

```
docker swarm join --token SWMTKN-1-5f4g4cbo8fdldw98a3hmhf8s12qe6ffx6s2sbluyyihpp5m3bg-eaia06w7v6w1z0ezon5dg61ac 192.168.100.90:2377
```

##### Firewall problem if cannot join swarm node

1) Run the code below if you cannot join the swarm in VM/ other computers

```
sudo apt install firewalld
sudo firewall-cmd --add-port=2376/tcp --permanent  
sudo firewall-cmd --add-port=2377/tcp --permanent  
sudo firewall-cmd --add-port=7946/tcp --permanent  
sudo firewall-cmd --add-port=7946/udp --permanent  
sudo firewall-cmd --add-port=4789/udp --permanent
sudo systemctl restart firewalld
sudo firewall-cmd --zone=public --add-port=2377/tcp --permanent
sudo firewall-cmd --reload
sudo ufw enable
sudo ufw allow 2376
sudo ufw allow 2377
sudo ufw allow 7946
sudo ufw allow 4789

systemctl reboot -i
```

Notes: the last command will restart your computer, make sure you save all of your works before run it.

#### Prepare the Environment

Execute the command below in ALL VMs.
Pull the docker image
```
docker pull newnius/hadoop:2.7.4
```

create dir `/data`
```
sudo mkdir -p /data
sudo chmod 777 /data

```

create dir for data persist inside `/data`
```
mkdir -p /data/hadoop/hdfs/
mkdir -p /data/hadoop/logs/
```

#### Configure Hadoop

Create a directory named `"config"` and copy paste five configuration files beneath it. 

```
config/
|-------core-site.xml
|-------hdsf-site.xml
|-------mapred-site.xml
|-------slaves
|-------yarn-site.xml
```

Copy and Paste the contents below for these 5 configuration files.

`core-site.xml`

```
<?xml version="1.0" encoding="UTF-8"?> 

<?xml-stylesheet type="text/xsl" href="configuration.xsl"?> 

<!-- Put site-specific property overrides in this file. --> 

<configuration> 

    <property>        

        <name>fs.defaultFS</name> 

        <value>hdfs://hadoop-node021:8020</value> 

    </property> 

    <property> 

        <name>fs.default.name</name> 

        <value>hdfs://hadoop-node021:8020</value> 

    </property> 

</configuration> 
```

`hdsf-site.xml`

```
<?xml version="1.0" encoding="UTF-8"?> 

<?xml-stylesheet type="text/xsl" href="configuration.xsl"?> 

<!-- Put site-specific property overrides in this file. --> 

<configuration> 

    <property> 

        <name>dfs.permissions</name> 

        <value>false</value> 

    </property> 

    <property> 

        <name>dfs.namenode.http-address</name> 

        <value>hadoop-node021:50070</value> 

    </property> 

    <property> 

        <name>dfs.namenode.secondary.http-address</name> 

        <value>hadoop-node022:50090</value> 

    </property>  

    <property>    

        <name>dfs.datanode.max.transfer.threads</name>    

        <value>8192</value>     

    </property> 

    <property> 

        <name>dfs.replication</name> 

        <value>3</value> 

    </property> 

</configuration> 
```

`mapred-site.xml`

```
<?xml version="1.0"?> 

<?xml-stylesheet type="text/xsl" href="configuration.xsl"?> 

<!-- Put site-specific property overrides in this file. --> 

<configuration> 

    <property>                       

        <name>mapreduce.framework.name</name> 

        <value>yarn</value> 

    </property> 

    <property> 

        <name>mapreduce.jobhistory.address</name> 

        <value>hadoop-node021:10020</value> 

    </property> 

    <property> 

        <name>mapreduce.jobhistory.webapp.address</name> 

        <value>hadoop-node021:19888</value> 

    </property>  

</configuration> 
```

`yarn-site.xml`

```
<?xml version="1.0"?> 

<!-- Site specific YARN configuration properties --> 

<configuration> 

    <property> 

        <name>yarn.application.classpath</name> 

        <value>/usr/local/hadoop/etc/hadoop, /usr/local/hadoop/share/hadoop/common/*, /usr/local/hadoop/share/hadoop/common/lib/*, /usr/local/hadoop/share/hadoop/hdfs/*, /usr/local/hadoop/share/hadoop/hdfs/lib/*, /usr/local/hadoop/share/hadoop/mapreduce/*, /usr/local/hadoop/share/hadoop/mapreduce/lib/*, /usr/local/hadoop/share/hadoop/yarn/*, /usr/local/hadoop/share/hadoop/yarn/lib/*</value> 

    </property> 

    <property> 

        <name>yarn.resourcemanager.hostname</name> 

        <value>hadoop-node021</value> 

    </property> 

    <property> 

        <name>yarn.nodemanager.aux-services</name> 

        <value>mapreduce_shuffle</value> 

    </property> 

    <property> 

        <name>yarn.log-aggregation-enable</name> 

        <value>true</value> 

    </property> 

    <property> 

        <name>yarn.log-aggregation.retain-seconds</name> 

        <value>604800</value> 

    </property> 

    <property> 

        <name>yarn.nodemanager.resource.memory-mb</name> 

        <value>2048</value> 

    </property> 

    <property> 

        <name>yarn.nodemanager.resource.cpu-vcores</name> 

        <value>2</value> 

    </property> 

    <property> 

        <name>yarn.scheduler.minimum-allocation-mb</name> 

        <value>1024</value> 

    </property> 

</configuration> 
```

`slaves`

```
hadoop-node022 

hadoop-node023 
```

Note: Using `nano` command in the terminal to insert all these codes, subject to change for the number of hadoop-node based on the "alias" set in the environment.

#### Distribute all these files across the nodes by manually repeating the step above

If you have 3 nodes, repeat the step above 3 times.

#### Bring up the nodes of Hadoop

Execute the command below at `master/manager node` for this case is `tehtehv2-virtual-machine` to bring up all the nodes.


Repeat this process `N` times as you have `N` nodes (3 times in our case) 

```
docker service create \
        --name {{ Alias }} \
        --hostname {{ Alias }} \
        --constraint node.hostname=={{ Hostname }} \
        --network hadoop-net \
        --endpoint-mode dnsrr \
        --mount type=bind,src={{HOME}}/data/hadoop/config,dst=/config/hadoop \
        --mount type=bind,src={{HOME}}/data/hadoop/hdfs,dst=/tmp/hadoop-root \
        --mount type=bind,src={{HOME}}/data/hadoop/logs,dst=/usr/local/hadoop/logs \
        newnius/hadoop:2.7.4
```
Note: Change the `{{Alias}}`, `{{Hostname}}` and `{{HOME}}` according to your node.

Example:

```
docker service create \ 
        --name hadoop-node021 \ 
        --hostname hadoop-node021 \ 
        --constraint node.hostname==tehtehv2-virtual-machine \ 
        --network hadoop-net \ 
        --endpoint-mode dnsrr \ 
        --mount type=bind,src=/home/tehteh-v2/data/hadoop/config,dst=/config/hadoop \ 
        --mount type=bind,src=/home/tehteh-v2/data/hadoop/hdfs,dst=/tmp/hadoop-root \ 
        --mount type=bind,src=/home/tehteh-v2/data/hadoop/logs,dst=/usr/local/hadoop/logs \ 
        newnius/hadoop:2.7.4 
```

```
docker service create \ 
        --name hadoop-node022 \ 
        --hostname hadoop-node022 \ 
        --constraint node.hostname==tehteh22v3-virtual-machine \ 
        --network hadoop-net \ 
        --endpoint-mode dnsrr \ 
        --mount type=bind,src=/home/tehteh22v3/data/hadoop/config,dst=/config/hadoop \ 
        --mount type=bind,src=/home/tehteh22v3/data/hadoop/hdfs,dst=/tmp/hadoop-root \ 
        --mount type=bind,src=/home/tehteh22v3/data/hadoop/logs,dst=/usr/local/hadoop/logs \ 
        newnius/hadoop:2.7.4 
```

```
docker service create \ 
        --name hadoop-node023 \ 
        --hostname hadoop-node023 \ 
        --constraint node.hostname==tehteh22v4-virtual-machine \ 
        --network hadoop-net \ 
        --endpoint-mode dnsrr \ 
        --mount type=bind,src=/home/tehteh22v4/data/hadoop/config,dst=/config/hadoop \ 
        --mount type=bind,src=/home/tehteh22v4/data/hadoop/hdfs,dst=/tmp/hadoop-root \ 
        --mount type=bind,src=/home/tehteh22v4/data/hadoop/logs,dst=/usr/local/hadoop/logs \ 
        newnius/hadoop:2.7.4 
```

#### Start Hadoop Services

Execute the command below on `tehtehv2-virtual-machine` to enter `master node`

```
docker exec -it hadoop-node021.1.$(docker service ps \
hadoop-node021 --no-trunc | grep Running | awk '{print $1}' ) bash
```

Now we will execute the following commands inside the master node

#### Format master node for first time running

```
# stop all Hadoop processes
sbin/stop-yarn.sh
sbin/stop-dfs.sh

# format namenode
bin/hadoop namenode -format

# start yarn and dfs nodes
sbin/start-dfs.sh
sbin/start-yarn.sh
```

#### Run MapReduce Program for π Calculation

```
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.4.jar pi 100 150
```

You should see the result after executing the command above.

Congratulations, now you have successfully run Hadoop on Multi Computer Multi Nodes.

### Shut Down the Swarm after you finish running the code
After you have run all the codes successfully, make sure you shut down your swarm before creating a new swarm.

Execute the command below on all the nodes that have join swarm. 
```
docker swarm leave (for worker node)
docker swarm leave --force  (for manager node)
```


### Result Comparison

You can scale the number of nodes and VMs to make the comparison. You may also change the parameter of `Number of Maps` and `Samples per Map` with different value to see the difference.

Example of Comparison:

```
yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar pi 100 150
Number of Maps  = 100
Samples per Map = 150

single computer with multi nodes
Job Finished in 123.303 seconds
Estimated value of Pi is 3.14160000000000000000

multi computer with multi nodes
Job Finished in 97.229 seconds
Estimated value of Pi is 3.14160000000000000000
```

## Reference

[Docker Swarm Official Docs](https://docs.docker.com/engine/swarm/)

[Docker image used for Hadoop](https://hub.docker.com/r/newnius/hadoop)

[GitHub Repo for Docker Image for Hadoop](https://github.com/newnius/Dockerfiles/tree/master/hadoop/2.7.4)

[How to quickly setup a Hadoop cluster in Docker](https://blog.newnius.com/how-to-quickly-setup-a-hadoop-cluster-in-docker.html#Start-Hadoop-Cluster)

[Setup a distributed Hadoop/HDFS cluster with docker](https://blog.newnius.com/setup-distributed-hadoop-cluster-with-docker-step-by-step.html)
