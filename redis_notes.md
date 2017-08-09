# Redis Notes

## Table of Contents

<!-- TOC START min:2 max:4 link:true update:true -->
  - [Table of Contents](#table-of-contents)
  - [What is Redis?](#what-is-redis)
  - [Very fast (that's the reputation)](#very-fast-thats-the-reputation)
  - [As a cache](#as-a-cache)
  - [As a Database](#as-a-database)
  - ["Traditional" Use Cases](#traditional-use-cases)
  - [More "Interesting" Use Cases](#more-interesting-use-cases)
  - [Data structures](#data-structures)
      - [Lists](#lists)
      - [Sets](#sets)
      - [Sorted Set](#sorted-set)
      - [Hash](#hash)
      - [Bitmap](#bitmap)
      - [Hyperloglog](#hyperloglog)
      - [Geospatial](#geospatial)
  - [Scaling](#scaling)
      - [Redis Cluster](#redis-cluster)
      - [Redis Sentinel](#redis-sentinel)
      - [RLEP (Redi Labs Enterprise Pack)](#rlep-redi-labs-enterprise-pack)
  - [Tips for operations](#tips-for-operations)
    - [What happens during failover](#what-happens-during-failover)
  - [Further Reading](#further-reading)
    - [Self learning](#self-learning)
      - [Links on Clustering/Sentinel (Redis in Production)](#links-on-clusteringsentinel-redis-in-production)

<!-- TOC END -->

## What is Redis?
 - Stands for: *REmote Dictionary Server*
 - Open source (BSD licensed), super fast, noSQL *in-memory cache/database/message broker*
 - *Key -> data structure store* (vs. key-value store, more than String)
 - In-memory, yes, but has *disk persistence*, so you survive a crash
 - Clustering, HA supported
 - Very fast paced open source development since 2009, huge active community
 - Backed by commercial company *Redis Labs*
 - Very efficient but *very simple*, low level tool (vs. SQL db indexes, query language, high level clustering, etc.)
 - Supports *pipelines* and *basic transactions*


## Very fast (that's the reputation)
 - Written in *C*
 - *Singe threaded*, so no locking needed
 - Runs on *ARM processors*
 - Base binary is <4MB, very small memory/CPU foorptint, can be *embedded in IoT* devices (think Raspberry Pi)


## As a cache
 - Configurable eviction, key expiration
 - Optional data persistance (unusual for a cache system) so this cache can recover from a melt-down (does not need to start "cold")
 - "Intelligent" cache, becasue the data can be analysed/quaried with a lots of commands
 - Binary safe (no encoding)
 - String alue is 512MB!
 - Ideal for:
   - Plain strings
   - Full JSON objects
   - Binary file content (e.g. for in-memory image manipulation, see BITOP)
   - Raw bits/flags/counters (e.g. realtime metrics), see SETBIT, INCR, INCRBY

## As a Database
 - In addition to String it also has: list, set, sorted set, hash (+bitmap +hyperloglog)
 - 180 high perdformant commands (server side execution)
 - PubSub (browsers subscribe to events with node.js)
 - LUA scripting and modules add flexibility (LUA compiles into memory)
 - Can be used as *1st class database*


## "Traditional" Use Cases
 - Counting stuff
 - User session store
 - Recent visitor list
 - Leader board, user voting (show top users/players based on score crieria)
 - Expired items in list (like user session expiration)
 - Unique items in a given amount of time
 - Real time basic analysis for stats, like inbound traffic tracking from IPs for DDOS detection)
 - PubSub
 - Simple Queues
 - Geolocation database
 - Distributed locks (e.g. Redlock)
 - Autocomplete word database


## More "Interesting" Use Cases
  - A buffer or pre-processor *in front of Mongo, MySQL* for write-intensive operations (like incoming Big Data)
  - A real-time data update digesting and reporting for short lived live updates
  - A IoT sensor data fed into Redis for live reports, archived later


## Data structures

#### Lists
 - Linked lists
 - Max 4 billion in size
 - Ideal for queues, stacks, top N, etc.

#### Sets
 - Unique values
 - Union, Diff between multiple Sets, see SINTER
 - Extract random members, see SPOP

#### Sorted Set
 - Adds a score to the set value
 - Can be used as indexes of other data

#### Hash
 - name value pairs inside of a single key
 - used for maps (like tables)

#### Bitmap
 - Operations on binary data in memory, see BITOP

#### Hyperloglog
 - Probabilistic cardinality estimator
 - Counts unique things statically

#### Geospatial
 - Members of a set representing lat/lon
 - Funcctions that return distance betwwen 2 memmbers and a list members in a given radius

## Scaling
 - Replication, master-slave is supported (no down time when master goes down)
 - In a common scenario Master can interact with clents, while slave stores replicated data to disk
 - Docker support in v4.0

#### Redis Cluster
  - data sharding and replication strategy with re-sharding between nodes while the nodes are running, with failover support
   - Cluster needs "smart" (cluster aware) Redis clients

#### Redis Sentinel
  - node monitoring, notification and failover management process, independent from Redis Cluster
   - Sentinel need "smart" (sentinel aware) Redis clients, OR having HAProxy in front of Redis to redirect to slave in case of master failure, OR use virtual IPs

#### RLEP (Redi Labs Enterprise Pack)
 - Offers a proxy based independent clustering solution for $$


## Tips for operations
* `redis-cli monitor` - this command, instead of going into the interactive mode, it will output commands received by that redis server, live, kinda like tail. Good for basic (low traffic) monitoring.
* `INFO replication` - This Redis command tells you if the server you are connected to is a Master or slave and other replication details.
* `SAVE` and `BGSAVE` - save the `dump.rdb` (or whatever file name is configured in redis.conf) to disk.
* Simple data recovery: `dump.rdb` can be copied into the filesystem of a stopped Redis instance, and when Redis starts, it will restore its state from it.
* **NEVER** use even number of sentines. They might not be able to pick a new master when half of the sentinels vote for one and the other half for another node. **ALWAYS** use odd number of sentines, at least 3.

### What happens during failover

Source: https://www.youtube.com/watch?v=wdPqFa3ru6U&list=PL83Wfqi-zYZF1MDKLr5djmLYUI0woy1wi&index=48 and in writing http://lolpack.me/rediswhitepaper.pdf

Steps of failover -- 1GB in memory

1. Master is unreachable
1. Sentinels (as they discover that they haven't heard from Master for a while) will reach quorum and initiate failover -- 30 sec, configurable
1. A Slave is elected as the new Master
1. Master starts serving requests, but the Slave(s) not yet!
1. The new Master does a full `BGSAVE` (dumps last known state to disk) -- ~9 sec
1. The new Master syncs data to discoverable Slaves. During this time the Slaves are **NOT** serving traffic -- ~40 sec, over very fast Internet connection
1. Slaves load data from disk into memory -- ~8 sec
1. Slaves start serving traffic

Downtime for Slaves:
* For 1GB of data: ~1.5 min.
* For 5GB of data: ~3 min.
* For 20GB of data: ~12.5 min!
* For 40GB of data: ~18 min!!!

That's when people start asking "Why can't you just restart it??"... Well, restarting would do nothing, the loading from disk would happen there too...

**Conclusion:** Do NOT interact with Slaves if you care about failover related down times. Interact with Masters exclusively, for both READs and WRITEs. The Slaves will sync up in the background during failover.

## Further Reading

### Self learning

 - https://redis.io/documentation
 - https://www.youtube.com/watch?v=Hbt56gFj998
 - https://www.youtube.com/watch?v=jTTlBc2-T9Q
 - https://gemalto.udemy.com/learn-redis/learn/v4/overview
 - https://redislabs.com/resources/ebook/
 - http://openmymind.net/redis.pdf
 - https://try.redis.io/
 - https://www.youtube.com/watch?v=qHkXVY2LpwU - Redis complementing MongoDb

#### Links on Clustering/Sentinel (Redis in Production)
 - https://redis.io/topics/sentinel
 - https://redis.io/topics/cluster-tutorial
 - https://scalegrid.io/blog/high-availability-with-redis-sentinels-connecting-to-redis-masterslave-sets/
 - http://code.flickr.net/2014/07/31/redis-sentinel-at-flickr/
 - http://www.programcreek.com/java-api-examples/index.php?source_dir=wint-master/wint-framework/src/main/java/wint/help/redis/SentinelRedisClient.java
 - Upgrading or restaring Redis without downtime: https://redis.io/topics/admin
 - Clustering alternative: https://github.com/twitter/twemproxy - Proxy in front of Redis nodes, client sees a single Redis instance. Supports fail-over and limited sharding. See https://www.youtube.com/watch?v=3zxYaI3RQyM
 - Redis Sentinel failover case study: https://www.youtube.com/watch?v=wdPqFa3ru6U&list=PL83Wfqi-zYZF1MDKLr5djmLYUI0woy1wi&index=48 . Very useful!
