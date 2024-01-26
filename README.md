# CST435 Assignment 2

## docker-compose

docker-compose up -d
docker exec -it cst435-assignment2-namenode-1 /bin/bash
yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar pi 10 15

## docker stack
Note:
hadoop-stack is the stack name

docker stack deploy -c docker-compose.yaml hadoop-stack
docker exec -it hadoop-stack_namenode.1.yuim9xv7zihwvagu5o03x7ccv /bin/bash
yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar pi 10 15

docker service scale testing-hadoop_datanode=3

docker stats testing-hadoop_resourcemanager.1.tnaqr8sqecjx136p0rhbpi32f

## Result Comparison in one single computer but different number of datanode

yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar pi 100 150
Number of Maps  = 100
Samples per Map = 150

With only one datanode
Job Finished in 123.303 seconds
Estimated value of Pi is 3.14160000000000000000

with three datanode
Job Finished in 97.229 seconds
Estimated value of Pi is 3.14160000000000000000

## Multipass to create VM

multipass launch --name vm-name
multipass shell vm-name

### Delete VM

multipass delete vm-name
multipass purge

## Install Docker into VM

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


## Docker Command

list out the container
`docker ps`
list out the connected node
`docker node ls`
check the status of your services
`docker stack services hadoop-stack`
delete the unused container
`docker container prune`
stop the container
`docker stop <container-id>`
delete the unused volume
`docker volume prune`
remove the whole stack
`docker stack rm hadoop-stack`

## Docker Swarm for multi computer communication

1) Check your <manager-node-ip> and initialize the docker swarm
using `hostname -I in linux`
or `ipconfig in windows`

`sudo docker swarm init --advertise-addr 192.168.100.19`

2) To add a worker to this swarm, run the following command:
    `docker swarm join --token <token-generated>`

3) go to your other computer/ VM terminal and run the code above to join the node.

4) Deploy your services to the swarm at your manager node(main computer)
`docker stack deploy -c docker-compose.yaml hadoop-stack`

5) go to your docktor desktop and find the name of the main namenode and exec it
`docker exec -it <docker-stack-namenode-name> /bin/bash`

6) run the mapreduce code

`yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar pi 10 15`

7) Compare the result before and after using multinode

## Firewall problem if cannot join swarm node

1) Run the code below if you cannot join the swarm in VM/ other computers
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
