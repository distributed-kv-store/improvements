# improvements
A roadmap to make the distributed core fault-tolerant

## Authors
- Tirthraj Parmar
- Manthan Thakar

# Consistent Hashing
## Overall Approach (Fetch and Set)
* The idea is to create servers as a point on a circle. For uniform distribution we also create certain number of virtual servers on the circle which are in-turn map to a server. 
* We take `N = 200` such that each server will have 200 points on the circle. Therefore, for total `S` servers, we have `S * N` points on the circle.  
* Each point is a  `hash(n * s)`, such that `n \in [1,N]` and `s \in [1,S]`.
* We maintain a Map of a point of virtual server to its corresponding actual server for all points. And a sorted list of all points (`S * N`).
* For a key `k` we take `hash(k)` as a point on the circle. Then we get its next closes server in clockwise direction by getting first value from sorted array which is greater than or equal to `hash(k)`. Then we get corresponding actual server from the virtual server point. And that is how a key is mapped to a server.

## Addition of a new server
* When a new server arrives, we first calculate all N virtual server points on the circle for the server. Then add them in map of virtual server to actual server and sorted list.
* To reorganize the data by keys, we take next server in clockwise direction of each new virtual server and rehash all its keys using modified map and sorted array and redistribute data accordingly.

## Deletion of a server
* Deletion of a server will require to remove all the virtual server points for that server from map and sorted list, get data of all keys residing in that server from its replica and rehash them to redistribute the data based on keys.

# Fault-Tolerance
- As described in the previous section, we achieve partition tolerance by employing Consistent Hashing. 
- In this section, we describe the design for fault-tolerance that we employ. Note that, we have chosen availability over consistency in our design, hence all the updates to nodes are eventually consistent.
- This design is more suitable for applications that require fast reads since we opt for weak consistency.
#### Availability
- To achieve high-availability in the face of failures, we replicate keys to N nodes, where N is the `REPLICATION FACTOR` configured for the proxy.
- **Replication Policy**
	- Replication policy is to choose `N-1` successor nodes on the ring.
- **Known Limitations**
	- One of the limitations of choosing `N-1` successors in the rings as replicas is that it doesn’t take into account the physical locations of the nodes. In some cases, it would be more beneficial to choose nodes in the same datacenter or rack as replicas.
- **Load Balancing**
- Since there exists multiple replicas of the same node, we allow for queries that can be made to any of the redundant copy.
- Note that since we opt for highly available design, the nodes might read outdated data from nodes.
- For balancing workloads across replicas, proxy randomly picks any one of the replicas for a given key and redirects the request to that node. 
#### Consistency
- For consistency across replicas, we use the approach of eventual consistency in lieu of Availability at all times.
- Every time there’s a new write to any of the nodes in the system, we instantiate a background process that synchronizes all the replicas.
#### Single Point of Failure
- In our design, there’s an obvious single point of failure, Proxy. Since, the proxy is responsible for keeping track of servers and insertion of keys. Therefore, if the proxy server is down, the system becomes ineffective.
- There are two approaches to overcome this problem:
- **Replicate Proxy Server**
- One approach is to replicate the proxy server and have multiple copies of proxy server. 
- If the proxy server goes down, we can elect any one of the replicas to become the primary proxy server.
- **Make Node aware of other nodes in the ring**
- The other approach would be to eliminate the concept of proxy from the design and make all the server nodes aware of other nodes.
- This way, as long as the nodes are replicated, we can have multiple entry points and hence there is no single point of failure.
### Local Storage
- Although the nodes store the key-value pairs in memory, it’s entirely possible that the node goes down momentarily and comes back up or it’s still available but for some reason has lost its copy.
- To prevent this, we could checkpoint the in-memory database periodically to disk. 
