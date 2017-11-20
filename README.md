# mongo-docker-compose
This repository provides a fully sharded mongo environment using docker-compose and local storage.

The MongoDB environment consists of the following docker containers

 - **mongosrs(1-3)n(1-3)**: Mongod data server with three replica sets containing 3 nodes each (9 containers)
 - **mongocfg(1-3)**: Stores metadata for sharded data distribution (3 containers)
 - **mongos(1-2)**: Mongo routing service to connect to the cluster through (1 container)

## Caveats

 - This is designed to have a minimal disk footprint at the cost of durability.
 - This is designed in no way for production but as a cheap learning and exploration vehicle.

## Installation (Debian base):

### Install Docker

    sudo apt-get install -y apparmor lxc cgroup-lite curl
    wget -qO- https://get.docker.com/ | sh
    sudo usermod -aG docker YourUserNameHere
    sudo service docker restart

### Install Docker-compose (1.4.2+)

    sudo su
    curl -L https://github.com/docker/compose/releases/download/1.4.2/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
    chmod +x /usr/local/bin/docker-compose
    exit

### Check out the repository

    git clone git@github.com:singram/mongo-docker-compose.git
    cd mongo-docker-compose


### Setup Cluster
This will pull all the images from [Docker index](https://index.docker.io/u/jacksoncage/mongo/) and run all the containers.

    docker-compose up

Please note that you will need docker-compose 1.4.2 or better for this to work due to circular references between cluster members.
You will need to run the following *once* only to initialize all replica sets and shard data across them

    ./initiate

You should now be able connect to mongos1 and the new sharded cluster from the mongos container itself using the mongo shell to connect to the running mongos process

    docker exec -it mongos1 mongo --port 21017

### Persistent storage
Data is stored at `./data/` and is excluded from version control. Data will be persistent between container runs. To remove all data `./reset`

## Convert configservers to replicaset without explicit downtime
Official documentation: https://docs.mongodb.com/v3.2/tutorial/upgrade-config-servers-to-replica-set/

### Turn off the balancer!
Connect to any mongos instance and execute:

        git clone git@github.com:melifaro/mongo-docker-compose.git
        mongos> sh.getBalancerState()
        true
        mongos> sh.setBalancerState(false)
        WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })

1. On the first config server (**mongocfg1**) connect to the mongo shell and execute:

        rs.initiate({
        _id: "cfg-rs",
        version: 1,
        members: [ { _id: 0, host : "mongocfg1:27017" } ]})

2. Shutdown first config server using ` docker-compose stop mongocfg1 `
3. Comment the first "command" line in docker-compose file and uncomment the second. This will start **mongocfg1** as a single member of replica set with legacy config server mode Sync Cluster Connection Config.
4. Stop **mongocfg2** and **mongocfg3** (`docker-compose stop mongocfg2 mongocfg3`) to ensure that cluster is in read-only state.
5. Uncomment the section about **mongocfg2-rs** and **mongocfg3-rs** in docker-compose file, start them with `docker-compose start mongocfg2-rs mongocfg3-rs`
6. Add them to replicaset (on **mongocfg1**):

        cfg-rs:PRIMARY> rs.add( { host: "mongocfg2-rs:27017", priority: 0, votes: 0 } )
        cfg-rs:PRIMARY> rs.add( { host: "mongocfg3-rs:27017", priority: 0, votes: 0 } )

7. Give the new members votes and priority:

        var cfg = rs.conf();
        cfg.members[0].priority = 1;
        cfg.members[1].priority = 1;
        cfg.members[0].votes = 1;
        cfg.members[1].votes = 1;
        rs.reconfig(cfg);
8. Step down on primary and wait for one of the new servers get promoted. Then shutdown **mongocfg1**:

        cfg-rs:PRIMARY> rs.stepDown()
        docker-compose stop mongocfg1

9. Update the configuration of mongos to include replicaset name and 2 new running servers by uncommenting the second line in their config and restart mongos:

        command: mongos --configdb cfg-rs/mongocfg2-rs:27017,mongocfg3-rs:27017 --port 27017
        docker-compose stop mongos1 mongos2
        docker-compose up -d mongos1 mongos2
Ensure that mongos are successfully connected to two new config servers.
10. Start the first config server **mongocfg1** without --configsvrMode=sccc option (comment the second config line for this and uncomment the third one). Wait till it finish the sync.

        command: mongod --noprealloc --replSet cfg-rs --smallfiles --dbpath /data/db --configsvr --noauth --port 27017
        docker-compose up -d mongocfg1

Make sure that mongos are aware of the change. **That doesn't requre restart!**
11. At this point you can promote **mongofcg1** to primary by setting the other secondaty priority to 0. Then shutdown **mongocfg2-rs** and edit the replicaset configuration:

        cfg-rs:PRIMARY> cfg = rs.conf()
        cfg-rs:PRIMARY> cfg.members[1].host = "mongocfg2:27017"
        cfg-rs:PRIMARY> rs.reconfig(cfg)

Start **mongocfg2** with datavolume from **mongocfg2-rs**:

        mongocfg2:
            container_name: mongocfg2
            image: mongo:3.2.16
            command: mongod --noprealloc --smallfiles --replSet cfg-rs --dbpath /data/db --configsvr --noauth --port 27017
            environment:
            TERM: xterm
            expose:
                 - "27017"
        volumes:
          - /etc/localtime:/etc/localtime:ro
          - ${HOME}/mongocfg2-rs:/data/db

Ensure that it's successfully connected to the cluster and mongos are aware of the change.
12. Repeat the previous two steps for **mongocfg3**. After this we'll have config servers replicaset running at exact the same nodes with the same ports as before the migration.
In our case that'll mean that mongos will be restarted one more time after appling the puppet changes.

## TODO

 - Add local Ops Mananger
 - Implement authentication across cluster.  http://docs.mongodb.org/manual/tutorial/deploy-replica-set-with-auth/

## Built upon
 - [Docker-compose wait to start](http://brunorocha.org/python/dealing-with-linked-containers-dependency-in-docker-compose.html)
 - [Bi directional docker commuication](http://abdelrahmanhosny.com/2015/07/01/3-solutions-to-bi-directional-linking-problem-in-docker-compose/)
 - [MongoDB Sharded Cluster by Sebastian Voss](https://github.com/sebastianvoss/docker)
 - [MongoDB](http://www.mongodb.org/)
 - [Mongo Docker ](https://github.com/jacksoncage/mongo-docker)
 - [DnsDock](https://github.com/tonistiigi/dnsdock)
 - [Docker](https://github.com/dotcloud/docker/)
