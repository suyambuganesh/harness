# Harness 0.3.0 Requirements

Harness 0.3.0 will run big-data Lambda style algorithms based, at first, on Spark. This will enable the use of MLlib and Mahout based algorithms in Engines. These include the Universal Recommender's CCO and MLlib's LDA in separate Engines.

Basic requirements:

- Spark 2.3 or latest stable
- MongoDB 4.x: read a collection into a Dataframe. Question: does the Spark lib for MongoDB support Spark 2.3?
- Elasticsearch 6.x: write a Spark distributed dataset (maybe Dataframe) to an ES index.Question: does the Spark lib for Elasticsearch support Spark 2.3?
- Scala 2.11
- Mahout 0.14.0-SNAPSHOT: this runs on Spark 2.3 (or so they say&mdash;compiles but untested except for unit tests)
- Either Yarn or Kubernetes for job management. The hard requirement is being able to run more than one job at once on a Spark cluster, something like Yarn's cluster mode, and being able to run the driver on a Spark worker. **Note:** rumor has it that k8 support is not as solid as Yarn so may not be a good choice, a small amount of research is required to answer this.

# Spark Support

Each Engine that uses Spark will contain a section of JSON, which defines name: value pairs like the ones spar-submit supports and like this:

```json
"sparkConf": {
    "master": "yarn", // or possible to do any master, "local[8]" etc.
    "deploy-mode": "cluster",
    "driver-memory": "4g",
    "executor-memory": "2g",
    "executor-cores": 1,
    "spark.serializer": "org.apache.spark.serializer.KryoSerializer",
    "spark.kryo.registrator": "org.apache.mahout.sparkbindings.io.MahoutKryoRegistrator",
    "spark.kryo.referenceTracking": "false",
    "spark.kryoserializer.buffer": "300m",
    "spark.es.index.auto.create": "true",
    "spark.es.nodes': "es-node-1,es-node-2"
    // any arbitrary spark name: value pairs added to the 
    // spark context without validation since some are params
    // for libs, like es.nodes
}    
```

The `sparkConf` section in the Engine config will validate only part of the config that is specific to the Engine's needs. Some `name: value` pairs are simply put in the context as params for libs. Above some are used by Mahout and Elasticsearch but others are known to the Spark code like `master` and `deploy-mode`

Note that there are special requirements to support Yarn's Cluster mode deployment and these will need to be enumerated here and supported.

# MongoDB Spark Support

MongoDB is already supported in the generic `Store` and `DAO` with a custom implementation for Mongo. Since not all Engines will require the reading (or writing) of distributed Spark datasets it might be good to separate this into a trait or some king of extension of the base classes. This support should be something the Engine can pick. We ideally want to allow future versions of Harness to inject the `DAO` and `Store` implementation so whatever mechanism used should probably allow injection. This might be something like a `DAO` abstract interface and a `SparkDAO` interface with implementation of the same structure. In any case injection is not a hard requirement so we can pick a solution without injection if it make things significantly easier.

# MongoDB Query Result Limit and Ordering

The UR requires a way to ask for `Events` from a collection of a certain size and ordered by datetime, which is a part of every Event. The ordering and limit is not part of DAO now, which returns all results.

The UR can work if the ordering is defaulted per collection and does not need to be specified in the query, if that makes the task easier. Mongo allows the ordering and limit to be specified in the query so this is also fine. Mixing a filter with order and limit is ideal so that the UR can ask for members of a collection with `"name": "buy"` and return `"limit": 100`, `"ordered_by": {"date": "descending"}`.

To make the order performant it may be required to create an index on the key that is used to order&mdash;at least with Mongo. Other DBs may not support this but this is not our primary concern now.

This can be encoded in any way that fits the DAO/Store pattern using an appropriate API. This is a general feature of DB Stores so is not part of the Spark extension and so may be best put in the generic `Store`/`DAO` interface.