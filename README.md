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

## Result Comparison

yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar pi 100 150
Number of Maps  = 100
Samples per Map = 150

With only one datanode
Job Finished in 123.303 seconds
Estimated value of Pi is 3.14160000000000000000

with three datanode
Job Finished in 97.229 seconds
Estimated value of Pi is 3.14160000000000000000

## Firewall

sudo firewall-cmd --add-port=2377/tcp                       ─╯
sudo firewall-cmd --runtime-to-permanent
sudo firewall-cmd --reload
systemctl reboot -i

## Docker Swarm

1) Check your <manager-node-ip>
using hostname -I in linux
or ipconfig in windows

docker swarm init --advertise-addr 192.168.100.19

2)To add a worker to this swarm, run the following command:
    docker swarm join --token SWMTKN-1-0kv89vf711srufqxdklvp2rw38ieh3gcq18f7kmj00rns654ha-5e342uam12f7i71aod3egy1ak 192.168.100.19:2377

3)Deploy your services to the swarm
docker stack deploy -c docker-compose.yaml hadoop-stack
