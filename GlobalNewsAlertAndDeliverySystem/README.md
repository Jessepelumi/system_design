## Global News Alert And Delivery System

### Requirements Gathering

1. **News Ordering:**


I assume we require strict reverse chronological order for standard feeds so traders see the newest market information first. However, do we have an Urgency/Flash Tier that needs to override the standard order and pin critical updates to the top of the terminal screen?

2. **Media:**


Are alerts strictly text headlines, or do they support rich media like charts, images, and video clips? If media is supported, I'll separate the heavy binary payloads from our real-time messaging data path to prevent network blocking and avoid impacting low-latency headline delivery.

3. **Traffic Volume (Scale):**


What is our scale or traffic volume? I'm assuming write traffic from journalists is relatively low (e.g., 10-50 stories per second), but our fan-out delivery traffic is massive and spiky—potentially faning out to millions of concurrent terminal connections worldwide during market opens.

4. **Important Features:**


Beyond simple headline distribution, are we responsible for user-defined keyword filtering (e.g., users subscribing to specific stock tickers), and do we need to support out-of-band delivery channels like SMS or email alerts alongside the terminal sockets?

5. **Latency Targets:**


What is our end-to-end latency budget? I am targeting a strict sub-100ms delivery window from the moment a journalist hits 'Publish' to the millisecond it renders on global client terminals. Is that constraint accurate?

### Phase 1: Ingestion & Buffering

When content originates from Bloomberg journalists or external wire feeds, it hits our Ingestion tier via a Load Balancer and stateless Web Servers. The Ingestion Service instantly verifies news item uniqueness via a cryptographic token (Idempotency Hash) lookup against a Redis Cache to guarantee message deduplication at the gateway. If unique, it drops the raw payload into our first Distributed Message Queue (Kafka) and returns a 202 acknowledgment.


**Note:** An idempotency hash is a value used to ensure that the same request processed multiple times produces only one final effect.

```JSON
{
  "headline": "Fed raises interest rates",
  "idempotency_key": "abc123"
}
```


Asynchronously, dedicated consumer workers pull from this Kafka topic: the Storage Processing Service commits metadata to our NoSQL Database, and the Media Processing Service extracts binary assets to dump into AWS S3 where they are cached globally at edge locations by a CDN.


**Note:** We use a NoSQL database because our workload is heavily write-optimized and horizontally scalable. The system primarily performs simple access patterns like fetching recent news by topic, ticker, or timestamp rather than complex relational joins or transactional queries. Since we don't require strong relational consistency or complex multi-table operations, a distributed NoSQL store is a better fit for high-throughput ingestion and global-scale fan-out. We trade away complex joins and some transactional flexibility in exchange for horizontal scalability, partition tolerance, and predictable high-volume writes.

### Phase 2: Regional Isolation & Delivery

Simultaneously, the first-stage Fanout Service passes the alert down a second, regionalized Kafka cluster. This decouples downstream client delivery mechanics entirely from our ingestion ingestion tier.


Fanout Workers pick up the message from the queue and consult our Redis Session Registry to see which terminal users are online and exactly which machine they are connected to. The worker then routes the lightweight alert payload directly to those targeted WebSocket Servers.


Because our Terminal Users maintain long-lived, active connections with these stateful WebSocket nodes, the nodes instantly push the alert down the open socket in sub-100ms. If a user has just logged on or reconnected after being offline, the WebSocket server will proactively query our News Feed Cache (a Redis Sorted Set) to catch them up on historical headlines before merging them into the active live stream.


![Global News Alert And Delivery System](GlobalNewsAlertAndDeliverySystem.png)


### Engineering Details

### Q1: "How do you handle failures in the asynchronous workers?"


If the `Storage Processing Service` crashes while trying to write a news story to the NoSQL database, how do you prevent that message from being lost forever?


**Answer:** Because we use Kafka, we don't lose data. We will utilize a Retry Topic and a Dead Letter Queue (DLQ) pattern. If a worker encounters a transient error, it will write the message to a retry topic to try again after a backoff period. If it repeatedly fails, the message moves to a DLQ for manual auditing, ensuring we maintain strict fault-tolerance without blocking the main Kafka partition.


### Q2: "How do you scale Kafka to match Bloomberg's global throughput?"


How do we ensure that messages don't pile up in Kafka during an intense market event?


**Answer:** We will partition our Kafka topics. For news alerts, we can partition by a combination of the `urgency_tier` and a hash of the `article_id`. This allows us to run multiple instances of our processing services concurrently in a Consumer Group, with each consumer handling a specific partition to process messages in parallel.


### Q3: "What happens if a specific WebSocket server node crashes with 50,000 active connections?"


**Answer:** The 50,000 terminals will detect the dropped socket connection and instantly attempt to reconnect. Our client-side load balancer will distribute those reconnection requests across our remaining active healthy WebSocket nodes. As users authenticate onto their new nodes, those nodes will update the Redis Session Registry, seamlessly shifting their routing paths for future alerts.


### Q4: "How do you handle a massive news flash that applies to millions of users at the exact same time without crashing the WebSocket servers (The Thundering Herd / Connection Fanout problem)?"


**Answer:** We keep our real-time payload incredibly lean—just the string text of the headline, unique identifier, and urgency score. We completely omit massive body paragraphs or media assets from the initial socket transmission. By keeping the packet sizes down to a few hundred bytes, our WebSocket servers can push out millions of concurrent messages across open TCP sockets without hitting internal network cards or memory buffering bounds.
