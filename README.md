# ForkBase #

*For details of the ForkBase technologies, please refer to [this post](https://blog.acolyer.org/2018/06/01/forkbase-an-efficient-storage-engine-for-blockchain-and-forkable-applications/) or our [research paper](http://www.vldb.org/pvldb/vol11/p1137-wang.pdf).*

## Introduction ##

ForkBase is a distributed data storage system which has rich semantics and a set of features that unifies and adds values to many classes of next generation applications.
In particular, ForkBase is mainly designed for applications that require internal data representation with versions, chains and branches.
Typical applications include Git-like versioning control, Blockchain, Collaborative analytics and versioned OLTP.
ForkBase focuses on providing such applications high performance, flexibility and high-level semantics, while reducing application-level development and maintenance effort.

## Features ##

We describe the most critical features in ForkBase along with the corresponding design principles.
All these features are selected to be desirable and beneficial to all applications built on top of ForkBase.

### In-memory ###

The increasing availability of large memory makes it feasible to use RAM as main data repository for data storage systems.
In-memory systems deliver low latency comparing with the disk-based approaches.

To support real-time analytics, all data served in ForkBase is kept in memory.
To ensure data persistence, newly written data will be asynchronously persisted to disk.

### Hardware Consciousness ###

New hardware primitives, such as Remote Direct Memory Access (RDMA), Hardware Transactional Memory (HTM), Non-Volatile Memory (NVM), offer new opportunities to improve system performance.

Current ForkBase system supports RDMA for fetching remote data with low latency.
The immutability feature of ForkBase promises that RDMA can achieve much better scalability, as there are no data synchronization or consistency issues.

Other hardware primitives will be integrated into ForkBase as planned.

### Immutability and Versioning ###

Immutability means that data never changes and any update results in a new version.
It is important for scenarios where each data version has its own value and thus should be persisted for future analysis.
From the system’s perspective, immutability simplifies the design for fault tolerance and replication.

In ForkBase, every piece of data is immutable once fed into the system.
Each data update is attached with a unique identifier, i.e., version number, which is used later to retrieve the data.
It is noteworthy that the data identifier is content addressable, i.e., purely based on the data content.
In other words, two pieces of data with same content share the single identifier and no duplicated copies are stored.

### Named and Unnamed Branching ###

Most data-driven applications require powerful management of collaboratively produced data, especially in the branching manner.
Multiple branches provide the isolation of local modifications from different collaborators/clients/users
and makes it more manageable to advance the global view of data.

Generalized from many applications, ForkBase provides two categories of branching mechanisms:
**named branching** and **unnamed branching**.

* Named branching is for those applications that need explicit management of their branches,
i.e., branches are created by branching operations.
Such applications include Git-like version control and collaborative analytics.

* Unnamed branching is for those applications that implicitly branches data from their application logics,
i.e., branches are created by concurrent put/update operations.
Such applications include Blockchain and Transaction with MVCC.

Creating a branch in ForkBase is lightweight, in which no existing data is copied.


### Sophisticated Data Types ###

We observe that sophisticated data types, such as list and map, are commonly desired from applications’ perspective.
It is more efficient and less error-prone for users to directly use those built-in data types,
than to build their own types on top of the key-value abstraction.

ForkBase supports two categories of data types: **primitive type** and **chunkable type**.

* Primitive type includes String, Boolean, Integer, Decimal, etc..
An object of these types is stored consecutively as a sequence of bytes in the internal storage.

* Chunkable type includes Blob, Map, List, Set, etc..
An object of these types is composed of a number of data chunks in the internal storage,
which may be shared among different versions/objects.
With the smart chunking strategies, ForkBase ensures that an update of the object does not need to re-write all its chunks.

With built-in data types, it is efficient in ForkBase to detect the different parts of two objects,
thus supporting fast auto-merge operations.
Besides, it is also facilitates migrating existing data sources, e.g., relational tables,
using built-in data types without additional efforts.


### Data Deduplication ###

The number of data versions and branches is always large and increases rapidly in many applications.
This scenario may result in the explosion of data volumes, if each version and branch has its own copy of the data.
Hence, data deduplication ensures that no duplicated copies of data should be kept.

ForkBase supports data deduplication, and ensures that new data versions and branches will share most unchanged content from existing objects.
However, unlike directly writing delta and reconstructing from a long chain of deltas, ForkBase provides more efficient data representation.

Data deduplication in ForkBase is achieved at the chunk level.
Each object of chunkable type is internally represented as a sequence of chunks.
Smart chunking strategies are employed to ensure that the consequent chunk sequences of the object are not affected by its evolving history, hence there will not be long chains of deltas from small updates.

### Sharing and Security ###

The exponential growth of data can be attributed to the growth in user-generated, machine-generated data and the need to log and collect data for future analytics.
Before the data can be meaningfully used, the owners must be able to share their data among each other.
It means that only authorized users can access the shared data.

In ForkBase, data sharing can be achieved in two ways: sharing inside a ForkBase instance, and sharing between ForkBase instances.

Sharing inside a ForkBase instance is easy. What a user needs to do is to grant another user access to a specific branch.
That user will get access only to that branch.

Sharing between ForkBase instances involves transferring data via the network.
Thanks to the data deduplication techniques, it is fast to detect which chunks are missing in the target ForkBase instance and only transfers those necessary chunks via the network.
