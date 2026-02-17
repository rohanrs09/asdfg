# 💻 SYSTEM DESIGN PRACTICAL GUIDE
## Implementation, Interview Strategies & Code Examples

---

## PART 1: HOW TO APPROACH A SYSTEM DESIGN INTERVIEW

### The 45-Minute Interview Breakdown

```
Total Time: 45 minutes

0:00-5:00   → Clarify Requirements (5 min)
5:00-15:00  → High-Level Design (10 min)
15:00-30:00 → Deep Dive (15 min)
30:00-40:00 → Scaling & Trade-offs (10 min)
40:00-45:00 → Follow-up Questions (5 min)

GOLDEN RULE: Start broad, go deep on feedback
```

### Step 1: Ask Clarifying Questions (CRITICAL!)

**What to Ask** (2-3 minutes):

```
FUNCTIONAL REQUIREMENTS:
- "What are the main features?"
- "Do we need real-time updates?"
- "What about user authentication?"
- "Any search functionality?"

NON-FUNCTIONAL REQUIREMENTS:
- "How many users? Total or daily active?"
- "What's the read-to-write ratio?"
- "Expected latency requirements?"
- "Should we be globally distributed?"
- "What about consistency? Real-time or eventual is OK?"

CONSTRAINTS:
- "Any specific technology requirements?"
- "Deployment environment? Cloud or on-premises?"
- "Cost constraints?"
- "Compliance/security requirements?"

SCALE:
- "100K users or 100M users?"
- "GB of data or TB?"
- "Requests/second expected?"
```

**Why This Matters**:
- Shows you think before designing
- Prevents designing for wrong scale
- Interviewer understands your assumptions
- Demonstrates problem-solving maturity

**Good vs Bad Approach**:
```
❌ BAD: Start drawing immediately
✅ GOOD: "So let me clarify - you need real-time 
         updates for 100M users with strong 
         consistency? That changes the design significantly.
         Can I confirm XYZ assumptions?"
```

---

### Step 2: High-Level Design (Draw Early!)

**What to Draw** (10 minutes):

```
BASIC ARCHITECTURE:
┌─────────────────────────────────────────┐
│           CLIENT/BROWSER                │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│         LOAD BALANCER (LB)              │
└──────────────────┬──────────────────────┘
                   │
    ┌──────────────┼──────────────────┐
    │              │                  │
┌───▼──┐       ┌───▼──┐          ┌───▼──┐
│Web   │       │Web   │   ...    │Web   │
│Srv 1 │       │Srv 2 │          │Srv N │
└───┬──┘       └───┬──┘          └───┬──┘
    │              │                  │
    └──────────────┼──────────────────┘
                   │
         ┌─────────▼──────────┐
         │  CACHE (Redis)     │
         └─────────┬──────────┘
                   │
         ┌─────────▼──────────┐
         │   DATABASE         │
         │  (Primary+Replica) │
         └────────────────────┘
```

**What to Include**:
- [ ] Main components (API, Cache, DB, Queue)
- [ ] Data flow (Request → Response)
- [ ] Scaling strategy (horizontal vs vertical)
- [ ] Key decisions (Why Redis? Why sharding?)

**Pro Tips**:
1. Use whiteboard/sketch tool
2. Keep it simple initially
3. Label each component
4. Add arrows for data flow
5. Write assumptions at bottom

---

### Step 3: Deep Dive (React to Interviewer)

**Listen for Hints**:
```
Interviewer says: "How would you handle 10x load?"
→ Go deep on caching, sharding, or load balancing

Interviewer says: "What if your database goes down?"
→ Design failover, replication, backup strategy

Interviewer says: "How do you ensure consistency?"
→ Discuss transaction handling, CAP theorem

Interviewer says: "Walk me through a request"
→ Trace single request from user to response
```

**Pick 1-2 Components to Deep Dive**:

Example 1: Database Design
```
DEEP DIVE: DATABASE
├─ Schema design
│  CREATE TABLE users (
│    id BIGINT PRIMARY KEY,
│    name VARCHAR(255),
│    email VARCHAR(255),
│    created_at TIMESTAMP
│  );
│
├─ Scaling strategy
│  Shard by user_id
│  Shard ID = user_id % NUM_SHARDS
│
├─ Handling hot partitions
│  Celebrity: shard further by post_id % 100
│
└─ Backup & replication
   Primary in US-East
   Replica in US-West (async)
```

Example 2: Caching Design
```
DEEP DIVE: CACHING
├─ What to cache?
│  - User profiles (read-heavy)
│  - Recent posts (time-bound)
│  - Trending tags (pre-computed)
│
├─ Invalidation strategy
│  User updates profile → Invalidate cache
│  TTL = 1 hour for posts
│
├─ Cache key structure
│  user:{userId}:profile
│  feed:{userId}:v1
│  trending:global:tags
│
└─ Implementation
   Redis
   Capacity: 100GB (hot data)
   Eviction: LRU when full
```

---

### Step 4: Scaling & Trade-offs

**Address Bottlenecks**:

```
CURRENT DESIGN BOTTLENECKS:
┌─────────────────────────────────────┐
│ Bottleneck Analysis (in order)      │
├─────────────────────────────────────┤
│ 1. Single Load Balancer (point of   │
│    failure) → Multiple load         │
│    balancers in different regions   │
│                                     │
│ 2. Single Cache Server → Distributed│
│    Redis cluster, replication       │
│                                     │
│ 3. Single Database → Replication +  │
│    Sharding + Read replicas         │
│                                     │
│ 4. Synchronous operations → Add     │
│    message queue for async tasks    │
│                                     │
│ 5. No monitoring → Add metrics,     │
│    logs, tracing infrastructure     │
└─────────────────────────────────────┘
```

**Key Trade-off Questions**:

```
CONSISTENCY vs AVAILABILITY:
"Our feed updates 100ms slower but 99.99% available"
(Choose AP for social media, CP for banking)

LATENCY vs THROUGHPUT:
"We optimize for latency (p99 < 100ms) 
but can only handle 10K req/sec per shard"
(Video streaming: throughput; trading: latency)

COST vs PERFORMANCE:
"Use cheaper instances + more servers
instead of expensive single server"
(Horizontal cheaper than vertical at scale)

SIMPLICITY vs ACCURACY:
"Eventual consistency is simpler than
distributed transactions"
(Most systems choose eventual)

MEMORY vs CPU:
"Cache uses more memory but saves CPU"
(Usually trade-off is worth it)
```

---

## PART 2: COMMON INTERVIEW QUESTIONS CHEAT SHEET

### URL Shortener

**Your Answer Structure**:
```
Requirements:
- 1M new URLs/day
- 1B clicks/month (100K clicks/sec)
- URL expiration: 2 years
- High availability, eventual consistency OK

High-Level:
User → LB → Web Servers → Cache → Database

Deep Dive:

1. HASH FUNCTION
   Problem: 1M URLs/day * 365 * 2 years = 730M URLs
   Need unique 6-8 character code
   
   Solution: Base62 encoding
   Base62 = 62^6 = 56.8B combinations (enough)
   
   Algorithm:
   unique_id = UUID → convert to base62
   
2. DATABASE SCHEMA
   CREATE TABLE urls (
     short_url VARCHAR(8) PRIMARY KEY,
     long_url VARCHAR(2000),
     created_at TIMESTAMP,
     expires_at TIMESTAMP,
     click_count INT
   );
   
3. TRAFFIC HANDLING
   Write: 1M/day = 12 writes/sec
           → 1 database node fine
   
   Read: 100K reads/sec
         → Cache necessary
         → Redis with TTL
         → Cache hot URLs (Pareto: 20% URLs = 80% clicks)
   
4. API DESIGN
   POST /api/v1/shorten
   {
     "long_url": "https://www.example.com/path"
   }
   
   Response:
   {
     "short_url": "https://short.url/aB1C2d",
     "long_url": "https://www.example.com/path",
     "expiration_time": "2026-02-06"
   }
   
   GET /api/v1/{short_url}
   → 301/302 Redirect to long_url

5. EDGE CASES
   - Collision handling (try different hash)
   - Expiration cleanup (background job)
   - Hot URLs (cache 1 hour)
   - Rate limiting (100 shortens/min per user)
```

### Twitter/Social Media Feed

**Your Answer Structure**:
```
Requirements:
- 500M users
- 300M daily active
- 1B tweets/day
- Real-time feed
- Strong consistency for own tweets, eventual for others

High-Level:
User → LB → Web Servers → Cache → Feed Service → Databases

Deep Dive:

1. FEED GENERATION OPTIONS

   Option A: FAN-OUT ON WRITE
   When user tweets:
   ├─ Insert to user's own timeline
   ├─ For each follower:
   │  └─ Insert tweet to follower's feed
   └─ Pre-generate feeds
   
   ✅ Pros: Fast reads (pre-generated)
   ❌ Cons: Slow writes, celebrity problem (100M writes)
   
   Option B: FAN-OUT ON READ
   When user loads feed:
   ├─ Get 100 latest from each following
   ├─ Merge & rank
   └─ Return top 20
   
   ✅ Pros: Fast writes, handles celebrities
   ❌ Cons: Slow reads (merge 1000s of timelines)
   
   SOLUTION: Hybrid
   ├─ Normal users: Fan-out on write
   ├─ Celebrities: Fan-out on read
   └─ Cache results (1 hour TTL)

2. DATABASE SCHEMA
   CREATE TABLE tweets (
     tweet_id BIGINT PRIMARY KEY,
     user_id BIGINT,
     content TEXT,
     created_at TIMESTAMP,
     likes INT,
     retweets INT
   );
   
   CREATE TABLE user_feed (
     user_id BIGINT,
     tweet_id BIGINT,
     created_at TIMESTAMP,
     PRIMARY KEY (user_id, created_at DESC)
   );
   
   Sharding: By user_id
   - User 1-1M → Shard 1
   - User 1M-2M → Shard 2

3. CACHE STRATEGY
   Cache Key: feed:{user_id}:v1
   Value: List of tweet IDs (latest 100)
   TTL: 1 hour
   
   When to invalidate:
   - New tweet from following user
   - Liked/retweeted (update cache)
   - User unfollows someone

4. RANKING
   TF-IDF + Engagement:
   score = (likes * 0.5 + retweets * 0.3 + 
            recency * 0.2) / time_since_post
   
   Sort by score DESC

5. CONSISTENCY ISSUES
   Problem: User sees tweet, then doesn't see it
   Solution: "Read your own writes"
   - For current user: always fresh from primary
   - For others: OK with 1 minute lag
```

### Real-time Chat/Messaging

**Your Answer Structure**:
```
Requirements:
- 500M users
- 100M daily active
- Real-time (sub-second) delivery
- Group chats & 1-on-1
- Message history

High-Level:
User ←→ WebSocket Server → Message Queue → Message Store
         (Connection Pool)   (Kafka)        (HBase/MongoDB)

Deep Dive:

1. CONNECTION MANAGEMENT
   Challenge: 100M concurrent connections
   Solution: WebSocket server cluster
   
   Per server: 10K concurrent = 10K servers needed
   (But with load balancer: 1K servers with 100K conn each)
   
   Technology: Node.js (handles many concurrent connections)

2. MESSAGE DELIVERY
   Path:
   Sender → WebSocket Server A
         → Kafka Topic (messages)
         → WebSocket Server B
         → Receiver

   Flow:
   1. Sender connects to Server A
   2. Sends message {"to": user_2, "text": "hi"}
   3. Server A publishes to Kafka
   4. Kafka: messages-user_2 topic
   5. Server B (where user_2 connected) consumes
   6. Server B sends to user_2 via WebSocket

3. MESSAGE ORDERING
   Problem: Messages arrive out of order
   Solution: 
   - Single partition per conversation
   - Sequence number in message
   - Receiver orders by sequence

4. MESSAGE PERSISTENCE
   Schema:
   CREATE TABLE messages (
     conversation_id BIGINT,
     message_id BIGINT,
     from_user BIGINT,
     to_user BIGINT,
     content TEXT,
     created_at TIMESTAMP,
     PRIMARY KEY (conversation_id, message_id)
   );
   
   Database: HBase (handles high throughput writes)
   or MongoDB (flexible schema)

5. DELIVERY GUARANTEES
   At-least-once: Message might duplicate
   ├─ Idempotent message ID
   ├─ Check before inserting
   
   Exactly-once: Hard, use at-least-once + dedup

6. OFFLINE MESSAGES
   When user offline:
   - Messages stored in queue
   - When user comes online, fetch messages
   - Mark as delivered
```

---

## PART 3: RED FLAGS (What Interviewers DON'T Like)

### ❌ What NOT to Do

```
ANTI-PATTERN 1: Jump to Solution
❌ "Let's use Kafka and Docker"
✅ "For 100K messages/sec, we need message queue.
    Kafka is suitable because it handles ordering
    and high throughput. Alternatives: RabbitMQ (simpler,
    less throughput), AWS SQS (managed, simpler)."

ANTI-PATTERN 2: Assume Requirements
❌ "Users want geo-distribution"
✅ "Should we optimize for global users? Currently
    US-only or worldwide?"

ANTI-PATTERN 3: Skip Trade-offs
❌ "We'll use eventual consistency"
✅ "We're choosing eventual consistency for speed,
    accepting that users might see stale likes for
    up to 1 minute. This is acceptable because..."

ANTI-PATTERN 4: No Monitoring
❌ "Done!"
✅ "For monitoring, we'll collect:
    - Metrics: p99 latency, error rate, QPS
    - Logs: Request tracing, errors
    - Alerts: CPU > 80%, errors > 1%"

ANTI-PATTERN 5: Ignore Security
❌ "Users connect to database"
✅ "API authentication: JWT tokens. 
    Database: Private subnet, encrypted. 
    Rate limiting: 100 req/min per user."

ANTI-PATTERN 6: Single Point of Failure
❌ "Cache server stores everything"
✅ "Cache is distributed (Redis cluster),
    replicates to different zones.
    If cache down, fall back to database."

ANTI-PATTERN 7: No Estimation
❌ "It'll be fine"
✅ "100M DAU, avg 10 requests/user = 1.15B req/day
    = 13K req/sec, need at least 5-10 servers
    assuming 2K req/sec per server."

ANTI-PATTERN 8: Dismissing Questions
❌ Interviewer: "What about latency?"
   You: "It's not important"
✅ "That's great point. P99 latency is important.
    With our current design, adding cache reduces it
    from 500ms to 50ms. Is this acceptable?"
```

---

## PART 4: CODE EXAMPLES & PSEUDOCODE

### Example 1: Rate Limiter Implementation (Token Bucket)

```python
# Python Pseudocode

class TokenBucket:
    def __init__(self, capacity, refill_rate):
        """
        capacity: max tokens in bucket
        refill_rate: tokens per second
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.tokens = capacity
        self.last_refill = time.time()
    
    def is_allowed(self, tokens_needed=1):
        # Refill tokens based on time passed
        now = time.time()
        time_passed = now - self.last_refill
        tokens_to_add = time_passed * self.refill_rate
        
        self.tokens = min(
            self.capacity,
            self.tokens + tokens_to_add
        )
        self.last_refill = now
        
        # Check if enough tokens
        if self.tokens >= tokens_needed:
            self.tokens -= tokens_needed
            return True
        return False

# Usage:
limiter = TokenBucket(capacity=100, refill_rate=10)

for i in range(150):
    if limiter.is_allowed(1):
        print(f"Request {i}: ALLOWED")
    else:
        print(f"Request {i}: REJECTED (rate limit)")
```

### Example 2: Consistent Hashing for Caching

```python
# Consistent Hashing (handles server additions/removals)

import hashlib

class ConsistentHash:
    def __init__(self, num_virtual_nodes=3):
        self.ring = {}  # hash -> server
        self.virtual_nodes = num_virtual_nodes
    
    def _hash(self, key):
        return int(hashlib.md5(str(key).encode()).hexdigest(), 16)
    
    def add_server(self, server):
        for i in range(self.virtual_nodes):
            virtual_key = f"{server}:{i}"
            hash_value = self._hash(virtual_key)
            self.ring[hash_value] = server
    
    def remove_server(self, server):
        for i in range(self.virtual_nodes):
            virtual_key = f"{server}:{i}"
            hash_value = self._hash(virtual_key)
            del self.ring[hash_value]
    
    def get_server(self, key):
        if not self.ring:
            return None
        
        hash_value = self._hash(key)
        
        # Find first server with hash >= key_hash
        sorted_hashes = sorted(self.ring.keys())
        for h in sorted_hashes:
            if h >= hash_value:
                return self.ring[h]
        
        # Wrap around
        return self.ring[sorted_hashes[0]]

# Usage:
ch = ConsistentHash()
ch.add_server("server1")
ch.add_server("server2")
ch.add_server("server3")

# Route keys to servers
for user_id in range(1, 101):
    server = ch.get_server(f"user:{user_id}")
    print(f"User {user_id} → {server}")

# Add new server (only ~N/k keys rehashed)
ch.add_server("server4")
```

### Example 3: Database Schema for URL Shortener

```sql
-- URL Storage
CREATE TABLE urls (
    short_code VARCHAR(8) PRIMARY KEY,
    long_url VARCHAR(2000) NOT NULL,
    user_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    is_deleted BOOLEAN DEFAULT FALSE,
    
    INDEX idx_user_created (user_id, created_at),
    INDEX idx_expires (expires_at)
);

-- Analytics (separate table for high-write scenarios)
CREATE TABLE url_clicks (
    click_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    short_code VARCHAR(8),
    user_agent VARCHAR(500),
    ip_address VARCHAR(15),
    clicked_at TIMESTAMP,
    
    FOREIGN KEY (short_code) REFERENCES urls(short_code),
    INDEX idx_short_code_time (short_code, clicked_at)
);

-- Queries
-- Get original URL
SELECT long_url FROM urls 
WHERE short_code = 'aB1C2d' 
  AND (expires_at IS NULL OR expires_at > NOW())
  AND is_deleted = FALSE;

-- Record click
INSERT INTO url_clicks (short_code, user_agent, ip_address) 
VALUES ('aB1C2d', 'Mozilla/5.0...', '192.168.1.1');

-- Get click stats
SELECT COUNT(*) as clicks 
FROM url_clicks 
WHERE short_code = 'aB1C2d' 
  AND clicked_at > DATE_SUB(NOW(), INTERVAL 1 DAY);
```

### Example 4: Message Queue Pattern (Pseudocode)

```python
# Message Queue / Pub-Sub Pattern

class MessageQueue:
    def __init__(self):
        self.topics = {}  # topic_name -> list of subscribers
        self.queue = {}   # topic_name -> list of messages
    
    def subscribe(self, topic, subscriber):
        if topic not in self.topics:
            self.topics[topic] = []
            self.queue[topic] = []
        
        self.topics[topic].append(subscriber)
        print(f"{subscriber} subscribed to {topic}")
    
    def publish(self, topic, message):
        if topic not in self.queue:
            self.queue[topic] = []
        
        self.queue[topic].append(message)
        
        # Notify all subscribers
        for subscriber in self.topics.get(topic, []):
            self._deliver(subscriber, message)
    
    def _deliver(self, subscriber, message):
        # Simulate delivery
        print(f"Delivering to {subscriber}: {message}")

# Usage:
mq = MessageQueue()

# Subscribers
mq.subscribe("orders", "email_service")
mq.subscribe("orders", "inventory_service")
mq.subscribe("orders", "notification_service")

# Publisher
mq.publish("orders", {
    "order_id": 123,
    "user_id": 456,
    "amount": 99.99,
    "timestamp": "2024-01-01T12:00:00Z"
})

# Output:
# email_service subscribed to orders
# inventory_service subscribed to orders
# notification_service subscribed to orders
# Delivering to email_service: {'order_id': 123, ...}
# Delivering to inventory_service: {'order_id': 123, ...}
# Delivering to notification_service: {'order_id': 123, ...}
```

---

## PART 5: INTERVIEW DOS AND DON'TS

### ✅ DO's

```
✅ START WITH QUESTIONS
   "Before I design, let me confirm requirements..."

✅ DRAW EARLY AND OFTEN
   "Let me sketch the high-level architecture"

✅ COMMUNICATE YOUR THINKING
   "I chose Redis because we need sub-second 
    cache lookups and it handles 100K read/sec"

✅ DISCUSS TRADE-OFFS
   "We're using eventual consistency to get 
    availability, accepting 1 minute latency"

✅ ESTIMATE NUMBERS
   "100M DAU × 10 requests = 1.15B requests/day = 13K RPS"

✅ HANDLE FEEDBACK
   "Great point about failover. Let me add 
    replication across zones..."

✅ ASK CLARIFYING FOLLOW-UPS
   "When you say real-time, do you mean <100ms latency?"

✅ THINK OUT LOUD
   "One approach: use cache, but if cache is down...
    Let me think about that"

✅ GO DEEP ON FEEDBACK
   Interviewer: "How do you handle 10x load?"
   You: "Great question. Current bottleneck is database.
        We'd add replication, sharding, and caching..."
```

### ❌ DON'Ts

```
❌ DON'T START CODING IMMEDIATELY
   (This is architecture, not coding interview)

❌ DON'T IGNORE EDGE CASES
   Hot shards, network failures, data consistency

❌ DON'T ASSUME REQUIREMENTS
   Always clarify with interviewer

❌ DON'T OVER-ENGINEER
   Start simple, add complexity as needed

❌ DON'T IGNORE FOLLOW-UP QUESTIONS
   They're testing your ability to adapt

❌ DON'T MAKE UP NUMBERS
   "We'd need 1000 servers" → "Why? How did you estimate?"

❌ DON'T BE DEFENSIVE
   Interviewer: "What about security?"
   You: "Oh... I didn't think about it" 
   (Not: "Security isn't important")

❌ DON'T MEMORIZE SOLUTIONS
   "This is how Uber designed X"
   (Apply principles, adapt to requirements)

❌ DON'T FORGET MONITORING/SECURITY
   Production systems need both

❌ DON'T RUN OUT OF TIME
   Manage: 5 min clarify, 10 min design, 
          15 min deep dive, 15 min scaling
```

---

## PART 6: DIFFICULT FOLLOW-UP QUESTIONS & ANSWERS

### Q: "What if your database goes down?"

```
GOOD ANSWER:
"Great question. We have several layers of protection:

1. REPLICATION
   Primary DB in US-East
   Synchronous replica in US-West
   If primary fails: automatic failover to replica
   RPO (Recovery Point Objective): 0 (no data loss)
   RTO (Recovery Time Objective): <30 seconds

2. READ-ONLY REPLICA SERVING
   While primary is recovering, read traffic 
   goes to read-only replicas
   Write traffic queued or returns error

3. BACKUP & RECOVERY
   Daily backups stored in S3
   Can restore to point-in-time
   Recovery takes ~1 hour (acceptable downtime)

4. CIRCUIT BREAKER
   If database not responding:
   - Return cached data or stale data
   - Don't hammer failed database
   - Retry after delay

Trade-off: Replication adds ~10ms latency 
(sync write to both zones) but gives us disaster recovery."
```

### Q: "How do you handle consistency with eventual consistency?"

```
GOOD ANSWER:
"With eventual consistency, there's temporary 
inconsistency. We handle it by:

1. READ-YOUR-WRITES CONSISTENCY
   User posts tweet → stored in cache for their feed
   User immediately sees their own tweet
   (Even though it might not be in database yet)

2. CONFLICTS RESOLUTION
   Last-write-wins for simple cases
   Timestamps included in all updates
   'If there's conflict, newer timestamp wins'

3. IDEMPOTENT OPERATIONS
   Like button: idempotent (click 10 times = 1 like)
   Prevents accidental duplicates on retry

4. NOTIFICATION ON EVENTUAL SYNC
   Real-time for critical (payment): strong consistency
   Non-critical (likes): eventual consistency
   'Like count updated momentarily'

5. ACCEPT CONSTRAINTS
   Users understand social media might be slightly stale
   'You see 99 likes, actually 100' is acceptable

For PAYMENT systems, we'd use strong consistency instead."
```

### Q: "How do you prevent DDoS attacks?"

```
GOOD ANSWER:
"DDoS protection has multiple layers:

1. RATE LIMITING
   Per-user: 100 requests/minute
   Per-IP: 1000 requests/minute
   Global: 1M requests/sec from any source
   
   Exceeding limit → 429 Too Many Requests response

2. WAF (Web Application Firewall)
   Block known attack patterns
   Geographic blocking (unusual countries)
   Behavioral analysis (sudden spike = attack)

3. CDN & EDGE PROTECTION
   Traffic filtered through CDN
   Edge servers absorb initial traffic
   Prevents hitting origin servers

4. AUTHENTICATION
   Real users: JWT token (verified)
   Bots: detected and blocked

5. SCALE
   If DDoS still gets through:
   Auto-scale servers horizontally
   Spread load across more resources

6. INFRASTRUCTURE
   Multiple data centers
   Geographically distributed
   If one DC flooded, others unaffected

Example: Netflix can handle 10x normal traffic 
spikes even when under DDoS."
```

---

## PART 7: MONITORING & OBSERVABILITY CHECKLIST

When designing, don't forget monitoring!

### What to Monitor

```
METRICS (numbers):
✅ Throughput: Requests/second, transactions/sec
✅ Latency: p50, p95, p99 (not just average!)
✅ Error Rate: 5xx errors, timeouts
✅ Saturation: CPU, memory, disk, network

LOGS:
✅ Request/response metadata
✅ Errors and exceptions
✅ User actions (login, purchase, etc.)
✅ Performance logs

TRACES:
✅ Full request journey through services
✅ Which service is slow?
✅ Database query performance

ALERTS:
✅ CPU > 80%
✅ Error rate > 1%
✅ Latency p99 > 1s
✅ Out of disk space < 10%
```

### Monitoring Tools

```
METRICS:
- Prometheus (open source)
- Datadog (enterprise)
- New Relic (enterprise)

LOGS:
- ELK Stack (Elasticsearch, Logstash, Kibana)
- Splunk (enterprise)
- CloudWatch (AWS)

TRACES:
- Jaeger (open source)
- Zipkin (open source)
- DataDog APM (enterprise)

DASHBOARDS:
- Grafana (open source)
- Custom dashboards
```

---

## 📋 PRE-INTERVIEW CHECKLIST

### 24 Hours Before
- [ ] Review 3 system designs you've practiced
- [ ] Brush up on CAP theorem
- [ ] Review trade-offs in caching and databases
- [ ] Get good sleep

### 2 Hours Before
- [ ] Test your internet connection
- [ ] Have whiteboard/drawing tool ready
- [ ] Have paper and pen nearby
- [ ] Close other applications
- [ ] Use bathroom (no interruptions!)

### During Interview
- [ ] Smile (even if video, they can hear it)
- [ ] Speak clearly and confidently
- [ ] Pause before answering (think!)
- [ ] Draw while explaining
- [ ] Ask for clarification
- [ ] Admit when you don't know

### Red Lights (Adjust Strategy)
- [ ] Interviewer: "That's wrong"
  → Pause, think, ask what's wrong, adapt

- [ ] Interviewer: "What if X?"
  → This is good! Go deep on their hint

- [ ] Interviewer: "Design is too simple"
  → Add complexity: replication, sharding, monitoring

- [ ] Running out of time?
  → Summarize quickly, ask what to focus on

---

## 🎯 FINAL INTERVIEW TIPS

### Building Confidence
```
"I've seen this pattern before" (you have!)
"Let me think about this" (pause is good, shows thinking)
"Great question, I should have mentioned" 
  (positive, shows listening)
```

### Handling Nervousness
```
Remember:
- Interviewer WANTS you to succeed
- They're not testing to fail you
- Some candidates fail, that's expected
- Your approach matters more than perfect design
```

### After Interview
```
Good signs:
✅ Interviewer asked many follow-up questions
✅ Interviewer extended time
✅ Interviewer seemed engaged/nodding
✅ You explained trade-offs clearly

Not necessarily bad:
❌ Interview ended early (you said all you could)
❌ Interviewer corrected you (you're learning)
❌ Some blank stares (they're thinking too)
```

---

**Now go practice! You've got this! 💪**
