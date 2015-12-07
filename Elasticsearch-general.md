# Implementing Elasticsearch on Azure

## Overview

Elasticsearch is a highly scalable open-source search engine and database. It is suitable for situations that require fast analysis and discovery of information held in big datasets. Common scenarios include:
* Large-scale free text search, where documents that match combinations of search terms can be quickly located and retrieved.
* Event logging, where information can arrive from a variety of sources. The data may need to be analyzed to ascertain how a chain of events has led to a specific conclusion.
* Storing data captured from remote devices and other sources. The data could contain varying information, but a common requirement is to present this information in a series of dashboards to enable an operator to understand the state of the overall system. Applications can also use the information to make quick decisions about the flow of data and business operations that need to be performed as a result.
* Stock control, where changes to inventory are recorded as goods are sold. Business systems can use this information to report stock levels to users, and reorder stock if the level of a product runs low. Analysts can examine the data for trends to help determine which products sell well under what circumstances.
* Financial analysis, where market information arrives in near real-time. Dashboards can be generated that indicate the up-to-the-minute performance of various financial instruments which can then be used to help make buy/sell decisions.

---

This document provides a brief introduction to the general structure of Elasticsearch and then describes how you can implement an Elasticsearch cluster using Azure. The document focuses on best practices for deploying an Elasticsearch cluster, concentrating on the various functional performance and management requirements of your system, and considering how your requirements should drive the configuration and topology that you select.

> **Note**: This guidance assumes some basic familiarity with Elasticsearch and its purpose. For more detailed information, visit the [Elasticsearch website](https://www.elastic.co/products/elasticsearch).

## The Structure of Elasticsearch

Elasticsearch is a document database highly optimized to act as a search engine. Documents are serialized in JSON format. Data is held in indexes, implemented by using [Apache Lucene](https://lucene.apache.org/), although the details are abstracted from view and it is not necessary to fully understand Lucene in order to use Elasticsearch.

### Clusters, Nodes, Indexes, and Shards

Elasticsearch implements a clustered architecture that uses sharding to distribute data across multiple nodes, and replication to provide high availability.

Documents are stored in indexes. The user can specify which fields in a document are used to uniquely identify it within an index, or the system can generate a key field and values automatically. The index is used to physically organize documents and is used as the principal means for locating documents. Additionally, Elasticsearch automatically creates a set of additional structures acting as inverted indexes over the remaining fields to enable fast lookup and perform analytics within a collection.

An index comprises a set of shards. Documents are evenly dispersed across shards by using a hashing mechanism based on the index key values and the number of shards in the index; once a document is allocated to a shard it will not move from that shard unless its index key is changed. Elasticsearch distributes shards across all available data nodes in a cluster; a single node may initially hold one or more shards that belong to the same index, but as new nodes are added to the cluster Elasticsearch relocates shards to ensure a balanced load across the system. The same rebalancing applies when nodes are removed.

Indexes can be replicated. In this case each shard in the index is copied. Elasticsearch ensures that each original shard for an index (referred to as a “primary shard”) and its replica always reside on different nodes.

> **Note**: The number of shards in an index cannot be easily changed once the index has been created, although replicas can be added.

When a document is added or modified, all write operations are performed on the primary shard first and then at each replica. By default, this process is performed synchronously to help ensure consistency. Elasticsearch uses optimistic concurrency with versioning when writing data. Read operations can be satisfied by using either the primary shard or any of its replicas.

Figure 1 shows the essential aspects of an Elasticsearch cluster comprising three nodes. An index has been created that consists of two primary shards with two replicas for each shard (six shards in all).

![](figures/Elasticsearch/ElasticsearchCluster1.png)
**Figure 1.**
A simple Elasticsearch cluster comprising two primary nodes and two sets of replicas

In this cluster, primary shard 1 and primary shard 2 are located on separate nodes to help balance the load across them. The replicas are similarly distributed. If a single node fails, the remaining nodes have sufficient information to enable the system to continue functioning; if necessary, Elasticsearch will promote a replica shard to become a primary shard if the corresponding primary shard is unavailable.
When a node starts running it can either initiate a new cluster (if it is the first node in the cluster), or join an existing cluster. The cluster to which a node belongs is determined by the *cluster.name* setting in the elasticsearch.yml file.

### Node Roles

The nodes in an Elasticsearch cluster can perform the following roles:
* A data node which can hold one or more shards that contain index data.
* A client node that does not hold index data but that handles incoming requests made by client applications to the appropriate data node.
* A master node that does not hold index data but that performs cluster management operations, such as maintaining and distributing routing information around the cluster (the list of which nodes contain which shards), determining which nodes are available, relocating shards as nodes appear and disappear, and coordinating recovery after node failure. Multiple nodes can be configured as masters, but only one will actually be elected to perform the master functions. If this node fails, another election takes place and one of the other eligible master nodes will be elected and take over.

---

By default, Elasticsearch nodes perform all three roles (to enable you to build a complete working cluster on a single machine for development and proof-of-concept purposes), but you can change their operations through the *node.data* and *node.master* settings in the *elasticsearch.yml* file, as follows:

###### elasticsearch.yml
---
```
node.data: true
node.master: false
```

**Configuration for a data node**

###### elasticsearch.yml
---
```
node.data: false
node.master: false
```

**Configuration for a client node**

###### elasticsearch.yml
---
```
node.data: false
node.master: true
```

**Configuration for a master node**

> **Note**: The elected master node is critical to the well-being of the cluster. The other nodes ping it regularly to ensure that it is still available. If the elected master node is also acting as a data node, there is a chance that the node can become busy and fail to respond to these pings. In this situation, the master is deemed to have failed and one of the other master nodes is elected in its place. If the original master is actually still available, the result could be a cluster with two elected masters, resulting in a "split brain" problem which can lead to data corruption and other issues. The document Configuring, Testing, and Analyzing Elasticsearch Resilience and Recovery describes how you should configure the cluster to reduce the chances of this from occurring. However, ultimately it is a good strategy in a moderate to large cluster to use dedicated master nodes that take no responsibility for managing data.

The nodes in a cluster share information about the other nodes in the cluster (by [gossiping](https://en.wikipedia.org/wiki/Gossip_protocol)) and which shards they contain. Client applications storing and fetching data can connect to any node in a cluster and requests will be transparently routed to the correct node. When a client application requests data from the cluster, the node that first receives the request is responsible for directing the operation, communicating with each relevant node to fetch the data and then aggregating the result before returning it to the client application. Using client nodes to handle requests frees data nodes from performing this scatter/gather work, and enables them to spend their time serving data. You can prevent client applications from accidentally communicating with data nodes (causing them to act as client nodes) by disabling the HTTP transport for the data nodes:

###### elasticsearch.yml
---
```
http.enabled: false
```

Data nodes can still communicate with other data nodes, client nodes, and dedicated master nodes on the same network by using Elasticsearch transport module (which uses TCP sockets to connect directly between nodes), but client applications can only connect to client nodes over HTTP. Figure 2 shows a topology comprising a mixture of dedicated master, client, and data nodes in an Elasticsearch cluster.

![](figures/Elasticsearch/ElasticsearchCluster2.png)
**Figure 2.**
An Elasticsearch cluster showing different types of nodes

### Costs and Benefits of Using Client Nodes

When an application submits a query to an Elasticsearch cluster, the node to which the application connects is responsible for directing the query process. The node forwards the request to each data node and gathers the results, returning the accumulated information to the application. If a query involves aggregations and other computations, the node to which the application connects performs the necessary operations after retrieving the data from each of the other nodes. This scatter/gather process can consume considerable processing and memory resources.

Using dedicated client nodes to perform these tasks allows data nodes to focus on managing and storing data. The result is that many scenarios that involve complex queries and aggregations can benefit from using dedicated client nodes. However, the impact of using dedicated client nodes will likely vary depending on your scenario, workload, and cluster size. For example, data ingestion workloads might be less efficient by using client nodes due to the additional network "hop" that is required when storing data. In a 3-node cluster with 6 shards, if the system is not configured with dedicated client nodes, with all environmental factors and node loadings being equal, there is a 1/3 probability that an application storing or modifying data will connect directly to the most appropriate shard, removing the need to perform an additional network jump in 1/3 of cases. On the other hand, workloads that perform complex aggregations could benefit from using dedicated clients as a single node will take responsibility for each set of scatter/gather operations that these operations perform. In a mixed environment, you should be prepared to run performance tests to assess the impact of using client nodes on your specific workloads.

> **Note**: The document Maximizing Data Aggregation and Query Performance with Elasticsearch on Azure summarizes a set of benchmarks that were performed by the Microsoft patterns & practices development team, partly for this purpose.

### Connecting to a Cluster

Elasticsearch exposes a series of REST APIs for building client applications and sending requests to a cluster. If you are developing applications using the .NET Framework, two higher levels APIs are available – [Elasticsearch.Net & NEST](http://nest.azurewebsites.net/).

If you are building client applications using Java, you can use the [Node Client API](https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/node-client.html) to create client nodes dynamically and add them to the cluster. Creating client nodes dynamically is convenient if your system uses a relatively small number of long-lived connections. Client nodes created by using the Node API are provided with the cluster routing map (the details of which nodes contain which shards) by the master node; this information enables the Java application to connect directly to the appropriate nodes when indexing or querying data, reducing the number of hops that may be necessary when using other APIs. The cost of this approach is the overhead of enrolling the client node into the cluster. If a large number of client nodes appear and disappear quickly, the impact of maintaining and distributing the cluster routing map can become significant.

#### Connection Load Balancing

Elasticsearch enables several mechanisms for implementing connection load balancing. The following list summarizes some common approaches:

**Client-based load-balancing**: If you are building client applications using the Elasticsearch.Net or NEST APIs, you can use a connection pool to round-robin connection requests across nodes, helping to load-balance requests without the need for an external load-balancer. The following code snippet shows how to create an *ElasticsearchClient* object configured with the addresses of three nodes. Requests from the client application will be distributed across these nodes:
###### C Sharp
---
```
var node1 = new Uri("http://node1.example.com:9200");
var node2 = new Uri("http://node2.example.com:9200");
var node3 = new Uri("http://node3.example.com:9200");

var connectionPool = new SniffingConnectionPool(new[] {node1, node2, node3});
var config = new ConnectionConfiguration(connectionPool);
var client = new ElasticsearchClient(config);
```
> **Note**: Similar functionality is available to Java applications through the [Transport Client API](https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/transport-client.html).

**Server-based load-balancing**: You can use a separate load balancer to distribute requests to nodes. This approach offers the advantage of address transparency; client applications do not have to be configured with the details of each node, making it easier to add, remove, or relocate nodes without modifying any client code. Figure 3 shows a configuration that uses a load balancer to route requests to a set of client nodes, although the same strategy can be used to connect directly to data nodes if client nodes are not used.

> **Note**: You can use the [Azure Load Balancer](https://azure.microsoft.com/documentation/articles/load-balancer-overview/) to expose the cluster to the public Internet, or you can use an [internal load balancer](https://azure.microsoft.com/documentation/articles/load-balancer-internal-overview/) if the client applications and cluster are contained entirely within the same private virtual network (VNET).

![](figures/Elasticsearch/ClientAppInstances.png)
**Figure 3.**
Client application instances connecting to an Elasticsearch cluster through the Azure Load Balancer

**Custom load balancing**: You can use [nginx](http://nginx.org/en/) as a reverse proxy server instead of the Azure Load Balancer. Nginx provides several load balancing methods, including round robin, least-connected (a request is routed to the destination with the fewest current connections), and hashing based on the IP address of the client.

> **Note**: You can deploy an nginx server as an Azure VM. To maintain availability, you should create at least two nginx servers in the same Azure availability set.

---

You should consider the following points when determining whether to use load balancing, and which implementation to use:

* Connecting to the same node to handle all requests for all instances of an application can cause that node to become a bottleneck. As the number of threads available in the node become exhausted, requests will be queued and may be rejected if the queue length becomes excessive (do not hard code the connection details of a single node in application code that may be deployed to many users).
* The round-robin mechanism of the Elasticsearch.Net, NEST, and Transport Client APIs handle failed connection requests by retrying the connection against the next available node in the connection pool. A connection to an unresponsive node in the pool can be temporarily marked as *dead*; it may come back to life later, and the pool can ping the node to determine whether it has become active again.
* The Azure load-balancer can transparently redirect requests to nodes based on a number of factors (client IP address, client port, destination IP address, destination port, protocol type). Following this strategy, an instance of a client application running on a given computer will most likely be directed to the same Elasticsearch node. Depending on the probe configuration of the load-balancer, if the Elasticsearch service fails on this node but the VM itself continues running, all connections to this node will timeout while connections by other client instances to other nodes may continue to succeed.
* The Azure load-balancer can be configured to take a node out of rotation if it fails to respond appropriately to health probe requests performed by the load-balancer.

---

### <a name="node-discovery">Node Discovery
Elasticsearch is based on peer-to-peer communications, so discovering other nodes in a cluster is an important part in the lifecycle of a node. Node discovery enables new data nodes to be added dynamically to a cluster, which in turn allows the cluster to scale out transparently. Additionally, if a data node fails to respond to communications requests from other nodes, a master node can decide that the data node has failed and take the necessary steps to reallocate the shards that it was holding to other operational data nodes.

Elasticsearch node discovery is handled by using a discovery module. The discovery module is a plugin that can be switched to use a different discovery mechanism. The default discovery module ([Zen](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-zen.html)) causes a node to issue ping requests to find other nodes on the same network. If other nodes respond, they gossip to exchange information. A master node can then distribute shards to the new node (if it is a data node) and rebalance the cluster.

The Zen discovery module also handles the master election process and the protocol for detecting node failure.

Prior to Elasticsearch version 2.0, the Zen discovery module used multicast communications to allow nodes to contact each other. This makes it very easy to introduce a new node into a cluster, but can also cause security problems if another installation of Elasticsearch on the same network happens to use the same cluster name; the new installation is considered part of the same cluster and shards may be directed to nodes in this installation. Additionally, if you are running Elasticsearch nodes as Azure virtual machines (VMs), multicast messaging is not supported. For these reasons, you should configure the Zen discovery to use unicast messaging and provide a list of valid contact nodes in the elasticsearch.yml configuration file. For more information, see [Zen Discovery](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-zen.html).

> **Note**: With Elasticsearch 2.0 and later, multicast is no longer the default discovery mechanism.

If you are hosting an Elasticsearch cluster within an Azure VNET, you can specify that the private DHCP-assigned IP addressed given to each VM in the cluster should remain allocated (static). You can configure Zen discovery unicast messaging using these static IP addresses. If you are using VMs with dynamic IP addresses, bear in mind that if a VM stops and restarts it could be allocated a new IP address making discovery more difficult. To handle this scenario, you can swap the Zen discovery module for the [Azure Cloud Plugin](https://www.elastic.co/blog/azure-cloud-plugin-for-elasticsearch). This plugin uses the Azure API to implement the discovery mechanism, which is based on Azure subscription information.

> **Note**: The current version of the Azure Cloud Plugin requires you to install the management certificate for your Azure subscription in the Java keystore on the Elasticsearch node, and provide the location and credentials for accessing the keystore in the elasticsearch.yml file. This file is held in clear text, so it is vitally important that you ensure this file is only accessible by the account running the Elasticsearch service.

>Currently, for maximum security it is recommended that you use static IP addresses for master nodes, and use these nodes to implement Zen discovery unicast messaging across the cluster. In the following configuration (taken from the elasticsearch.yml file for a sample data node), the host IP addresses reference master nodes in the cluster:

>`discovery.zen.ping.multicast.enabled: false`  
`discovery.zen.ping.unicast.hosts: ["10.0.0.10","10.0.0.11","10.0.0.12"]`

## General System Guidelines

Elasticsearch can run on a variety of computers, ranging from a single laptop to a cluster of high-end servers. However, the more resources in terms of memory, computing power, and fast disks that are available the better the performance. The following sections summarize the basic hardware and software requirements for running Elasticsearch.

### <a name="memory-requirements"></a>Memory Requirements
Elasticsearch attempts to store data in-memory for speed. A production server hosting a node for a typical enterprise or moderate-sized commercial deployment on Azure should have between 14GB and 28GB of RAM (D3 or D4 VMs). Spread the load across more nodes rather than creating nodes with more memory (Experiments have shown that using larger nodes with more memory can cause extended recovery times in the event of a failure.) However, although creating clusters with a very large number of small nodes can increase availability and throughput, it also escalates the effort involved in managing and maintaining such a system.

Allocate 50% of the available memory on a server to the Elasticsearch heap. If you are using Linux set the ES_HEAP_SIZE environment variable before running Elasticsearch. Alternatively, if you are using Windows or Linux, you can specify memory size in the *Xmx* and *Xms* parameters when you start Elasticseach; set both is these parameters to the same value to avoid the Java Virtual Machine (JVM) resizing the heap at runtime. However, do not allocate more than 30GB. Use the remaining memory for the operating system file cache.

> **Note**: Elasticsearch utilizes the Lucene library to create and manage indexes. Lucene structures use a disk-based format, and caching these structures in the file system cache will greatly enhance performance.

> Additionally, Elasticsearch is written in Java. The maximum optimal heap size for Java on a 64 bit machine is just above 30GB. Above this size Java switches to using an extended mechanism for referencing objects on the heap which increases the memory requirements for each object and reduces performance. The default Java garbage collector (Concurrent Mark and Sweep) may also perform sub-optimally if the heap size is above 30GB. It is not currently recommended to switch to a different garbage collector as Elasticsearch and Lucene have only been tested against the default.

Do not overcommit memory as swapping main memory to disk will severely impact performance. If possible, disable swapping completely (the details depend on the operating system). If this is not possible then enable the *mlockall* setting in the Elasticsearch configuration file (elasticsearch.yml) as follows:

###### elasticsearch.yml
---
```
bootstrap.mlockall: true
```

This configuration setting causes the JVM to lock its memory and prevents it being swapped out by the operating system.

### Disk and File System Requirements

Use Solid State Drives (SSD) for storing shards. Disks should be sized to hold the maximum amount of data anticipated in your shards, although it is possible to add further disks later. You can extend a shard across multiple disks on a node.

> **Note**: Elasticsearch compresses the data for stored fields by using the LZ4 algorithm, and in Elasticsearch 2.0 onwards you can change the compression type. You can switch the compression algorithm to DEFLATE as used by the *zip* and *gzip* utilities. This compression technique can be more resource intensive, but you should consider using it for cold indexes such as archived log data. This approach can help to reduce index size.

It is not essential that all nodes in a cluster should have the same disk layout and capacity. However, a node with a very large disk capacity compared to other nodes in a cluster will attract more data and will require increased processing power to handle this data. Consequently the node can become "hot" compared to other nodes, and this can, in turn, affect performance.

> **Note**: Azure provides several disk storage options, include standard storage, premium storage, and ephemeral storage. Standard storage is based on spinning media whereas premium storage uses SSDs. Depending on the SKU used to implement a VM, ephemeral storage can be implemented as spinning media (A series VMs), or SSDs (D series VMs and above). There are financial and technical tradeoffs that you need to balance when considering the storage options, and these are described in more detail in the document Maximizing Data Ingestion Performance with Elasticsearch on Azure.

If possible, use RAID 0 (striping). Other forms of RAID that implement parity and mirroring are unnecessary as Elasticsearch provides its own HA solution in the form of replicas, and Azure disks save three copies of the disk data anyway.

> **Note**: Prior to Elasticsearch 2.0.0, you could also implement striping at the software level by specifying multiple directories in the *path.data* configuration setting. In Elasticsearch 2.0.0, this form of striping is no longer supported. Instead, different shards may be allocated to different paths, but all of the files in a single shard will be written to the same path. If you require striping, you should stripe data at the operating system or hardware level.

You should also be aware that total I/O throughput for Azure disks attached to a VM is limited by the storage group to which they belong; a single storage account can handle a request rate of up to 20,000 IOPS. If this figure is less than your anticipated disk I/O traffic, then spread the disks for the VMs in your cluster across multiple storage accounts, bearing in mind that a single subscription can create up to 100 storage accounts.

The Lucene library can use a large number of files to store index data, and Elasticsearch can open a significant number of sockets for communicating between nodes and with clients. Make sure that the operating system is configured to support an adequate number of open file descriptors (up to 64000 if sufficient memory is available). Note that the default configuration for many Linux distributions limits the number of open file descriptors to 1024 which is much too small.

Elasticsearch uses a combination of memory mapped (mmap) IO and Java New IO (NIO) I/O to optimize concurrent access to data files and indexes. If you are using Linux, you should configure the operating system to ensure that there is sufficient virtual memory available with space for 256K memory map areas.

> **Note**: Many Linux distributions default to using the Completely Fair Queuing (CFQ) scheduler when arranging to write data to disk. This scheduler is not optimized for SSDs. Consider reconfiguring the operating system to use either the NOOP scheduler or the Deadline scheduler, both of which are more effective for SSDs.

### CPU Requirements

Use CPUs with multiple cores. Prefer more cores over faster CPUs with fewer cores. This is because the default Elasticsearch threadpool size is configured based on the number of CPU cores available. The algorithms used by Elasticsearch are optimized based on these calculations, and it is recommended that you do not change the default threadpool settings of Elasticsearch.

> **Note**: Azure VMs are available in a variety of CPU configurations, supporting between 1 and 32 cores. For a data node, a good starting point is to consider a standard D-series VM, and select either the D3 (4 cores) or D4 (8 cores) SKUs. The D3 also provides 14GB of RAM, while the D4 includes 28GB. If you require premium disk storage, you can use a DS-series VM (consider a DS3 or DS4 machine), while the Dv2-series provides higher powered 2.4 GHz Intel Xeon processors if you suspect that CPU performance is likely to be a critical factor in limiting throughput. The G-series (for standard storage) and GS-series (for premium storage) use Xeon E5 V3 processors which may be useful for workloads that are heavily compute-intensive, such as large-scale aggregations. For the latest information, visit [Sizes for Virtual Machines](https://azure.microsoft.com/documentation/articles/virtual-machines-size-specs/).

### <a name="network-requirements"></a>Network Requirements

Elasticsearch requires a network bandwidth of between 1 and 10Gbps, depending on the size and volatility of the clusters that it implements. Elasticsearch migrates shards between nodes as more nodes are added to a cluster. Elasticsearch assumes that the communication time between all nodes is roughly equivalent and does not consider the relative locations of shards held on those nodes. Additionally, replication can incur significant network I/O between shards. For these reasons, avoid creating clusters on nodes that are geographically dispersed.

Note that network bandwidth in Azure VMs is restricted by the total network usage of the physical hardware on which the VMs run. VMs that share hardware will have less bandwidth than those that run on dedicated machinery, but are cheaper to deploy. This means that other than selecting the series and SKU for your VMs, you have little control over the available network bandwidth. For example, although you can create VMs with multiple virtual NICs, doing so will not increase the overall bandwidth available to your VMs; it just means that more virtual NICs compete for the same physical network resources.

### Software Requirements

You can run Elasticsearch on Windows or on Linux. The Elasticsearch service is deployed as a Java jar library and has dependencies on other Java libraries that are included in the Elasticsearch package. You must install the Java 7 (update 55 or later) or Java 8 (update 20 or later) JVM to run Elasticsearch.

> **Note**: Other than the *Xmx* and *Xms* memory parameters (specified as command line options to the Elasticsearch engine – see [Memory Requirements](#memory-requirements)) do not modify the default JVM configuration settings. Elasticsearch has been designed using the defaults; changing them can cause Elasticsearch to become detuned and perform poorly.

### Deploying Elasticsearch on Azure

Although it is not difficult to deploy a single instance of Elasticsearch, creating a number of nodes and installing and configuring Elasticsearch on each one can be a time consuming and error-prone process. If you are considering running Elasticsearch on Azure VMs, you have two options that can help to reduce the chances of errors.
* Using the [Azure Resource Manager (ARM) template](http://azure.microsoft.com/documentation/templates/elasticsearch/) to build the cluster. This template is fully parameterized to enable you to specify the size and performance tier for the VMs that implement the nodes, the number of disks to use, and other common factors. The template can create a cluster based on Windows Server 2012 or Ubuntu Linux 14.0.4.
* Using scripts which can be automated or run unattended. Scripts that can create and deploy an Elasticsearch cluster are available on the [Azure Quickstart Templates](https://github.com/Azure/azure-quickstart-templates/blob/master/elasticsearch/elasticsearch-windows-install.ps1) site.

---

## Cluster and Node Sizing and Scalability Considerations

Elasticsearch enables a number of deployment topologies, designed to support differing requirements and levels of scale. This section discusses some common topologies, and describes the considerations for implementing clusters based on these topologies.

### Elasticsearch Topologies

Figure 4 illustrates a starting point for designing an Elasticsearch topology for the cloud based on Azure VMs.

![](figures/Elasticsearch/StartingPoint.png)
**Figure 4.**
Suggested starting point for building an Elasticsearch cluster with Azure

This topology comprises six data nodes together with three client nodes and three master nodes (only one master node is elected, the other two are available for election should the elected master fail.) Each node is implemented as a separate VM. Azure web applications are directed to client nodes via a load balancer. In this example, all nodes and the web applications reside in the same Azure VNET which effectively isolates them from the outside world. If the cluster needs to be available externally (possibly as part of a hybrid solution incorporating on-premises clients), then you can use the Azure Load Balancer to provide a public IP address, but you will need to take additional security precautions to prevent unauthorized access to the cluster. The optional "Jump Box" is a VM that is only available to administrators. This VM has a network connection to the Azure VNET, but also an outward facing network connection to permit administrator logon from an external network (this logon should be protected by using a strong password or certificate). An administrator can log on to the Jump Box, and then connect from there directly to any of the nodes in the cluster. Alternative approaches include using a site-to-site VPN between an organization and the VNET, or using [ExpressRoute](https://azure.microsoft.com/documentation/articles/expressroute-introduction/) circuits to connect to the VNET. These mechanisms permit administrative access to the cluster without exposing the cluster to the public internet.

To maintain VM availability, the data nodes are grouped into the same Azure availability set. Similarly, the client nodes are held in another availability set and the master nodes are stored in a third availability set.

This topology is relatively easy to scale out; simply add more nodes of the appropriate type and ensure that they are configured with the same cluster name in the elasticsearch.yml file. Client nodes also need to be added to the backend pool for the Azure load balancer.

#### Geo-locating Clusters

Don’t spread nodes in a cluster across regions as this can impact the performance of inter-node communication (see [Network Requirements](#network-requirements)). Geo-locating data close to users in different regions requires creating multiple clusters. In this situation, you need to consider how (or even whether) to synchronize clusters. Possible solutions include:

**Connecting to the cluster using [tribe nodes](https://www.elastic.co/blog/tribe-node).** A tribe node is similar to a client node except that it can participate in multiple Elasticsearch clusters and view them all as one big cluster. Data is still managed locally by each cluster (updates are not propagated across cluster boundaries), but all data is visible. A tribe node can query, create, and manage documents in any cluster. The primary restrictions are that a tribe node cannot be used to create a new index, and index names must be unique across all clusters. Therefore it is important that you consider how indexes will be named when you design clusters intended to be accessed from tribe nodes.

Using this mechanism, each cluster can contain the data that is most likely to be accessed by local client applications, but these clients can still access and modify remote data albeit with possible extended latency. Figure 5 shows an example of this topology. The tribe node in Cluster 1 is highlighted; the other clusters can also have tribe nodes although these are not shown on the diagram:

![](figures/Elasticsearch/TribeNode.png)
**Figure 5.**
A client application accessing multiple clusters through a tribe node

In this example, the client application connects to the tribe node in Cluster 1 (co-located in the same region), but this node is configured to be able to access Cluster 2 and Cluster 3 which might be located in different regions. The client application can send requests that retrieve or modify data in any of the clusters.

> **Note**: Tribe nodes require multicast discovery to connect to clusters, which may present a security concern. See the section [Node Discovery](#node-discovery) for more details.

*	Implementing geo-replication between clusters. In this approach, changes made at each cluster are propagated in near real-time to clusters located in other data centers. Third-party plugins are available for Elasticsearch that support this functionality, such as the [PubNub Changes Plugin](http://www.pubnub.com/blog/quick-start-realtime-geo-replication-for-elasticsearch/).
*	Using the [Elasticsearch Snapshot and Restore module](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-snapshots.html). If the data is very slow-moving and is modified only by a single cluster, you can consider using snapshots to take a periodic copy of the data and then restore these snapshots in other clusters (snapshots can be stored in Azure Blob Storage if you have installed the [Azure Cloud Plugin](https://github.com/elastic/elasticsearch-cloud-azure#azure-repository)). However, this solution does not work well for rapidly changing data or if data can be changed in more than one cluster.

---

#### Small-scale Topologies

Large-scale topologies comprising clusters of dedicated master, client, and data nodes might not be appropriate for every scenario. If you are building a small-scale production or development system, consider the 3-node cluster shown in Figure 6. Client applications connect directly to any available data node in the cluster. The cluster contains three shards labelled P1-P3 (to allow for growth) plus replicas labelled R1-R3. Using three nodes allows Elasticsearch to distribute the shards and replicas so that if any single node fails no data will be lost.

![](figures/Elasticsearch/ThreeNodeCluster.png)
**Figure 6.**
A 3-node cluster with 3 shards and replicas

If you are running a development installation on a standalone machine you can configure a cluster with a single node that acts as master, client, and data storage. Alternatively, you can start multiple nodes running as a cluster on the same computer by starting more than one instance of Elasticsearch. Figure 7 shows an example.

![](figures/DevelopmentConfiguration.png)
**Figure 7.**
A development configuration running multiple Elasticsearch nodes on the same machine

Note that neither of these stand-alone configurations are recommended for a production environment as they can cause contention unless your development machine has a significant amount of memory and several fast disks. Additionally, they do not provide any high availability guarantees; if the machine fails, all nodes are lost.

### Considerations for Scaling a Cluster and Data Nodes

Elasticsearch can scale in two dimensions: vertically (using bigger, more powerful machines) and horizontally (spreading the load across machines).

#### Scaling Elasticsearch Data Nodes Vertically

If you are hosting an Elasticsearch cluster by using Azure VMs, each node can correspond to a VM. The limit of vertical scalability for a node is largely governed by the SKU of the VM and the overall restrictions applied to individual storage accounts and Azure subscriptions. The page [Azure Subscription and Service Limits, Quotas, and Constraints](https://azure.microsoft.com/documentation/articles/azure-subscription-service-limits/) describes these limits in detail, but as far as building an Elasticsearch cluster is concerned, the items in the following list are the most pertinent. Additionally, you should probably not consider using VMs with more than 64GB of memory without good cause; as described in the section [Memory Requirements](#memory-requirements), you should not allocate more than 30GB of RAM on each VM to the JVM and allow the operating system to utilize the remaining memory for I/O buffering:
*	Each storage account is restricted to 20,000 IOPS; using the same storage account to hold a number of VHDs can limit the performance of those VHDs.
*	The number of data nodes in a VNET. If you are not using the Azure Resource Manager (ARM), there is a limit of 2048 VM instances per VNET. While this should prove sufficient for many cases, if you have a very large configuration with thousands of nodes this could be a limitation.
*	Number of storage accounts per subscription per region. You can create up to 100 storage accounts per Azure subscription in each region. Storage accounts are used to hold virtual disks, and each storage account has a limit of 500TB of space.
*	Number of cores per subscription. The default limit is 20 cores per subscription, but this can be increased by Microsoft up to 10,000 cores. Remember that some VM sizes (A9, A11, D14, and DS14) can contain 16 cores, while a G5 VM has 32 cores.
*	The amount of memory per VM size. Smaller size VMs have limited amounts of memory available (D1 machines have 3.5GB, and D2 machines have 7GB). These machines might not be suitable for scenarios that require Elasticsearch to cache significant amounts of data to achieve good performance (aggregating data, or analyzing a large number of documents during data ingestion, for example).
*	The maximum number of disks per VM size. This restriction can limit the size and performance of a cluster. Fewer disks means that less data can be held, and performance can be reduced by having fewer disks available for striping.
*	The number of update domains / fault domains per availability set. If you create VMs using the ARM, each availability set can be allocated up to 3 fault domains and 20 update domains. This limitation can impact the resilience of a large cluster that is subjected to frequent rolling updates.

---

With these restrictions in mind, you should always spread the virtual disks for the VMs in a cluster across storage accounts to reduce the chances of I/O throttling. In a very large cluster, you may need to redesign your logical infrastructure and split it into separate functional partitions. For example, you might need to split the cluster across subscriptions, although this process can lead to further complications because of the need to connect VNETs.

>	**Note**: Be aware that with Azure, storage accounts are pinned to a specific storage stamp. This is an internal mechanism used to maintain consistency and availability. The paper [A Highly Available Cloud Storage Service with Strong Consistency](http://blogs.msdn.com/b/windowsazurestorage/archive/2011/11/20/windows-azure-storage-a-highly-available-cloud-storage-service-with-strong-consistency.aspx) provides more details on how this works. If you have a storage outage loca]lized to a specific stamp, you will get errors on all drives created using that account. When this occurs, any VMs using these drives may fail. Using multiple storage accounts to host the different drives for a VM can therefore increase the risk of failure for that VM. For this reason, it is recommended that you use a single storage account per node, and store the system drive and all data drives in that account.

#### Scaling an Elasticsearch Cluster Horizontally

Internally within Elasticsearch, the limit of horizontal scalability is determined by the number of shards defined for each index. Initially, many shards can be allocated to the same node in a cluster, but as the volume of data grows additional nodes can be added and shards can be distributed across these nodes. In theory, only when the number of nodes reaches the number of shards will the system cease to scale horizontally.

As with vertical scaling, there are some issues that you should consider when contemplating implementing horizontal scaling, including:

*	The maximum number of VMs that you can connect in an Azure VNET. This can limit of horizontal scalability for a very large cluster. You can create a cluster of nodes that spans more than one VNET to circumvent this limit, but this approach can lead to reduced performance due to the lack of locality of each node with its peers.
*	The number of disks per VM Size. Different series and SKUs support different numbers of attached disks. Additionally, you can also consider using the ephemeral storage included with the VM to provide a limit amount of faster data storage, although there are resiliency and recovery implications that you should consider (see the document Configuring, Testing, and Analyzing Elasticsearch Resilience and Recovery for more information). The D-series, DS-series, Dv2-series, and GS-series of VMs use SSDs for ephemeral storage.
*	The maximum IOPS per virtual disk. Regular attached disks created by using standard storage are limited to 500 IOPS per disks. Attached SSDs can support up to 5000 IOPS per disk. To use SSDs for attached disks (not ephemeral storage) you must create a VM that supports premium storage (DS-series or GS-series machines).

You could consider using Azure autoscaling to start and stop VMs as demands dictates. However, this approach might not be appropriate for an Elasticsearch cluster for the following reasons:

*	Auto-scaling is best suited for stateless VMs. Each time you add or remove a node from an Elasticsearch cluster, shards are reallocated to balance the load, and this process can generate considerable volumes of network traffic and disk I/O and can severely impact data ingestion rates. You must assess whether this overhead is worth the benefit of the additional processing and memory resources that become available by dynamically adding more VMs.
*	Auto-scaling does not happen instantaneously, and it may take several minutes before additional VMs become available or they are shut down. Auto-scaling should only be used to handle sustained changes in demand.
*	Auto-scaling for VMs is implemented per availability set. You must define your availability sets carefully to prevent scaling back from removing nodes that contain a shard and all of its replicas at the same time.
*	After scaling out, do you actually need to consider scaling back? Many Elasticsearch implementations grow over time, but the nature of the data is such that it tends not to shrink in volume. It is possible to delete documents, and documents can be configured with a TTL (time to live) after which they expire and get removed, but in most cases it is likely that the space previously allocated will be quickly reused by new or modified documents. Fragmentation within an index might occur when documents are removed or changed, in which case you can use the Elasticsearch HTTP [Optimize](https://www.elastic.co/guide/en/elasticsearch/reference/1.7/indices-optimize.html) API (Elasticsearch 2.0.0 and earlier) or the [Force Merge](https://www.elastic.co/guide/en/elasticsearch/reference/2.1/indices-forcemerge.html) API (Elasticsearch 2.1.0 and later) to perform defragmentation.

> **Note**: Optimization is a very costly operation, and you should not perform it on an index that is highly active. However, it is a good idea to optimize an index that is quiescent because this will help to reduce the resources required to perform searching.

---

### Determining the Number of Shards for an Index

The number of nodes in a cluster can vary over time, but the number of shards in an index is fixed once the index has been created. To add or remove shards requires reindexing the data – a process of creating a new index with the required number of shards and then copying the data from the old index to the new (you can use aliases to insulate users from the fact that data has been reindexed – see the document Maximizing Data Aggregation and Query Performance with Elasticsearch on Azure for more details). Therefore, it is important to determine the number of shards that you are likely to require in advance of creating the first index in your cluster. You can perform the following steps to establish this number:

1. Create a single-node cluster using the same hardware configuration that you intend to deploy in production.
2. Create an index that matches the structure that you plan to use in production. Give this index a single shard and no replicas.
3. Add a specific quantity of realistic production data to the index.
4. Perform typical queries, aggregations, and other workloads against the index and measure the throughput and response time.
5. If the throughput and response time are within acceptable limits, then repeat the process from step 3 (add more data).
6. When you appear to have reached the capacity of the shard (response times and throughput start becoming unacceptable), make a note of the volume of documents.
7. Extrapolate from the capacity of a single shard to the anticipated number of documents in production to calculate the required number of shards (you should include some margin of error in these calculations as extrapolation is not a precise science).

> **Note**: Remember that each shard is implemented as a Lucene index that consumes memory, CPU power and file handles. The more shards you have, the more of these resources you will require.
Additionally, creating more shards may increase scalability (depending on your workloads and scenario) and can increase data ingestion throughput, but it might reduce the efficiency of many queries. By default, a query will interrogate every shard used by an index (you can use [custom routing](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-routing-field.html) to modify this behavior if you know on which shards the data you require is located).

Following this process can only generate an estimate for the number of shards, and the volume of documents expected in production might not be known. In this case, you should determine the initial volume (as above) and the predicted growth rate. Create an appropriate number of shards that can handle the growth of data for the period until you are willing to reindex the database. Other strategies used for scenarios such as event management and logging include using rolling indexes; create a new index for the data ingested each day and access this index through an alias that is switched daily to point to the most recent index. This approach enables you to more easily age-out old data (you can delete indexes with information that is no longer required) and keeps the volume of data manageable.

Bear in mind that the number of nodes does not have to match the number of shards. For example, if you create 50 shards, you can spread them across 10 nodes initially, and then add more nodes to scale the system out as the volume of work increases. However, avoid creating an exceptionally large number of shards on a small number of nodes (1000 shards spread across 2 nodes, for example). Although the system could theoretically scale to 1000 nodes with this configuration, running 500 shards on a single node risks crippling the performance of the node.

> **Note**: For systems that are data-ingestion heavy, consider using a prime number of shards. The default algorithm that Elasticsearch uses for routing documents to shards produces a more even spread in this case.

### Security Considerations

By default, Elasticsearch implements minimal security and does not provide any means of authentication and authorization. These aspects require configuring the underlying operating system and network, and using plugins and third-party utilities. Examples include [Shield](https://www.elastic.co/products/shield), and [Search Guard](https://github.com/floragunncom/search-guard).

> **Note**: Shield is a plugin provided by Elastic for user authentication, data encryption, role-based access control, IP filtering, and auditing. It may be necessary to configure the underlying operating system to implement further security measures, such as disk encryption.

In a production system, you should consider how to:
*	Prevent unauthorized access to the cluster.
*	Identify and authenticate users.
*	Authorize the operations that authenticated users can perform.
*	Protect the cluster from rogue or damaging operations.
*	Protect the data from unauthorized access.
* Meet regulatory requirements for commercial data security (if appropriate).

---

### Securing Access to the Cluster

Elasticsearch is a network service. The nodes in an Elasticsearch cluster listen for incoming client requests using HTTP, and communicate with each other using a TCP channel. You should take steps to prevent unauthorized clients or services from being able to send requests over both the HTTP and TCP paths. Consider the following items:
*	Define network security groups to limit the inbound and outbound network traffic for a VNET or VM to specific ports only.
*	Change the default ports used for client web access (9200) and programmatic network access (9300). Use a firewall to protect each node from malicious Internet traffic.
*	Depending on the location and connectivity of clients, place the cluster on a private subnet with no direct access to the Internet. If the cluster must be exposed outside the subnet, route all requests through a bastion server or proxy sufficiently hardened to protect the cluster.
*	If you must provide direct access to nodes, use the Elasticsearch Jetty plugin to provide SSL connectivity, authentication, and connection logging. Alternatively, configure an nginx proxy server and configure HTTPS authentication.
> **Note**: Using a proxy server such as nginx, you can also restrict access to functionality. For example, you can configure nginx to only allow requests to the \_search endpoint if you need to prevent clients from performing other operations.

*	If you require more comprehensive network access security, use the Shield or Search Guard plugins.

---

### Identifying and Authenticating Users

All requests made by clients to the cluster should be authenticated. Additionally, you should prevent unauthorized nodes from joining the cluster as these can provide a backdoor into the system that bypass authentication.

Elasticsearch plugins are available that can perform different types of authentication, including:
*	HTTP Basic Authentication. Usernames and passwords are included in each. All requests should be encrypted by using SSL/TLS or an equivalent level of protection.
*	LDAP and Active Directory integration. This approach requires that clients are assigned roles in LDAP or AD groups.
*	Native authentication by using identities defined within the Elasticsearch cluster itself.
*	TLS authentication within a cluster to authenticate all nodes.
*	IP filtering, to prevent clients from unauthorized subnets from connecting, and also preventing nodes from these subnets joining the cluster.

---

### Authorizing Client Requests

Authorization depends on the Elasticsearch plugin used to provide this service. For example, a plugin that provides Basic Authentication typically provides features that define the level of authentication, whereas a plugin that uses LDAP or AD will typically associate clients with roles, and then assign access rights to those roles. When using any given plugin, you should consider the following points:
*	Do you need to restrict the operations that a client can perform? For example, should a client be able to monitor the status of the cluster, or create and delete indexes?
*	Should the client be restricted to specific indexes? This is useful in a multi-tenant situation where tenants may be assigned their own specific set of indexes, and these indexes should be inaccessible to other tenants.
*	Should the client by able to read and write data to an index? A client may be able to perform searches that retrieve data using an index but must be prevented from adding or deleting data from that index, for example.

---

Currently, most security plugins scope operations to the cluster or index level, and not to subsets of documents within indexes. This is for efficiency reasons. It is therefore not easy to limit requests to specific documents within a single index. If you require this level of granularity, save documents in separate indexes and use aliases that group indexes together. For example, in a personnel system, if user A requires access to all documents that contain information about employees in department X, user B requires access to all documents that contain information about employees in department Y, and user C requires access to all documents that contain information about employees in both departments, create two indexes (for department X and department Y), and an alias that references both indexes. Grant user A read access to the first index, grant user B read access to the second index, and grant user C read access to both indexes through the alias. For more information, see [Faking Index per User with Aliases](https://www.elastic.co/guide/en/elasticsearch/guide/current/faking-it.html).

### Protecting the Cluster

The cluster can become vulnerable to misuse if it is not protected carefully, as outlined by the following points:
*	Disable dynamic query scripting in Elasticsearch queries as they can lead to security vulnerabilities. Use native scripts in preference to query scripting; a native script is an Elasticsearch plugin written in Java and compiled into a JAR file.
> **Note**: Dynamic query scripting is now disabled by default; do not re-enable it unless you have a very good reason to do so.

*	Avoid exposing query-string searches to users. Query-string searching allows users to perform resource-intensive queries unhindered. These searches could severely impact the performance of the cluster and can render the system open to DOS attacks. Additionally, query-string searching can expose potentially private information.
*	Prevent operations from consuming a lot of memory as these can cause out-of-memory exceptions resulting in Elasticsearch failing on a node. Long-running resource intensive operations can also be used to implement DOS attacks. Examples include:
  *	Search requests that attempt to load very large fields into memory (if a query sorts, scripts, or facets on these fields).
> **Note**: Using [Doc Values](https://www.elastic.co/guide/en/elasticsearch/guide/current/doc-values.html) can reduce the memory requirements of indexes by saving fielddata to disk rather than loading it into memory. This can help to reduce the chances of memory exhaustion on a node but with a reduction in speed.

  * Searches that query multiple indexes at the same time.
  *	Searches that retrieve a large number of fields. These searches can exhaust memory by causing a vast amount of field data to be cached. By default, the field data cache is unlimited in size, but you can set the [indices.fielddata.cache.*](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-fielddata.html) properties in the elasticsearch.yml configuration file to limit the resources available. You can also configure the [field data circuit breaker](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-fielddata.html#fielddata-circuit-breaker) to help prevent the cached data from a single field from exhausting memory, and the [request circuit breaker](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-fielddata.html#request-circuit-breaker) to stop individual queries from monopolizing memory. The cost of setting these parameters is the increased likelihood of some queries failing or timing out.
> **Note**: Elasticsearch always assumes that it has enough memory to perform its current workload. If this is not the case, then the Elasticsearch service can crash. Elasticsearch provides endpoints that return information about resource usage (the HTTP [cat APIs](https://www.elastic.co/guide/en/elasticsearch/reference/1.7/cat.html)), and you should monitor this information carefully.

  * Waiting for too long to flush an in-progress memory segment. This can exhaust in-memory buffer space. If necessary, [configure the translog](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-translog.html) to reduce the thresholds at which data is flushed to disk.

  * Creating indexes with large amounts of metadata. An index that contains documents with a large variation in field names can consume a lot of memory. For more information, see [Mapping Explosion](https://www.elastic.co/blog/found-crash-elasticsearch#mapping-explosion).
> **Note**: The definition of a long-running or query intensive operation is highly scenario-specific. The workload typically expected by one cluster might have a completely different profile from the workload on another. Determining which operations are unacceptable requires significant research and testing of your applications.

---

Be proactive; detect and stop malicious activities before they cause significant damage or data loss. Consider using a security monitoring and notification system that can quickly detect unusual patterns of data access and raise alerts when, for example, user login requests fail, unexpected nodes join or leave the cluster, or operations are taking longer than expected. Tools that can perform these tasks include Elasticearch [Watcher](https://www.elastic.co/products/watcher).

### Protecting the Data

You can protect data inflight by using SSL/TLS, but Elasticsearch does not provide any built-in form of data encryption for information that is stored on disk. Remember that this information is held in ordinary disk files, and any user with access to these files may be able to compromise the data that they hold, for example by copying them to their own cluster. Consider the following points:
*	Protect the files used by Elasticsearch to hold the data. Do not allow arbitrary read or write access to identities other than the Elasticsearch service.

*	Encrypt the data held in these files by using an encrypting file system.
> **Note**: Azure now supports disk encryption for Linux and Windows VMs. For more information, see [Azure Disk Encryption for Windows and Linux IaaS VMs Preview](https://azure.microsoft.com/documentation/articles/azure-security-disk-encryption/).

---

### Meeting Regulatory Requirements

Regulatory requirements are primarily concerned with auditing operations to maintain a history of events, and ensuring the privacy of these operations to help prevent them being monitored (and replayed) by an external agency. In particular, you should consider how to:
*	Track all requests (successful or not), and all attempts to access the system.
*	Encrypt communications made by clients to the cluster as well as node-to-node communications performed by the cluster. You should implement SSL/TLS for all cluster communications. Elasticsearch also supports pluggable ciphers if your organization has requirements distinct from those available through SSL/TLS.
*	Store all audit data securely. The volume of audit information can grow very rapidly and must be protected robustly to prevent tampering of audit information.
*	Safely archive audit data.

### Monitoring Considerations

Monitoring is important both at the operating system level and at the Elasticsearch level.

You can perform monitoring at the operating system level using operating-system specific tools. Under Windows, this includes items such as Performance Monitor with the appropriate performance counters, while under Linux you can use tools such as *vmstat*, *iostat*, and *top*. The key items to monitor at the operating system level include CPU utilization, disk I/O volumes, disk I/O wait times, network traffic. In a well-tuned Elasticsearch cluster, CPU utilization by the Elasticsearch process should be high, and disk I/O wait times should be minimal.

At the software level, you should monitor the throughput and response times of requests, together with the details of requests that fail. Elasticsearch provides a number of APIs that you can use to examine the performance of different aspects of a cluster. The two most important APIs are *\_cluster/health* and *\_nodes/stats*. The \_cluster/health API can be used to provide a snapshot of the overall health of the cluster, as well as providing detailed information for each index, as shown in the following example:

`GET _cluster/health?level=indices`

The example output shown below was generated by using this API:

```
{
	  "cluster_name":"elasticsearch",
	  "status":"green",
	  "timed_out":false,
	  "number_of_nodes":6,
	  "number_of_data_nodes":3,
	  "active_primary_shards":10,
	  "active_shards":20,"
	  relocating_shards":0,
	  "initializing_shards":0,
	  "unassigned_shards":0,
	  "delayed_unassigned_shards":0,"
	  "number_of_pending_tasks":0,
	  "number_of_in_flight_fetch":0,
	  "indices":{
	    "systwo":{
	      "status":"green",
	      "number_of_shards":5,
	      "number_of_replicas":1,
	      "active_primary_shards":5,
	      "active_shards":10,
	      "relocating_shards":0,
	      "initializing_shards":0,
	      "unassigned_shards":0
	    },
	    "sysfour":{
	      "status":"green",
	      "number_of_shards":5,
	      "number_of_replicas":1,
	      "active_primary_shards":5,
	      "active_shards":10,
	      "relocating_shards":0,
	      "initializing_shards":0,
	      "unassigned_shards":0
	    }
	  }
}
```

This cluster contains two indexes named *systwo* and *sysfour*. Key statistics to monitor for each index are the status, active_shards, and unassigned_shards. The status should be green, the number of active_shards should reflect the number_of_shards and number_of_replicas, and unassigned_shards should be zero. If the status is "red", then part of the index is missing or has become corrupt. You can verify this if the active_shards setting is less than number_of_shards * (number_of_replicas + 1) and unassigned_shards is non-zero. Note that a status of yellow indicates that an index is in a transitional state, either as the result of adding more replicas or shards being relocated. The status should switch to green when the transition has completed. If it stays yellow for an extended period or changes to red, you should look to see whether any significant I/O events (such as a disk or network failure) have occurred at the operating system level.
The \_nodes/stats API emits extensive information about each node in the cluster:

`GET _nodes/stats`

The output generated includes details about how indexes are stored on each node (including the sizes and numbers of documents), time spent performing indexing, querying, searching, and merging, caching, operating system and process information, statistics about the JVM (including garbage collection performance), and thread pools. For more information, see [Monitoring Individual Nodes](https://www.elastic.co/guide/en/elasticsearch/guide/current/_monitoring_individual_nodes.html#_monitoring_individual_nodes).

If a significant proportion of Elasticsearch requests are failing with *EsRejectedExecutionException* error messages, then Elasticsearch is failing to keep up with the work being sent its way. In this situation, you need to identify the bottleneck that is causing Elasticsearch to fall behind. Consider the following items:
*	If the bottleneck is due to a resource constraint, such as insufficient memory allocated to the JVM causing an excessive number of garbage collections, then consider allocating additional resources (in this case , configure the JVM to use more memory, up to 50% of the available storage on the node – see [Memory Requirements](#memory-requirements)).
*	If the cluster is showing large I/O wait times and the merge statistics gathered for an index by using the \_node/stats API contain large values then the index is write-heavy. Revisit the points raised in the section [Optimizing Resources for Indexing Operations](Optimizing Resources for Indexing Operations) to tune indexing performance.
*	Throttle client applications that are performing data ingestion operations and determine the effect that this has on performance. If this approach shows significant improvement, then consider either retaining the throttle, or scaling out by spreading the load for write-heavy indexes across more nodes. For more information, see the document Maximizing Data Ingestion Performance with Elasticsearch on Azure.
*	If the searching statistics for an index indicate that queries are taking a long time then consider how the queries are optimized. For more information, see the section [Query Tuning](#query-tuning). Note that you can use the *query_time_in_millis* and *query_total* values reported by the search statistics to calculate a rough guide to query efficiency; the equation *query_time_in_millis* / *query_total* will give you an average time for each query.

---

### Tools for Monitoring Elasticsearch

A variety of tools are available for performing everyday monitoring of Elasticsearch in production. These tools typically use the underlying Elasticsearch APIs to gather information and present the details in a manner that is easier to observe than the raw data. Common examples include [Elasticsearch-Head](https://mobz.github.io/elasticsearch-head/), [Bigdesk](http://bigdesk.org/), [Kopf](https://github.com/lmenezes/elasticsearch-kopf), and [Marvel](https://www.elastic.co/products/marvel).

Elasticsearch-Head, Bigdesk, and Kopf run as plugins for the Elasticsearch software. More recent versions of Marvel can run independently, but require [Kibana](https://www.elastic.co/downloads/kibana) to provide a data capture and hosting environment. The advantage of using Marvel with Kibana is that you can implement monitoring in a separate environment from the Elasticsearch cluster, enabling you to explore problems with Elasticsearch that might not be possible if the monitoring tools run as part of the Elasticsearch software. For example, if Elasticsearch repeatedly fails or is running very slowly, tools that run as Elasticsearch plugins will also be effected, making monitoring and diagnosis more difficult.

At the operating system level, you can use suites such as [New Relic Servers](https://docs.newrelic.com/docs/servers) or [Azure Diagnostics with the Azure Portal](https://azure.microsoft.com/en-us/blog/windows-azure-virtual-machine-monitoring-with-wad-extension/) to capture performance data for VMs hosting Elasticsearch nodes. Another approach is to use [Logstash](https://www.elastic.co/products/logstash) to capture performance and log data, store this information in a separate Elasticsearch cluster (don't use the same cluster that you are using for your application), and then use Kibana to visualize the data. For more information, see [Microsoft Azure Diagnostics with ELK](https://github.com/mspnp/semantic-logging/tree/elk).

### Tools for Testing Elasticsearch Performance

Other tools are available if you are benchmarking Elasticsearch or subjecting a cluster to performance testing. These tools are intended to be used in a development or test environment rather than production. A frequently-used example is [Apache JMeter](http://jmeter.apache.org/).

JMeter was used to perform benchmarking and other load tests described in documents related to this guidance. The document Running Performance Tests on Elasticsearch Using JMeter describes in detail how JMeter was configured and used.

### Resources
*	[Elasticsearch](https://www.elastic.co/products/elasticsearch)
*	[Elasticsearch: The Definitive Guide](https://www.elastic.co/guide/en/elasticsearch/guide/master/index.html)
*	[Sizes for Virtual Machines](https://azure.microsoft.com/documentation/articles/virtual-machines-size-specs/)
*	[Install Elasticsearch cluster on Virtual Machines](https://azure.microsoft.com/documentation/templates/elasticsearch/)
*	[Elasticsearch Java API](https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/index.html)
*	[Shield](https://www.elastic.co/products/shield)
*	[Search Guard](https://github.com/floragunncom/search-guard)
*	[Elasticsearch-Head](https://mobz.github.io/elasticsearch-head/)
*	[Bigdesk](http://bigdesk.org/)
*	[Kopf](https://github.com/lmenezes/elasticsearch-kopf)
*	[Marvel](https://www.elastic.co/products/marvel)
*	[Kibana](https://www.elastic.co/downloads/kibana)
*	[Watcher](https://www.elastic.co/products/watcher)
*	[New Relic Servers](https://docs.newrelic.com/docs/servers)
*	[Apache JMeter](http://jmeter.apache.org/)

---