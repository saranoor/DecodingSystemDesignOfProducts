# Monzo
I recently came across Monzo and decided to deep dive into its architecture and design. Here is the information about Monzo tech stask and high level design that I have found out or infered through Monzo Blog, LinkedIn, InfoQ, and claude(AI). I would be happy to receive any input you have over the flow, architect and resources. Also as the information mentioend in the article is gathered from sources published in prevous years, there is a possibility that Monzo may have evolved. Therefore, I would appreciate if any up to date information is provided as a feedback.


![ScreenShot](/Monzo/monzo_title_diagram.png)



# Functional Requriemnt
- View balance and real-time transaction feed.
- Payments
    - Pay by card (contactless, chip, online) with instant authorisation. https://monzo.com/blog/delightful-payments
    - Send and receive bank transfers (Faster Payments, Bacs, and so on).
    - scheduled payments
    This includes Monzo to Monzo, Monzo to banks, banks to Monzo
- Manage pots, savings, direct debits, standing orders.
- Receive instant push notifications for every transaction.
- Freeze or unfreeze cards, and open accounts with KYC (identity verification).
- Contact support in-app 
- Fraud detection
- undo a payment (https://monzo.com/blog/undo-payments)
- Making an international payment(foreign exchange) [https://monzo.com/blog/building-a-processing-system-for-international-payments]
- Agent chip (https://lnkd.in/p/dDxxEdjh)

# Non Functional Requirement
- High availability: a card authorisation must succeed even if parts of the system are down.
- Correctness and consistency: no double-spend, no lost transaction.
- Low latency: card networks expect a decision within seconds.
- Auditability: every money movement is traceable, as regulators require.
- Security and compliance: zero-trust networking, encryption, and fraud controls.
- Scalability: must handle spikes like salary day.

# Services
- Ledger and accounts service: the source of truth for balances.
- Card processing service: real-time authorisation decisions. (card authorization, card issuer)
- Payments service: bank transfers over external schemes.
- Feed and notification service: the customer-facing view.
- Risk and fraud service: decisions before money moves.

# Capacity estimation
- Total Customers: 15million [https://monzo.com/annual-report/2026]
    - Monzo has published its actual peak numbers: over 2,000,000 reads and 100,000 writes per second on Amazon Keyspaces, across more than 350 TB of data. That is about a 20:1 read:write ratio.

# Design Evolution
- Early stage (2015–2016): Monzo started with a small set of services. By the beta launch the backend had grown to nearly 100 services, and the team reconsidered its architectural choices ahead of the banking licence.
- Scaling stage: by 2019 there were around 1,500 services with over 9,300 inter-service interactions. A 2022 engineering blog post puts it at around 2,000 microservices. In 2024, it was published that there are 2800.(https://monzo.com/blog/how-we-run-migrations-across-2800-microservices)


# Tech stack
- Go: 
Why GO? According to the presentation by Matt Heath and Suhail Patel[https://www.infoq.com/presentations/monzo-microservices/] Go it's quite simple and It's statically typed. It makes it quite easy for Monzo to get people on board. Go also has some interesting things such as a backwards compatibility guarantee. Monzo have been using Go from the very early versions of Go 1. Every time a new version of Go comes out, it has a guarantee that code can be recompiled, and  basically get all of the improvements. What that means is the garbage collector, for example, has improved several orders of magnitude over the time that Monzo've had their infrastructure running. Every time Monzo recompile it, test that it still works and just get those benefits for free.

- Cassandra/ Amazon Keyspaces (a managed, Cassandra-compatible service):
Why Cassandra: Monzo uses Cassandra because it is horizontally scalable, with no single write bottleneck, it is highly available across nodes and zones and suits high-volume, append-heavy data such as transactions. Cassandra  provides a masterless, horizontally scalable database that gives user a lot of controls over writing the data to multiple locations. Monzo don't have this one server, a primary that fails over to a secondary. People assume banks need strict all-or-nothing transactions (ACID). Monzo's engineers argue that real banking is often more relaxed: the system stays available and becomes consistent a little later (BASE). The cost of having high availability is Engineers must build their own safety (locks, idempotency, reconciliation). So, Monzo use strict control where the risk is high, and looser, faster methods where it is safe. SO Monzo does implement locks: it locks the account, does the work, then unlocks. For others, such as late-arriving card payments, they don't need a lock. They put the payments in a queue and apply them to accounts one by one. 
In sum, Monzo chose Cassandra because it never depends on one server and scales easily, and they handle the "bank-level correctness" part themselves with locks and careful design.

- Kafka
- Kubernetes and Docker 
- Envoy Proxy for RPC
- AWS for most of production infrastructure and GCP for most of data infrastructure
- React for Monzo public web apps and internal tools
- Prometheus, Grafana, The Elastic Stack
- etcd for distributed locking
- OpenTracing and OpenTelemetry, and open-source tools like Jaeger
- pots

# Architect
- Monzo follow Microservice based Architecture. Single responsibility per service, clear bounded contexts (cards, payments, accounts, notifications), and independent deployment.

- Monzo uses a hybrid architecture that relies on both synchronous and asynchronous patterns, depending on the specific task being performed. Services communicate by RPC for synchronous calls and by events for asynchronous flows.

# High Level Design
Monzo high level architecture is illustrated as follow:

![ScreenShot](/Monzo/mono_high_level_architect.png)

# Flow
The ledger, card authorization flow and payment is illustrative in the diagram below. This is my own inference, I have not come across any publicly available resource which verifies these illustration.

## The ledger flow is illustrated by the flow chart below:
![ScreenShot](/Monzo/ledger_flow.png) 


## The Card authorization flow is illustrated by the diagram below:
![ScreenShot](/Monzo/card_authorization.png)


## The payment state machine is illustrated by the diagram below:
![ScreenShot](/Monzo/payment_state_machine.png)

# Challenge
1. For resilience against total platform failure (the biggest theme), Monzo had designed Monzo Stand-in(https://monzo.com/blog/tolerating-full-cloud-outages-with-monzo-stand-in), a separate, minimal backup banking system that keeps essential services running during major outages of Monzo's primary platform. 

# References:
- https://monzo.com/blog/tolerating-full-cloud-outages-with-monzo-stand-in
- https://monzo.com/blog/the-engineering-behind-the-platform
- https://aws.amazon.com/blogs/database/how-monzo-bank-reduced-cost-of-ttl-from-time-series-index-tables-in-amazon-keyspaces/
- https://monzo.com/blog/2016/09/19/building-a-modern-bank-backend
-https://monzo.com/blog/2019/01/14/crowdfunding-technology-backend-architecture
-https://monzo.com/blog/2022/02/17/my-first-6-months-at-monzo-as-a-backend-engineer
- https://www.infoq.com/presentations/monzo-microservices/
- https://www.infoq.com/news/2019/12/network-isolation-kubernetes/
- https://monzo.com/blog/engineering-the-future-of-customer-operations-the-monzo-ops-agent
- https://monzo.com/blog/2022/02/08/processing-payments-safely-at-scale
- https://monzo.com/blog/2022/02/18/how-we-calculate-balances
- https://www.infoq.com/articles/cassandra-kubernetes-microservices/

# Questions
- It is mentioned in the article Monzo run some SQL queries for checking coherence, however, according to my understanding Mozo uses Cassandra https://monzo.com/blog/2022/02/08/processing-payments-safely-at-scale so my question is there a relational database used in Monzo?