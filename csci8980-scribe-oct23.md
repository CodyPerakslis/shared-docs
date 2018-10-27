# Notes for CSCI8980 2018/10/23


## CloudPath: A Multi-Tier Cloud Computing Framework
**Long Paper: Presented by Achal Shanthram**
* Cloud computing background (2-tier)
* Edge computing (3-tier with edge aggregator)
* Path computing (generalize to n-tier with successive computer power increases)

Challenges
* Partitioning application logic and optimal task placement
  * Developer specifies
* Some of the infrastructure may be owned by different ISPs
  * ISPs must share compute

System Components
1. Path Execute
  * Each application own container (no explicit shared state like lambda)
  * Specify location (each function own URI)
2. Path Store
  * Hierarchy of independent object stores
  * Using Cassandra
  * Each parent superset of it's children
    * Hierarchal Cassandra
    * Reduce read to a heavy extend
  * Writes propagated by a daemon
  * Eventual consistency across nodes
  * Time-stamping, assumes tightly synchronized (not always true)
3. Path Route
  * Routes requests with NGINX proxy
  * If not deployed on that node, calls PathDeploy to run, or re-routed to parent node
  * During a handover, it is up to the client to reestablish a new connection
  * End-to-end principle studied in distributed systems
4. Path Deploy
  * Makes decisions based on user preferences, system policies, resource availability to deploy here or not
5. Path Monitor and Init
  * Monitor to monitor health and paths
  * Init is the web interface to initialize

Test cases
* Face detection (compute offloaded from device)
* Face recognition (geo-specific trained model)
* IoTStat aggregation

Validation
* Java servlets even smaller than a container (3.5-4 seconds vs 13.2 seconds running Java)
* 0.9ms overhead routing
* Handover: ~32 ms to reestablish new TCP connection

Read performance substantially improved
Writes
* 2 round trips cloud --> edge (do you have it, then here it is)

### Discussion
* Can CloudPath be used for real time systems?
If not, how could it be supported?

They don't consider the actual size of the data.  

It would not be suitable for stock trading. Sequence of high fequency transactions. Only eventually consistent, so not consistent. Improvements: writes could be routed directly to the data center for small datas.

Directly route the writes to the cloud for better consistency. Sync then read (sync happens on a daemon based on frequently done queries). Latency will also go up.

* How can it be integrated with any of the geo-distributed analytic systems?

How to optimize bandwidth (this only does CPU and memory). Choose the system based on the application. Like AWStream to where to run that function rather than the developer.

Focus on mobile devices within a locality. Less replication required (focus on clients and not synchronization).  
You don't really need to replicate.  
Need to split into multiple regions.  

Eventual consistency. In the Gaia paper, you keep the model in a hub, with pieces separated and when sig above threshold, update. Updates only updated at error rate threshold. Implement this effectively that way.

Challenge: Partition for geo-specific applications.

Similarity with Steel on application placement and storage decisions
  * Focused on dependency and DAGs, this is more just a serverlet
  * This way is more scalable
  * State storage in Steel causes the overhead by Steel
  * ad-hoc topology for nodes can be added
    * Random tree topology which can be optimized to produced balanced trees (globally or locally)
  * Is a tree topology good enough for most applications?
    * [KaZaA](https://en.wikipedia.org/wiki/Kazaa) (super peers). Not with pure peer-to-peer


### Counter-point
This requires a channel to be established with a mobile endpoints, which suffers from the mobility problem. It does not effectively handle the lack of a central source of data [4].

## Towards a Solution to the Red Wedding Problem
**Short Paper: Presented by Mahendra Maiti**

Advantage of Edge
* reduced user-percieved Latency and load on origin
Current inability to handle write speaks  
Traditionally a cache to handle read spikes  

When GoT aired, a huge number of writes to update the description of an episode.

Reads handles from PoP, but writes routed back
(Points of presences)

Concurrent writes and database action challenges

Every request invokes a lambda, and each container is initialized with a copy of a local db to take updates from that db, those dbs are eventually merged

Anti-entropy session takes info from local DBs, and CRDTs to eventually converge

Heavy use of lambda: major challenge it many communications (communication between containers). Use an indirect method (AMQP) shared topic for containers.

Lambda limits:
* Intercommunication
* Concurrent invocations

Message brokering at the edge: latency costs.

Each instance aware of others, termination notification, message broker (nice-to-have-features)

* Lambda cold-start time, concurrent invocations synchronized, inter-replica latency (similarly to if AWS did it themselves). Bottleneck for high writes

* Not all CRDTs
* Cold source

### Discussion
* What are the current drawbacks of lambda?

* Other data types beyond CRDTs for merges
* Time-stamping the writes?
* Anti-entropy determine them to be in proper order

* Limitations of this system?

* Multiply replicas during the writes. Latency.
* Accuracy and correctness. Batching independent writes. Alright if there some staleness is allowed. Not too much communications.  
* Writes batched on user side, so vector clocks not useful (to handle CRDTs)

* Suitable only for non-critical real-time applications
* Batch updates not the best for a real-time system (propagate immediately)

### Citations
Cited by one paper [5], on fog resource management. Paper is still relatively young with the database mentioned a prototype design.

## Apache OpenWhisk
**Demo: Presented by Aishwarya and Manish**  
Serverless cloud platform service

Serverless: event driven platform to execute code

Traditional messages:
  * Continuous polling
  * Capacity management
Serverless:
  * Scales inherently

Good applications:
  * Sort-division applications
  * Micro-service architecture
  * Small: ingestion, conversion, filtering
  * Image recognition with model passed in

Components of the model:
  * Actions (functions)
  * Trigger
  * Rules (bind triggers to rules)
  * Chaining (sequences of actions)

Some basic applications available

Architecture
  * Edge: NGINX
  * Controller (Consul, CouchDB)
  * Kafkak
  * Docker invoker

Docker containers:
  * isolation, portability, resource control
  * action invoke --> docker run
  * Code injected into prewarm container (container that can run anything)
    * Opt-for cold start

On IBM cloud  
  * Implement wordcount
  * Monitor page
  * Console and cli

Scaling
  * Limits to application size
  * Break down actions into containers

Cold vs Prewarm vs Warm
  * Decided by OpenWhisk
  * OpenWhisk only uses cold
    * Re-uses containers
    * Time-frame for keeping alive
  * Each action separate invoker (even with chaining)
  * DAG structure with chaining


## Resources

[1] Seyed Hossein Mortazavi, Mohammad Salehe, Carolina Simoes
Gomes, Caleb Phillips, and Eyal de Lara. [Cloudpath: A Multi-
tier Cloud computing framework](http://sysweb.cs.toronto.edu/publication_files/0000/0309/sedgec17-final51.pdf).  In
Proceedings of the Sec-
ond ACM/IEEE Symposium on Edge Computing
, SEC ’17, pages
20:1–20:13, New York, NY, USA, 2017. ACM.

[2] Christopher Meiklejohn, Heather Miller, and Zeeshan Lakhani.   [Towards a
solution to the red wedding problem](http://christophermeiklejohn.com/publications/hotedge-2018-preprint.pdf). In
USENIX Workshop on Hot Topics in Edge
Computing (HotEdge 18)
, Boston, MA, 2018. USENIX Association.

[3] [Apache OpenWhisk](https://openwhisk.apache.org/)

[4] Ioannis Psaras, Onur Ascigil, Sergi Rene, George Pavlou, Alex Afanasyev, and
Lixia Zhang. 2018. [Mobile Data Repositories at the Edge](http://discovery.ucl.ac.uk/10055929/1/Psar-18-edge-mobile-data.pdf). In
Proceedings of the 1st
USENIX Workshop on Hot Topics in Edge Computing (HotEdge’18).

[5] A. Yousefpour et al. [All One Needs to Know about Fog Computing and Related Edge
Computing Paradigms: A Complete Survey](https://arxiv.org/pdf/1808.05283.pdf)
