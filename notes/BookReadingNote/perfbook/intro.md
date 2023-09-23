# Introduction to parallel programming

## Historical Difficulties

1. The historic high cost and relative rarity of parallel systems. 
2. The typical researcher’s and practitioner’s lack of experience with parallel systems. 
3.  The paucity of publicly accessible parallel code. 
4.  The lack of a widely understood engineering discipline of parallel programming. 
5. The high overhead of communication relative to that of processing, even in tightly coupled shared-memory computers.



## Alternatives to PP

### Multiple instances

Run multiple instances of a sequential application.



### Optimize serial program

#### Bottlenecks undermine parallel computing

1. Main memory. If a single thread consumes all available memory, additional threads will simply page themselves silly. 
2.  Cache. If a single thread’s cache footprint completely fills any shared CPU cache(s), then adding more threads will simply thrash those affected caches
3. . Memory bandwidth. If a single thread consumes all available memory bandwidth, additional threads will simply result in additional queuing on the system interconnect. 
4.  I/O bandwidth. If a single thread is I/O bound, adding more threads will simply result in them all waiting in line for the affected I/O resource.

  In fact, if the program was reading from a single large file laid out sequentially on a rotating disk, parallelizing your program might well make it a lot slower due to the added seek overhead. You should split the file into chunks which can be accessed in parallel from different drives, cache frequently accessed data in main memory, or, if possible, reduce the amount of data that must be read



## Why hard

### Work partitioning

1. uneven partitioning can result in sequential execution once the small partitions have completed
2. Furthermore, the number of concurrent threads must often be controlled, as each such thread occupies common resources
3. However, because communication incurs overhead, careless partitioning choices can result in severe performance degradation.

### Parallel Access Control

1. Form of access to resource depends on its location: local or remote
2. How threads coordinate access to resources with synchronizations: message passing, locking, transactions, reference counting, explicit timing, shared atomic variables, and data ownership. This might cause  deadlock, livelock, and transaction rollback.

### Resource partitioning and replication

It is usually wise to begin parallelization by partitioning your write-intensive resources and replicating frequently accessed read mostly resources.

### Interact with hardware

Hardware interaction is normally the domain of the operating system, the compiler, libraries, or other software-environment infrastructure. However, developers working with novel hardware features and components will often need to work directly with such hardware.

