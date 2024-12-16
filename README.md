# mongod-sharding

### <a name="content">Table of contents:</a>

1. [Introduction](#intro)

    1.1 [What is Sharding ?](#shard)

    1.2 [Why is Sharding Important ?](#imp)

2. [Sharding in Mongodb Database](#shard_mongo)

    2.1 [Cluster Illustration](#cluster)
    2.2 [Understanding Shard Key and Chunks](#key)
    2.3 [Configure Sharding](#config)


## <a name="intro">1. Introduction</a>

This article provides a friendly guide to understanding the concept of sharding, its importance, and a step-by-step explanation of how to establish sharding in MongoDB, one of the leading NoSQL databases.

### <a name="shard">1.1 What is Sharding?</a>
Sharding is a process of splitting data across multiple servers or databases to manage large datasets and ensure better performance. It helps distribute data so that no single server is overloaded.

### <a name="imp">1.2 Why is Sharding Important?</a>
As data grows, a single server may struggle to handle the load. Sharding solves this by distributing the workload, improving performance, scalability, and reliability.

## <a name="shard_mongo">2. Sharding in MongoDB Database</a>
MongoDB uses sharding to scale horizontally, allowing data to be distributed across multiple shards. Each shard contains a subset of the data, and a central component called the mongos ensures that applications can access the correct shard seamlessly. This approach ensures high availability and efficient data management for large-scale applications.


### <a name="cluster">2.1 Cluster Illustration</a>

In a MongoDB sharded cluster, the architecture typically includes:

- **Shards**: These are the data-bearing servers, each containing a portion of the total data.
- **Config Servers**: These maintain metadata and the mapping of data to shards.
- **Mongos Router**: This acts as a query router, directing client queries to the appropriate shard.

![sharding](https://github.com/user-attachments/assets/4d0b719a-cde6-436a-af1e-f3ae7a200413)

### <a name="key">2.2 Understanding Shard Key and Chunks</a>

### Shard Key
- The shard key is a field (or a combination of fields) in your MongoDB documents used to determine how data is distributed across shards.
- It acts as a guide for the database, helping it decide where to place or retrieve each document within the sharded cluster.
- Common shard key considerations include:
  - **Cardinality**: Choose a key with a wide range of unique values to ensure even distribution.
  - **Query Patterns**: Select a key that aligns with your application's most common queries for optimal performance.

### Chunks
- Data is divided into chunks, which are contiguous ranges of data defined by the shard key.
- MongoDB automatically splits chunks when they exceed a configured size to maintain performance and balance.
- Chunks can be migrated between shards to ensure even distribution of data and avoid overloading a single shard.
- If your shard key is **userId**, MongoDB might divide the data into chunks like:
   - Chunk 1: userId from 1 to 1000
   - Chunk 2: userId from 1001 to 2000
- These chunks are distributed across shards, and as new data comes in, MongoDB ensures it is added to the correct chunk based on the shard key.


### <a name="config">2.3 Configure Sharding</a>

**[0] -**  Inside a folder called `sharding`, create the following folders:
1. **config** : for the Config Server.
2. **rs1** : for single member replica set of shard1.
3. **rs2** : for single member replica set of shard2.
4. **log** : for mongos router logs.

**[1] -**  Configure the config server:

`• mongod --configsvr --replSet configRS --bind_ip 127.0.0.1 --port 27018 --dbpath sharding/config --logpath sharding/config/mongos.log`

- the `--configsvr` option indicates to the mongod that you are planning to use it as a configserver.
- connect to the configserver and initiate the `replica set`:
   - `• mongosh --host 127.0.0.1  --port 27018`
   - `• rs.initiate({ _id:"configRS", configsvr: true, members:[{ _id:0, host:"127.0.0.1:27018" }]})`


**[2] -** Start the first shard:

`• mongod --shardsvr --replSet shardset1 --bind_ip 127.0.0.1 --port 27030 --dbpath sharding/rs1 --oplogSize 200`

- the `--shardsvr` option indicates this is a shard
- connect to the first shard and initiate the `replica set`:
  - `• mongosh --host 127.0.0.1  --port 27030`
  - `• rs.initiate({ _id:"shardset1", members:[ { _id:0, host:"127.0.0.1:27030" }]})`

**[3] -** Start the second shard

`• mongod --shardsvr --replSet shardset2 --bind_ip 127.0.0.1 --port 27040 --dbpath sharding/rs2 --oplogSize 200`
- connect to the second shard and initiate the `replica set`:
   - `• mongosh --host 127.0.0.1  --port 27040`
   - `• rs.initiate({ _id:"shardset2", members:[ { _id:0, host:"127.0.0.1:27040" }]})`


**[4] -** Configure mongos:

`• mongos --configdb configRS/127.0.0.1:27018  --bind_ip 127.0.0.1 --port 27019 --logpath sharding/log/mongos.log`
- we must provide where our config server using the option `--configdb`

**[5] -** Add the shards to mongos router:

- connect to `mongos`:
`• mongosh  --port 27019`
- then: 
`• sh.addShard("shardset1/127.0.0.1:27030")`
`• sh.addShard("shardset2/127.0.0.1:27040")`

**[6] -** Enable sharding on a Database:

- inside mongos:
`• sh.enableSharding("movies")`
- Operations like enabling sharding and defining shard keys must always happen through the mongos router.
- in sharding we enable sharding for the database then enable individual collections for the sharding.


**[7] -** changing Chunk Size in mongos: (for test purpose)

- inside `mongos` to Get `Chunk Size`, switch to `config` database
`• use config`
- then:
`• db.settings.find({ _id: "chunksize" })`
- if no document is returned, it means the chunk size is set to the default value of 64 MB
- to change it to 1 MB:
```js
• db.settings.updateOne(
  { _id: "chunksize" },
  { $set: { value: 1 } },
  { upsert: true }
);
```
- re-check again `db.settings.find({ _id: "chunksize"})`

**[8] -** Adding data to collection:

- insde `mongos` switch to `movies` database then trigger the following command: 
```js
• for(var i=0; i<100000; i++){db.films.insertOne({name: "film-"+i, createdAt: new Date() });}
```
- this loop may take long time, you can open a new instance of mongos and continue work.


**[9] -**  creating an index on a key: (will be the shard key)

`• db.films.createIndex({name:1})`

**[10] -**  adding collection to sharding:

`• sh.shardCollection("movies.films", {name: 1})`

- `"movies.films"` : is a collection namespace [database.collection]
- `{name: 1}` : shard key, must be indexed

**[11] -** checking data distribution across the shards:

To analyze the distribution of data across shards for a particular collection (`films` in this case) in a sharded cluster:

`• sh.status()`

The command provides detailed information about how the data is distributed among the shards.

- Example Output:

```js
...
chunkMetadata: [ 
    { shard: 'shardset1', nChunks: 2 },
    { shard: 'shardset2', nChunks: 1 }
],
chunks: [
    { min: { name: MinKey() }, max: { name: 'film-25467' }, 'on shard': 'shardset1', 'last modified': Timestamp({ t: 2, i: 0 }) },
    { min: { name: 'film-25467' }, max: { name: 'film-40936' }, 'on shard': 'shardset1', 'last modified': Timestamp({ t: 3, i: 0 }) },     
    { min: { name: 'film-40936' }, max: { name: MaxKey() }, 'on shard': 'shardset2', 'last modified': Timestamp({ t: 3, i: 1 }) }
],
...
```
