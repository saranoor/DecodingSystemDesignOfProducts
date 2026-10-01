# Monzo
![ScreenShot](/Monzo/mono_high_level_architect.png)

# Functional Requriemnt
Functional
- View balance and real-time transaction feed.
- Pay by card (contactless, chip, online) with instant authorisation.
- Send and receive bank transfers (Faster Payments, Bacs, and so on).
- Manage pots, savings, direct debits, standing orders, and scheduled payments.
- Receive instant push notifications for every transaction.
- Freeze or unfreeze cards, and open accounts with KYC (identity verification).
- Contact support in-app.


# Non Functional Requirement
- Correctness and consistency: no double-spend, no lost transaction.
- High availability: a card authorisation must succeed even if parts of the system are down.
- Low latency: card networks expect a decision within seconds.
- Auditability: every money movement is traceable, as regulators require.
- Security and compliance: zero-trust networking, encryption, and fraud controls.
- Scalability: must handle spikes like salary day.

# Services/Extra
-Ledger and accounts service: the source of truth for balances.
-Card processing service: real-time authorisation decisions. (card authorization, card issuer)
-Payments service: bank transfers over external schemes.
-Feed and notification service: the customer-facing view.
-Risk and fraud service: decisions before money moves.

# Capacity estimatin
Card and payment transactions	10M × 3/day = 30M/day ≈ 350 TPS average, ~3,500 TPS peak (10× at salary time)
App API calls	10M users × 4 sessions × 10 calls = 400M/day ≈ 4,600 RPS average, ~20k RPS peak
Storage	30M tx × ~2 KB = ~60 GB/day ≈ 22 TB/year, ~66 TB/year with replication factor 3
Notifications	~30M/day, one per transaction
- users = 10 M
- Read = ?
- Writes = ?

Writes. These come from money movement.

10M active users x 3 card or payment transactions per day = 30M transactions/day.
Each transaction fans out to about 8 database writes (authorisation hold, ledger entries, feed entry, idempotency key, notification record, and so on).
30M x 8 = 240M writes/day, which is about 2,800 writes/sec average.
With a 10x peak, that is about 28,000 writes/sec.

Reads. These come from app usage.

10M users x 4 sessions x 10 API calls = 400M API calls/day.
Each call causes about 5 database reads across services (balance, feed, pots, and so on).
400M x 5 = 2B reads/day, which is about 23,000 reads/sec average.
With a 5x peak, that is about 115,000 reads/sec.

Ratio. Roughly 8:1 reads to writes.

Monzo has published its actual peak numbers: over 2,000,000 reads and 100,000 writes per second on Amazon Keyspaces, across more than 350 TB of data. That is about a 20:1 read:write ratio.

# Design Evolution
- Early stage (2015–2016): Monzo started with a small set of services. By the beta launch the backend had grown to nearly 100 services, and the team reconsidered its architectural choices ahead of the banking licence.
- Scaling stage: by 2019 there were around 1,500 services with over 9,300 inter-service interactions. A 2022 engineering blog post puts it at around 2,000 microservices.


# Tech stack
- Go: GO it's quite simple. It's statically typed. It makes it quite easy for us to get people on board. Go also has some interesting things such as a backwards compatibility guarantee. We've been using Go from the very early versions of Go 1. Every time a new version of Go comes out, it has a guarantee that we can recompile our code, and we basically get all of the improvements. What that means is the garbage collector, for example, has improved several orders of magnitude over the time that we've had our infrastructure running. Every time we recompile it, test that it still works. Then we just get those benefits for free.

- Cassandra: 
- Kafka
- Kubernetes and Docker 
- Envoy Proxy for RPC
- AWS for most of our production infrastructure and GCP for most of our data infrastructure
- React for our public web apps and internal tools
-Prometheus, Grafana, The Elastic Stack
- etcd for distributed locking

# Architect
Microservices Architecture. Single responsibility per service, clear bounded contexts (cards, payments, accounts, notifications), and independent deployment.

Is Monzo synchronous or asynchronous?
Services communicate by RPC for synchronous calls and by events for asynchronous flows.


# High Level Design

- Data base
- Data Model Design
- distributed system 

## Data base
- Horizontally scalable, with no single write bottleneck.
- Highly available across nodes and zones.
- Suits high-volume, append-heavy data such as transactions
- Cassandra because it provides a masterless, horizontally scalable database that gives us a lot of controls over writing the data to multiple locations. We don't have this one server, a primary that fails over to a secondary.

I think there's a common misconception that banks must be ACID compliant. Eric Brewer wrote a really interesting post several years ago, about how banks are basically available with soft state and eventual consistency as their BASE rather than ACID. I think that's the thing that we really have to think about here.

In our case, we use Cassandra because it provides a masterless, horizontally scalable database that gives us a lot of controls over writing the data to multiple locations. We don't have this one server, a primary that fails over to a secondary. We can avoid those problems. We've traded off the ability to have transactional consistency. That might sound insane for a bank, but we can provide those things. In many cases, the financial networks already deal with this. If you tap your card, which supports offline transactions in a store, most cards will allow you to do three or so transactions for up to £30. Hypothetically, your bank may only find out about that two or three days later, at which point the bank is now told you spent £28 on your card. If you don't have £28, they're still going to remove £28 from your account. You have a series of commutative operations that are like debit £28, credit lots of pounds, debit some number of pounds.

When we talk about consistency and transactional isolation, you're trading off financial risk for consistency. Sometimes that's ok. Sometimes that's not ok. We have a variety of different systems. Some things we won't allow transactions to happen unless we can achieve a lock and successfully complete a transaction and then unlock. In many other cases, we get told about it two or three days later, in which case, we don't need a lock. We have a queue of things to apply to people's accounts. We don't really rely on the eventual consistency that much, in that particular case, but we don't need transactional isolation.

# Low Level Design

# Flow
The ledger, card authorization flow and payment is illustrative in the diagram below
![ScreenShot](/Monzo/card_authorization.png)
![ScreenShot](/Monzo/ledger_flow.png)
![ScreenShot](/Monzo/payment_state_machine.png)
# Challenge
1. Resilience against total platform failure (the biggest theme). Monzo had designed Monzo Stand-in, a separate, minimal backup banking system that keeps essential services running during major outages of Monzo's primary platform. 

# References:

- https://monzo.com/blog/tolerating-full-cloud-outages-with-monzo-stand-in
- https://monzo.com/blog/the-engineering-behind-the-platform
- https://aws.amazon.com/blogs/database/how-monzo-bank-reduced-cost-of-ttl-from-time-series-index-tables-in-amazon-keyspaces/
- https://monzo.com/blog/2016/09/19/building-a-modern-bank-backend
-https://monzo.com/blog/2019/01/14/crowdfunding-technology-backend-architecture
-https://monzo.com/blog/2022/02/17/my-first-6-months-at-monzo-as-a-backend-engineer
- https://www.infoq.com/presentations/monzo-microservices/
- https://www.infoq.com/news/2019/12/network-isolation-kubernetes/

# Extra
- chat system
- IFTTT so you can get real-time notifications
- Flux, so you get receipts in the app
- From payment networks, moving money, maintaining a ledger, fighting fraud, financial crime, and providing world-class customer support.
- direct card processor interacts
- Deployment tool Shipper
- OpenTracing and OpenTelemetry, and open-source tools like Jaeger
- pots
- Monzo Stand-in, to support card payments and transfers