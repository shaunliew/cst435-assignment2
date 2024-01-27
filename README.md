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

### Case 1: Single Computer Multi Nodes

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

### Case 2: Multi Computer Multi Nodes

### Create VM using Multipass(optional if you are using multipass)

#### Multipass to create VM and get into VM terminal

```
multipass launch --name vm-name
multipass shell vm-name
```

#### Install Docker into VM

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

#### Delete VM

```
multipass delete vm-name
multipass purge
```

### Docker Swarm Setup for multi-computer communication

1) Check your <manager-node-ip> for docker swarm initialization

    using `hostname -I`  in linux

    or `ipconfig`  in windows

    Example: `192.168.100.19`

2) Initialize the docker swarm

`docker swarm init --advertise-addr <host-ip-address>`


the output:

```                                                  ─╯
Swarm initialized: current node (klowy9df6t62jbikdpw3szb2h) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token <token-generated> <host-ip-address>:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

2) Use the output given from the swarm initialization to add the worker-node from VM/another computer. Run the command above at different machines. 

```
docker swarm join --token <token-generated> <host-ip-address>:2377
```


4) Deploy your services to the swarm at your manager node(main computer). Make sure that your terminal is at `cst435-assignment2` directory now.

```
docker stack deploy -c docker-compose.yaml hadoop-stack
```

5) Get the `namenode` container name by using the comman below.

```
docker ps
```

Note:
hadoop-stack is the stack name

6) Exec into `namenode` container

```
docker exec -it <namenode-container-name> /bin/bash
```


7) run the mapreduce code inside your `namenode` container
```
yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar pi 10 15
```

### Scale the datanode
What you have done so far is one computer one node. Now you can scale the datanode for multiprocessing. 

1) Exit from the `namenode` terminal.

```
exit
```

2) Scale the number of datanode to 3(any number you prefer) make sure that you put the datanode-service name.

Get the `datanode-service-name`
```
docker stack services <stack-name>
```

Scale the `datanode` to 3
```
docker service scale <datanode-service-name>=3
```

example of  <datanode-service-name>: `hadoop-stack_datanode`
example of  <stack-name>: `hadoop-stack`

3) Repeat the same code for exec into `namenode` and run the `mapreduce` program. You should see the difference in the time taken.

```
docker exec -it <namenode-container-name> /bin/bash
yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar pi 10 15
```

note: You can change the value of 10 and 15 accordingly.

### Shut Down the Swarm
After you have run all the codes successfully, make sure you shut down your swarm before creating a new swarm.

```
docker swarm leave (for worker node)
docker swarm leave --force  (for manager node)
```

### Issue Faced

The code above have one problem which is it cannot connect to multicomputer as the network is unreachable.
```
docker swarm Error response from daemon: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing dial tcp 192.168.65.3:2377: connect: network is unreachable
```

one possible reason is `firewall`, can refer the section below to solve this firewall issue.

#### Firewall problem if cannot join swarm node

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

Still pending to solve this problem.
### Result Comparison in one single computer but different number of datanode

You may do the result comparison with the setup below.

1) Single Computer, Single Node

Run the mapreduce program using single computer single datenode

2) Single Computer, Multi Node

Run the mapreduce program using single computer and scale the datanodes

3) Multi Computer, Single Node (Pending to be solved)

Run the mapreduce program using multi computer single datenode

4) Multi Computer, Multi Node (Pending to be solved)

Run the mapreduce program using multi computer and scale the datanodes


Example of Comparison:

```
yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar pi 100 150
Number of Maps  = 100
Samples per Map = 150

single computer with only one datanode
Job Finished in 123.303 seconds
Estimated value of Pi is 3.14160000000000000000

single computer with three datanodes
Job Finished in 97.229 seconds
Estimated value of Pi is 3.14160000000000000000
```

## Reference

[Docker Swarm Official Docs](https://docs.docker.com/engine/swarm/)

[Docker image used for Hadoop](https://hub.docker.com/r/newnius/hadoop)

[GitHub Repo for Docker Image for Hadoop](https://github.com/newnius/Dockerfiles/tree/master/hadoop/2.7.4)

[How to quickly setup a Hadoop cluster in Docker](https://blog.newnius.com/how-to-quickly-setup-a-hadoop-cluster-in-docker.html#Start-Hadoop-Cluster)

[Setup a distributed Hadoop/HDFS cluster with docker](https://blog.newnius.com/setup-distributed-hadoop-cluster-with-docker-step-by-step.html)
