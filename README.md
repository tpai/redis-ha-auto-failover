# Redis High Availability (HA) Auto-Failover Verification

This project is to verify the auto-failover functionality of a Redis High Availability (HA).

## Project Structure

The project consists of several Docker containers:

- redis-master: Main Redis server.
- redis-slave1 and redis-slave2: Secondary Redis servers.
- redis-sentinel1, redis-sentinel2, and redis-sentinel3: Monitors for Redis servers.
- sentinel1-proxy, sentinel2-proxy, and sentinel3-proxy: Proxies for TCP requests to Redis master.
- nginx: Load balancer for user connections to proxies.

## Running the Project

To run the project, use Docker Compose:

```
docker-compose up -d
```

This will start all the containers in the correct order, with the Nginx server depending on the Sentinel proxies, which in turn depend on the Redis servers and Sentinels.

## Testing Auto-Failover

First, connect to the redis-master container and check the replication state.

```
docker run --net=host --rm -it redis:7 bash
$ redis-cli -p 36379
> info replication
# Replication
role:master
connected_slaves:2
```

To test the auto-failover, you can stop the redis-master and redis-sentinel1 container:

```
docker-compose stop redis-master redis-sentinel1
```

The Sentinels will detect the failure and promote one of the slaves to be the new master. The proxies will update their configuration to point to the new master, and the Nginx server will continue to balance load between them. 

```
+sdown master mymaster 0.0.0.0:6379
+vote-for-leader
+elected-leader master mymaster 0.0.0.0 6379
+selected-slave slave 127.0.0.1:6380 127.0.0.1 6380
+promoted-slave slave 127.0.0.1:6380 127.0.0.1 6380
+odown master mymaster 0.0.0.0:6379 #quorum 2/2
+switch-master mymaster 0.0.0.0:6379 127.0.0.1:6380
```

Run the info command again. You will see the number of connected slaves has decreased, but you are still connected to the Redis master.

```
> info replication
# Replication
role:master
connected_slaves:1
```

## References

- https://hub.docker.com/r/flant/redis-sentinel-proxy
- https://github.com/880831ian/docker-compose-redis-sentinel
