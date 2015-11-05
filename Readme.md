echo "1. Step 1: Set up a key-value store"

docker-machine create -d virtualbox mh-keystore

docker $(docker-machine config mh-keystore) run -d \
    -p "8500:8500" \
    -h "consul" \
    progrium/consul -server -bootstrap

eval "$(docker-machine env mh-keystore)"
docker ps


echo "2. Create a Swarm Cluster"
docker-machine create \
-d virtualbox \
--swarm --swarm-image="swarm" --swarm-master \
--swarm-discovery="consul://$(docker-machine ip mh-keystore):8500" \
--engine-opt="cluster-store=consul://$(docker-machine ip mh-keystore):8500" \
--engine-opt="cluster-advertise=eth1:2376" \
mhs-demo0

docker-machine create -d virtualbox \
    --swarm --swarm-image="swarm:1.0.0-rc2" \
    --swarm-discovery="consul://$(docker-machine ip mh-keystore):8500" \
    --engine-opt="cluster-store=consul://$(docker-machine ip mh-keystore):8500" \
    --engine-opt="cluster-advertise=eth1:2376" \
mhs-demo1


echo "3. Create the overlay Network""
eval $(docker-machine env --swarm mhs-demo0) && docker info

docker network create --driver overlay my-net

eval $(docker-machine env mhs-demo0) && docker network ls

eval $(docker-machine env mhs-demo1) && docker network ls


echo "4. Run an application on your Network"
eval $(docker-machine env mhs-demo0)

docker run -itd --name=web --net=my-net --env="constraint:node==mhs-demo0" nginx

eval $(docker-machine env mhs-demo1)

docker run -it --rm --net=my-net --env="constraint:node==mhs-demo1" busybox wget -O- http://web


echo "5. Check external connectivity"
eval $(docker-machine env mhs-demo1) && docker network ls

eval $(docker-machine env mhs-demo0) && docker network ls

docker exec web ip addr


echo "6. Extra Credit with Docker Compose"
docker-compose up --x-networking up -d
