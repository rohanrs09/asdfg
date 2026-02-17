# 🗺️ SYSTEM DESIGN VISUAL LEARNING ROADMAP
## Week-by-Week Breakdown & Concept Map

---

## WEEK-BY-WEEK PROGRESSION (12-WEEK INTENSIVE TRACK)

### WEEK 1: Client-Server & Basics
**Daily Focus**:
- Monday: What is system design? Why it matters?
- Tuesday: Client-server architecture
- Wednesday: HTTP/HTTPS, REST basics
- Thursday: APIs, requests/responses
- Friday: Design simple API

**Resources**:
- Video: Gaurav Sen "What is System Design" (12 min)
- Video: ByteByteGo "Client-Server Model" (8 min)
- Course: Grokking Fundamentals - Section 1

**Mini Project**: Design API for TODO app
- GET /todos → list all
- POST /todos → create
- PUT /todos/{id} → update
- DELETE /todos/{id} → delete

**Self-Test**: 
- [ ] Draw client-server diagram
- [ ] Explain HTTP method differences
- [ ] Design simple CRUD API

---

### WEEK 2: Scalability & Load Balancing
**Daily Focus**:
- Monday: Vertical vs horizontal scaling
- Tuesday: Load balancing basics
- Wednesday: Load balancing algorithms
- Thursday: Session management
- Friday: Design scalable API

**Resources**:
- Video: Gaurav Sen "Vertical vs Horizontal Scaling" (12 min)
- Video: Gaurav Sen "Load Balancing" (15 min)
- Course: Grokking Fundamentals - Section 2-3
- Read: System Design Interview Vol 1 - Chapter 1

**Mini Project**: Design API for 1M concurrent users
- Implement load balancer logic
- Calculate server capacity
- Design session management

**Self-Test**:
- [ ] Calculate: 100M daily users → requests/sec?
- [ ] Explain round robin vs least connections
- [ ] Solve: Handle session with 3 servers?

---

### WEEK 3: Caching & Databases
**Daily Focus**:
- Monday: Caching fundamentals
- Tuesday: Cache invalidation strategies
- Wednesday: Database basics
- Thursday: SQL vs NoSQL
- Friday: Caching + DB design

**Resources**:
- Video: ByteByteGo "Caching Explained" (12 min)
- Video: Gaurav Sen "Caching Strategies" (18 min)
- Course: Grokking Fundamentals - Section 4-5
- Read: System Design Interview Vol 1 - Caching chapter

**Mini Project**: Add caching to URL shortener
- Cache hot URLs (most accessed)
- Implement TTL expiration
- Handle cache misses

**Self-Test**:
- [ ] Compare write-through vs write-back caching
- [ ] When to use Redis vs Memcached?
- [ ] Design cache for 1B daily requests?

---

### WEEK 4: Database Scaling
**Daily Focus**:
- Monday: Database replication
- Tuesday: Database sharding
- Wednesday: Sharding strategies
- Thursday: SQL vs NoSQL deep dive
- Friday: Design database for scale

**Resources**:
- Video: Gaurav Sen "Database Sharding" (18 min)
- Video: Gaurav Sen "Master-Slave Replication" (12 min)
- Course: ByteByteGo - Database scaling section
- Read: DDIA - Chapters 4-5

**Mini Project**: Scale Twitter database
- 300M users, 1.5B tweets/day
- Design sharding key (by user ID)
- Handle hot shards (celebrities)

**Self-Test**:
- [ ] Calculate: Need replication or sharding?
- [ ] Design sharding for 1B users?
- [ ] Handle hot partition problem?

---

### WEEK 5: Message Queues & Async
**Daily Focus**:
- Monday: Message queue fundamentals
- Tuesday: Pub/Sub pattern
- Wednesday: Event-driven architecture
- Thursday: Kafka vs RabbitMQ vs SQS
- Friday: Design async system

**Resources**:
- Video: ByteByteGo "Message Queues" (12 min)
- Video: Gaurav Sen "Pub/Sub Pattern" (10 min)
- Course: ByteByteGo - Async processing
- Read: DDIA - Chapter 11

**Mini Project**: Notification system
- User subscribes to events
- Publish events (new post, like, comment)
- Multiple consumers (email, SMS, in-app)

**Self-Test**:
- [ ] When to use queue vs direct call?
- [ ] Compare RabbitMQ vs Kafka?
- [ ] Design notification for 1M users?

---

### WEEK 6: Interview Problem #1 - URL Shortener
**Daily Focus**:
- Monday-Friday: Design, implement, optimize URL shortener

**Resources**:
- Read: "System Design Interview" Vol 1 - URL Shortener
- Video: ByteByteGo "Design URL Shortener" (YouTube)
- Code: Implement actual logic (Python/Java)

**Full Design**:
```
Requirements:
- Shorten long URLs
- Redirect to original URL
- Analytics (clicks, locations)
- 1M URLs/day, 1B redirects/day

Architecture:
┌─ API (POST /shorten, GET /:hash)
├─ Web servers (stateless)
├─ Load balancer
├─ Cache (Redis) - hot URLs
├─ Database - URL mapping
├─ Analytics service (async)
└─ Monitoring

Deep dives:
1. Hash function (collision avoidance)
2. Database schema
3. Cache strategy
4. Hot URL handling
```

**Self-Test**:
- [ ] Implement /shorten API
- [ ] Optimize /redirect with caching
- [ ] Handle 1M/day scale

---

### WEEK 7: Interview Problem #2 - Chat/Messaging
**Daily Focus**:
- Monday-Friday: Design real-time messaging system

**Resources**:
- Video: "Design Messenger" (Gaurav Sen or ByteByteGo)
- Study: Message ordering, real-time delivery

**Full Design**:
```
Requirements:
- 1-on-1 messages, groups
- Real-time delivery
- Message history
- 500M users, 1B messages/day

Challenges:
1. Real-time (WebSockets, not polling)
2. Message ordering
3. Delivery guarantees
4. Group scalability

Architecture:
┌─ API Gateway
├─ Connection pool (WebSockets)
├─ Message queue (Kafka)
├─ Message store (HBase)
├─ Cache (Redis) - recent messages
└─ Notification service
```

---

### WEEK 8: CAP Theorem & Consistency
**Daily Focus**:
- Monday: CAP theorem
- Tuesday: Consistency models
- Wednesday: Eventual consistency patterns
- Thursday: Consensus algorithms
- Friday: Apply to systems

**Resources**:
- Video: ByteByteGo "CAP Theorem Explained" (10 min)
- Read: DDIA - Chapters 8-9
- Course: ByteByteGo - Distributed systems

**Deep Concepts**:
```
CAP Theorem:
- Consistency: All reads see latest write
- Availability: System always responds
- Partition tolerance: Works despite network splits

Trade-offs:
CP: SQL (MySQL) - consistency over availability
AP: NoSQL (DynamoDB) - availability over consistency

Examples:
- Banking (CP): Strong consistency matters
- Social Media (AP): Slight stale data OK
```

**Self-Test**:
- [ ] Design CP system (banking)?
- [ ] Design AP system (feed)?
- [ ] Use consensus algorithms?

---

### WEEK 9: Interview Problem #3 - Feed System (Hard)
**Daily Focus**:
- Monday-Friday: Design social media newsfeed

**Resources**:
- "Design Twitter/Facebook Feed"
- YouTube: Complex system design videos

**Challenges**:
```
Requirements:
- 300M daily active users
- 500M followers relationship
- Real-time feed
- Likes, comments, retweets

Problems:
1. Feed generation (1000s of posts for 1 user)
2. Ranking (which posts first?)
3. Hot users (celebrity with 100M followers)
4. Real-time consistency

Solutions:
- Fan-out on write (new post → all followers' feeds)
- Fan-out on read (read time, merge latest)
- Cache hot feeds (trending, celebrities)
- Denormalization (store user name in post)
```

---

### WEEK 10: Advanced Topics - Microservices & API Gateway
**Daily Focus**:
- Monday: Microservices architecture
- Tuesday: API Gateway
- Wednesday: Service discovery
- Thursday: Circuit breakers
- Friday: Design microservices system

**Resources**:
- Video: ByteByteGo "Microservices"
- Book: "Building Microservices" (Sam Newman)

**Key Patterns**:
```
Service Mesh:
Auth Service → API Gateway → Order Service
                          → Inventory Service
                          → Payment Service

Benefits:
- Independent scaling
- Technology agnostic
- Fault isolation

Challenges:
- Network latency
- Data consistency
- Operational complexity
```

---

### WEEK 11: Monitoring, Logging, Security
**Daily Focus**:
- Monday: Monitoring & metrics
- Tuesday: Logging & aggregation
- Wednesday: Distributed tracing
- Thursday: Security (auth, encryption)
- Friday: Apply to system

**Resources**:
- ByteByteGo "Observability"
- OWASP Top 10

**Setup**:
```
Monitoring:
- Metrics (Prometheus): CPU, memory, requests/sec
- Alerts: When metrics exceed threshold

Logging:
- Centralized logs (ELK stack)
- Structured logging (JSON)

Tracing:
- Follow request through services (Jaeger)

Security:
- HTTPS/TLS encryption
- JWT tokens for auth
- Rate limiting
- Input validation
```

---

### WEEK 12: Mock Interviews & Review
**Daily Focus**:
- Monday: Mock interview #1 (easy)
- Tuesday: Review + practice
- Wednesday: Mock interview #2 (medium)
- Thursday: Review + weak areas
- Friday: Mock interview #3 (hard)

**Practice Platforms**:
- Exponent.dev
- Codemia.io
- Friends mock interview

**Evaluation**:
- [ ] Clarified ambiguity?
- [ ] Drew diagrams?
- [ ] Discussed trade-offs?
- [ ] Handled feedback?
- [ ] Explained clearly?

---

## 🎯 CONCEPT MAP (How Everything Connects)

```
SYSTEM DESIGN

├─ SCALABILITY
│  ├─ Vertical Scaling (upgrade single server)
│  ├─ Horizontal Scaling (add more servers)
│  └─ Load Balancing (distribute requests)
│     ├─ Round Robin
│     ├─ Least Connections
│     ├─ IP Hash
│     └─ Weighted

├─ PERFORMANCE
│  ├─ Caching
│  │  ├─ Browser Cache
│  │  ├─ CDN Cache
│  │  ├─ Application Cache (Redis)
│  │  ├─ Database Cache
│  │  └─ Cache Invalidation
│  │     ├─ TTL
│  │     ├─ Write-through
│  │     ├─ Write-back
│  │     └─ Event-based
│  │
│  └─ Content Delivery
│     ├─ CDN (geographically distributed)
│     └─ Compression

├─ STORAGE
│  ├─ Database Selection
│  │  ├─ SQL (ACID, structured)
│  │  │  └─ Scaling: Replication, Sharding
│  │  └─ NoSQL (flexible, scalable)
│  │     ├─ Document (MongoDB)
│  │     ├─ Key-Value (DynamoDB)
│  │     ├─ Graph (Neo4j)
│  │     └─ Wide Column (Cassandra)
│  │
│  ├─ Replication (for reads)
│  │  ├─ Primary-Replica
│  │  └─ Master-Master
│  │
│  └─ Sharding (for writes)
│     ├─ Range-based
│     ├─ Hash-based
│     ├─ Directory-based
│     └─ Geo-based

├─ RELIABILITY
│  ├─ Redundancy
│  ├─ Replication
│  ├─ Backup & Recovery
│  └─ Failover

├─ CONSISTENCY
│  ├─ Strong Consistency (all reads = latest)
│  ├─ Eventual Consistency (reads may be stale)
│  ├─ CAP Theorem
│  │  ├─ CP Systems (SQL)
│  │  └─ AP Systems (NoSQL)
│  └─ Consensus
│     ├─ Raft
│     ├─ Paxos
│     └─ PBFT

├─ COMMUNICATION
│  ├─ Synchronous
│  │  ├─ REST APIs
│  │  └─ gRPC
│  │
│  └─ Asynchronous
│     ├─ Message Queues
│     │  ├─ RabbitMQ
│     │  ├─ Kafka
│     │  └─ AWS SQS
│     └─ Pub/Sub
│        ├─ Topics
│        └─ Event-driven

├─ ARCHITECTURE PATTERNS
│  ├─ Monolithic (single application)
│  ├─ Microservices (multiple services)
│  ├─ Serverless (functions as a service)
│  └─ Service Mesh

├─ OBSERVABILITY
│  ├─ Metrics (numbers: CPU, req/sec)
│  ├─ Logs (detailed events)
│  └─ Traces (request journey)

└─ SECURITY
   ├─ Authentication (who are you?)
   │  ├─ OAuth 2.0
   │  └─ JWT
   │
   ├─ Authorization (can you do this?)
   │  └─ RBAC
   │
   ├─ Encryption
   │  ├─ HTTPS/TLS
   │  └─ Data encryption
   │
   └─ Protection
      ├─ Rate limiting
      ├─ Input validation
      └─ DDoS protection
```

---

## 📊 COMPLEXITY PROGRESSION

### Simple Systems (Week 1-4)
```
Traditional Web App:
User → Web Server → Database
(Single server setup)
```

### Medium Systems (Week 5-8)
```
Scalable Web App:
User → Load Balancer → Web Servers (N) 
                    → Cache (Redis)
                    → Database (Primary + Replicas)
```

### Complex Systems (Week 9-12)
```
Large-Scale Platform:
User → CDN → API Gateway → Load Balancer → Web Servers
                         → Cache Layer (multi-tier)
                         → Database (Sharded + Replicas)
                         → Message Queue (async jobs)
                         → Search Engine
                         → Analytics Pipeline
                         → Monitoring System
```

---

## 🎓 PROBLEM DIFFICULTY PROGRESSION

### Easy (Week 4-6)
- URL Shortener
- Web Crawler
- API Rate Limiter
- Cache Implementation

### Medium (Week 7-9)
- Social Media Feed
- Real-time Chat
- Ride Sharing (Uber)
- Video Streaming (YouTube)

### Hard (Week 10-12)
- Google Search
- WhatsApp (messaging at scale)
- Distributed Database
- Payment System

---

## 💡 DESIGN CHECKLIST FOR EVERY PROBLEM

### Phase 1: Requirements (5 minutes)
- [ ] Users: How many? Daily active? Peak?
- [ ] Reads vs Writes: What's the ratio?
- [ ] Latency: Real-time or eventual?
- [ ] Consistency: Strong or eventual?
- [ ] Geographic: Global or regional?
- [ ] Data: How much? Growth rate?

### Phase 2: High-Level Design (10 minutes)
- [ ] Draw architecture with 4-6 main components
- [ ] Data flow: Request path from user to response
- [ ] Key decisions: Which database? Which cache?
- [ ] Bottleneck analysis: What's the constraint?

### Phase 3: Deep Dive (15 minutes)
- [ ] Pick 1-2 components based on interviewer hints
- [ ] Detailed design of that component
- [ ] Trade-offs and why you chose this way

### Phase 4: Scaling & Edge Cases (5 minutes)
- [ ] How to handle 10x users?
- [ ] How to handle outages?
- [ ] How to monitor and alert?

---

## 🔧 TECHNOLOGY STACK BY SYSTEM TYPE

### Web Application
```
Frontend: React/Vue (JavaScript)
Backend: Node.js/Java/Python
API: REST with JSON
Database: PostgreSQL
Cache: Redis
Queue: RabbitMQ
Monitoring: Prometheus + Grafana
```

### Real-Time Chat
```
Frontend: React + WebSockets
Backend: Node.js (WebSocket server)
Message Store: MongoDB (flexible schema)
Real-time: Socket.io or Kafka
Cache: Redis (recent messages)
Search: Elasticsearch
```

### Video Platform
```
Frontend: React + HLS player
Backend: Node.js/Go
Upload: Direct to S3
Processing: Lambda (serverless transcoding)
Streaming: CloudFront (CDN)
Database: DynamoDB (metadata)
Search: Elasticsearch
Analytics: Kafka → Data warehouse
```

### Social Network
```
Frontend: React/React Native
Backend: Java/Go (multiple services)
Database: Cassandra (read-optimized)
Cache: Redis (feeds, sessions)
Queue: Kafka (events)
Search: Elasticsearch (users, posts)
Notification: Firebase
Analytics: Spark + HDFS
```

---

## 📈 SCALE BENCHMARKS (Real-world Reference)

### Companies & Their Scale
```
Facebook:
- 3B monthly active users
- Billions of messages/day
- Billions of photos/day

Twitter:
- 500M users
- 500M tweets/day (average)
- Real-time ranking

Netflix:
- 250M subscribers
- Billions of streams/day
- Personalized recommendations

Uber:
- 100M monthly active users
- 15M daily rides
- Real-time location tracking

Spotify:
- 500M users
- Personalized playlists for each
- Music recommendations

Amazon:
- 300M+ daily active users
- Real-time inventory
- Distributed globally
```

### Your System Targets
```
1M daily active users: Moderate scale
- 1 database server with replication
- 2-5 web servers
- Single cache server

10M daily active users: Large scale
- Sharded database
- 20-50 web servers
- Distributed cache
- Message queue for async

100M daily active users: Enterprise scale
- Multiple data centers
- Sharded + replicated database
- 100+ servers
- Complex monitoring
- Service-oriented architecture

1B daily active users: Google/Facebook scale
- Multiple geographic regions
- Edge computing
- Advanced caching strategies
- Specialized data stores
```

---

## 🎬 PRACTICE PROJECT ROADMAP

### Project 1: URL Shortener (2 weeks)
**Skills**: API design, database, caching
**Deliverable**: 
- Working API
- Database schema
- Load testing

### Project 2: Real-time Notification System (3 weeks)
**Skills**: Message queues, async processing, real-time
**Deliverable**:
- Push notifications
- Message queue integration
- Multiple notification channels

### Project 3: Mini Social Network (4 weeks)
**Skills**: Feed, ranking, social features
**Deliverable**:
- User profiles
- Post creation
- Feed generation
- Like/comment

### Project 4: Video Platform (4 weeks)
**Skills**: Video encoding, streaming, recommendations
**Deliverable**:
- Upload & encoding
- Adaptive streaming
- Basic recommendations

---

## 📝 NOTES TEMPLATE FOR EACH CONCEPT

When learning a new concept, use this template:

```
CONCEPT: [Name]

DEFINITION:
[2-3 sentence explanation]

WHEN TO USE:
[Scenarios where this applies]

TRADE-OFFS:
[Pros] [Cons]

EXAMPLE:
[Real-world usage]

IMPLEMENTATION:
[How to actually do it]

RELATED CONCEPTS:
[What connects to this]

INTERVIEW NOTES:
[What interviewer might ask]
```

---

## ✅ FINAL CHECKLIST

### Knowledge
- [ ] Understand scalability concepts
- [ ] Know caching strategies
- [ ] Understand database scaling
- [ ] Know CAP theorem
- [ ] Understand consistency models
- [ ] Know microservices patterns
- [ ] Understand message queues
- [ ] Know API design best practices
- [ ] Understand monitoring/logging
- [ ] Know security patterns

### Skills
- [ ] Draw architecture diagrams clearly
- [ ] Make reasonable estimates
- [ ] Discuss trade-offs confidently
- [ ] Handle ambiguous questions
- [ ] Explain decisions clearly
- [ ] Ask clarifying questions
- [ ] Adapt design based on feedback

### Practice
- [ ] Solve 20+ interview problems
- [ ] Do 5+ mock interviews
- [ ] Build 3+ projects
- [ ] Explain each design to someone else
- [ ] Time yourself (45 min per problem)

### Ready to Interview
- [ ] Comfortable with core concepts
- [ ] Can design any system
- [ ] Handle follow-up questions smoothly
- [ ] Confident in your explanations

---

## 🚀 RESOURCES AT A GLANCE

```
FREE (YouTube):
- Gaurav Sen (system design basics)
- ByteByteGo (visual diagrams)
- NeetCode (beginner-friendly)

PAID COURSES ($39-79):
- Grokking System Design Fundamentals (DesignGurus)
- Grokking System Design Interview (DesignGurus)
- ByteByteGo (annual subscription, ~$50)

BOOKS:
- System Design Interview Vol 1 & 2 (Alex Xu, ~$40)
- Designing Data-Intensive Applications (Kleppmann, ~$40)

PRACTICE:
- LeetCode System Design
- Exponent (mock interviews)
- Codemia (mentorship)

GITHUB:
- System Design Primer (comprehensive reference)
```

---

**Now you have a complete learning roadmap. Pick one and START TODAY! 🎯**
