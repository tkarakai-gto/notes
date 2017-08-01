# What is Redis?
 - Open source, super fast, noSQL in-memory cache/database
 - Key -> data structure store (vs. key-value store, more than String)
 - In-memory, yes, but has disk persistance, so you survive a crash
 - Clustering, HA supported
 - Very fast pase open source development since 2009, huge active community
 - Backed by comercial company Redis Labs
 - Very efficient but very simple, low level tool (vs. SQL db indexes, query language, high level clustering, etc.)
 - Supports pipelines and basic transactions

## Very fast (that's the reputation)
 - Written in C
 - Singe threaded, so no locking needed
 - Runs on ARM processors
 - Base binary is <4MB, very small memory/CPU foorptint, can be embedded in IoT devices (think Raspberry Pi)

## As a cache:
 - On top of Mongo, MySQL, etc.: your app or web site that needs faster then those db can response
 - Binary safe (no encoding)
 - String alue is 512MB!
 - Ideal for:
   - Plain strings
   - Full JSON objects
   - Binary file content (e.g. for in-memory image manipulation, see BITOP)
   - Raw bits/flags/counters (e.g. realtime metrics), see SETBIT, INCR, INCRBY

## As a database:
 - In addition to String it also has: list, set, sorted set, hash (+bitmap +hyperloglog)
 - 160 high perdformant commands (server side execution)
 - PubSub (browsers subscribe to events with node.js)
 - LUA scripting and modules add flexibility (LUA compiles into memory)
 - Can be used as 1st class database

## Some use cases:
 - User session store
 - Recent visitor list
 - Leader board, user voting (show top users/players based on score crieria)
 - Expired items in list
 - Counting stuff
 - Unique items in a given amount of time
 - Real time basic analysis for stats, like inbound traffic tracking from IPs for DDOS detection)
 - PubSub
 - Simple Queues
 - Geolocation database
 - Distributed locks (e.g. Redlock)
 - Autocomplete word database

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

#### Links on Clustering/Sentinel
 - https://redis.io/topics/sentinel
 - https://redis.io/topics/cluster-tutorial
 - https://scalegrid.io/blog/high-availability-with-redis-sentinels-connecting-to-redis-masterslave-sets/
 - http://code.flickr.net/2014/07/31/redis-sentinel-at-flickr/
 - http://www.programcreek.com/java-api-examples/index.php?source_dir=wint-master/wint-framework/src/main/java/wint/help/redis/SentinelRedisClient.java
