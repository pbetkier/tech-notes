# MongoDB for Developers - 10gen education

## Schema design

### Living without constraints:

Instead of having foreign key constraints you embed related data in the document. Operations on a single document are atomic.

#### 1:1 relations

Two approaches:

* Two separate documents, id for reference.
* Embed one document into another.

Some tips:

* If your document A is read very often and related document B is large, perhaps embedding is bad idea.
* If document B is changed very often, perhaps you don't want to make it a part of A. That would cause A to be fetched as well.
* If embedding B would make you exceed the limit of 16MB per document.
* Embed, if documents are changed together and you cannot afford inconsistency.

#### 1:n relations

Embedding on the n side may cause inconsistency. Think whether it's a one-to-many or one-to-few situation:

 * one-to-many: true linking.
 * one-to-few: embed.

#### n:n relations

For few-to-few you can use linking on one of the sides - which one depends on your queries. Inconsistency can occur. Embedding would cause duplication.

For many-to-few embedding is bad idea, because you won't be able to create one of your models on its own, without relations.

### Transactions

There are no transactions known from relational databases, but operations on a single document are atomic.

How to ensure that you don't need transactions:

 * Model your domain so that this atomcity is sufficient.
 * Make locks on the software side: e.g. banks are able to perform transactions between each other even though they use separate systems and schemas.
 * Tolerate incosistency to some extent.

### Benefits of embedding

You get read performance, because of reading from a contiguous area on disk. On the other hand, if you extend your document a lot, the document will get moved and slow writes down.

### Representing trees

In relational databases you would have a parent attribute and navigating to the tree root would require iterative querying for parent.

Here, you can keep a list of ancestors.

### When to denormalize

You denormalize when:

* 1:1, embed, but there is no duplication, so no anomalies.
* 1:n, embed, no anomalies if from many to one.
* n:n, linking.

If you want to store blobs of more than 16MB you should use GridFS, which will split your file into chunks.

## Performance, indexes

Indexes are the most important factor. Index keys are ordered in a b-tree. A key is a list of attributes, e.g. (name, address, code).

You can use an index, if you provide a left-most set of attributes, so (name), (name, address) or (name, address, code).

Indexes slow inserts down, but make reads much faster. They take space on disk. Each collection has an index on the _id field by default.

### Other index types

You can have a multikey index on an array field, which will create an index on every element of the array. You cannot have a compound index on two array fields though - this blocks against polynomial explosion.

If a compound index is already created in a document, you won't be able to insert a document with array field where simple type was expected.

Index may be unique, but it is not by default. Id index is unique, even though getIndexes() doesn't say so.

You can have an index on nested attributes.

Sparse index is a unique index on keys, that don't exist in every document. It indexes only documents with non-null key value. If you use that index for sorting, it will skip the documents with null value on the key.

### Creating indexes

Indexes are creating in foreground by default, which is fast but blocks writes. Background is slow, but doesn't block writers. You can have one background creation at a time.

In production environment you rather use background creation, so as not to block writers. But you can play with forefround if you have a replica set.

### Index inspection

Use getIndexes() to list all indexes on a collection. Use explain() for details about executing a query:

 * query execution time in ms,
 * cursor type (index type),
 * number of docuements/indexes scanned,
 * number of documents scanned,
 * number of returned documents,
 * bounds used to lookup the index,
 * many more.

How does db know which index to use? The first time you run a query, it runs all possible query plans in parallel, returns the result of the first plan to finish and remembers than this plan should be used for the future queries of this type. Every 100 queries the experiment is run again.

You can hint which index to use. 

Indexes should fit in the memory for maximum performance.

### Geospatial indexes

It's and index of type '2d'. It allows to use *$near* operator, which returns documents in order of increasing distance, you set a limit.

You can make it consider the spherical model of the world. Distance will be defined in radians then - arc between two points on a circle with the middle being the earth center.

### Profiling

Mongo automatically writes slow queries (>100ms) info to a log.

#### Profiler
You can also configure a profiler, which will write entries to system.profile for queries, 3 levels:

 * 0: off (default),
 * 1: only slow queries (you define the threshold in ms),
 * 2: all queries (useful for debugging).

#### mongotop
Shows where mongo spens its time, in which collections etc.

## Aggregation

Aggregation works as a pipeline of aggregation strategies:

 * $project: select and reshape documents - remove keys, add new keys. Also used to clean up the documents before returning, to eliminate unnecessary data before grouping for performance.
 * $match: filter on documents,
 * $unwind: unwinds an array of length X into X documents. Makes it possible to group on elements of an array. Double $unwind creates the cartesian product of 2 array + the rest of the document.
 * $group: aggregation, like GROUP BY,
 * $sort: in memory, used after grouping, doesn't use indexes, used at the end of the chain to make the output neater to the user or with conjunction with $limit,
 * $skip & $limit: make sens when sorting is used.
 * $first & $last: return first or last element in a group, makes sense with $sort.

You can group with the following expressions: $sum, $avg, $min, $max, $push, $addToSet ($push but uniquely), $first, $last.

## Application engineering

### Replication

Replication is for fault-tolerance. You can create a replica-set (minimum 3 nodes), with one primary. When the primary fails, a new primary is selected among the secondary nodes.

Several node types:

 * regular: the most common, primary or secondary,
 * arbiter: just for majority when voting, e.g. if you have even number of nodes in the replica set,
 * delayed: is behind other nodes, cannot become primary,
 * hidden: used for analysis, cannot be primary.

By default, writes and reads go to primary. You can route reads to secondaries as well, but you may read stale data (there is a lag that is not guaranteed). Options for reads:

 * primary,
 * secondary,
 * secondary preferred,
 * primary preferred,
 * nearest,
 * tagging: tag which nodes to use for reads.

MongoDB is different from some of the other NoSQL DBs - it doesn't offer eventual consistency by default.

#### Failover and rollback

Failover may result in a rollback for data that was already commited. Failover takes time, during that time you can't complete reads nor writes. Your driver will throw an exception in that case.

You need to catch those exceptions and retry to perform you operation after some small delay. You should allow to perform a few retries.

Keep in mind that you can get an exception even when the data was saved correctly (e.g. a network error). So in your retry loop you should check for DuplicateKeyError to prevent from trying to save the same document again.

#### Write concerns

Writes in mongoDB are by default fire & forget. If you want to know about an error you need to call getLastError. Drivers typically do it for you. You can specify, that you want to wait for the data to arrive in the journal before getLastError is called.

Two options:

 * w: how many nodes must acknowledge the write, you wait for duration of wtimeout,
 * j: should you wait for the data to arrive in the journal.

#### Last thoughts

Replica sets are very transparent for the developer, but:

 * there are seed lists,
 * write concerns need to be understood,
 * read preferences need to be thought out,
 * errors can happen.

### Sharding

Gives horizontal scalability. Shards may actually be replica sets themselves.

Mongos is a router that will take care of distributing your operations to correct shards basing on the shard key. If a query doesn't utilise a shard key, then the query will go to every shard and mongos will perform a merge. Also: 

 * Each insert will require the shard key to be specified.
 * Shard key is immutable.
 * You need an index that stands with the shard key.

When you shard, you will probably add replication as well.

Choosing a shard key:

 * sufficient cardinality (variety of values, at least equal to the number of shards),
 * not increasing monothonically (will not split the writes), so not a timestamp or increasing id,
 * you need to think about it in advance.
