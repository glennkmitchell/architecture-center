# Data considerations for microservices

![](./images/data-considerations.png)


A basic principle of microservices is that each service manages its own data. Two services should not share a database. A service is responsible for its own private database, which other services cannot access directly.

![](../guide/architecture-styles/images/cqrs-microservices-wrong.png)

The reason for this rule is to avoid tight coupling between services. We want every service to be updated and deployed independently, and sharing a database creates a dependency. If there is a change to the data schema, the change must be coordinated across every service that relies on that database. By isolating each service's data store, we can limit the scope of change, and preserve the agility of truly independent deployments.

> It's fine for services to share the same physical database server. The problem occurs when services shared the same schema, or read and write to the same set of database tables.

A second reason to avoid shared databases is that each service may have its own data models, queries, or read/write patterns. Using a shared database would limit each team's ability to optimize data storage for their particular service. 

This approach naturally leads to [polyglot persistence](https://martinfowler.com/bliki/PolyglotPersistence.html) &mdash; the use of multiple data storage technologies within a single application. One service might require the schema-on-read capabilities of a document database. Another might need the referential integrity provided by an RDBMS. Each team is free to make the best choice for their service. For more about the general principle of polyglot persistence, see [Use the best data store for the job](../guide/design-principles/use-the-best-data-store). 

## Challenges

Some challenges arise from this approach to managing data. There may be redundancy across the data stores, and the same item of data may appear in mulitple places. For example, data might be stored as part of a transaction, then stored elsewhere for analytics, reporting, and archiving. This can lead to issues of data integrity and consistency. 

Traditional data modeling uses the rule of "one fact in one place." Every entity appears exactly once in the schema. Other entities may hold references to it but not duplicate it. The obvious advantage to the traditional approach is that updates are made in a single place. With microservices, you have to think about how updates are propaged across services. 

For example, consider the case of a customer creating an order. In a monolithic application, the flow might be as follows: The front end receives a request and creates an order entry in a database. The backend queries the same database to pick up the new orders. In microservices, you would avoid this pattern. Instead, the front-end service might send a message to a back-end service. In general, asynchronous messaging patterns become more important when services do not share a database.

## Approaches to managing data

There is no single approach that's correct in all cases, but here are some general guidelines for managing data in a microservices architecture.

- Embrace eventual consistency where possible. Understand the places in the system where you need strong consistency or ACID transactions, and the places where eventual consistency is acceptable.

- When you need strong consistency guarantees, one service may represent the source of truth for a given entity, which is exposed through an API. Other services might hold their own copy of the data, or a subset of the data, that is eventually consistent with the master data but not considered the source of truth. For example, imagine a customer order service and a recommendation service. The recommendation service might listen to events from the order service, but if a customer requests a refund, it is the order service, not the recommendation service, that has the complete transaction history.

- For transactions, use patterns such as [Scheduler Agent Supervisor](../patterns/scheduler-agent-supervisor) and [Compensating Transaction](../patterns/compensating-transaction) to keep data consistent across several services.  You may need to store an additional piece of data that captures the state of a unit of work that spans multiple services, to avoid partial failure among multiple services. For example, keep a work item on a durable queue while a multi-step transaction is in progress. 

- Avoid duplicating data unnecessarily. A service might only need to store a subset of information about a domain entity. For example, in the delivery bounded context, we need to know which customer is associated to a particular delivery. But we don't need the customer's billing address - that's handled in the Accounts bounded context. Thinking carefully about the domain, and using a DDD approach, can help here. 

- Consider whether your services are coherent and loosely coupled. If two services are continually exchaning information with each other (leady to chatty APIs), it's possible you should redraw your service boundaries, and merge two services together or refactor their functionality.

- Use an [event driven architecture](../guide/architecture-styles/event-driven.md). When a service updates a data store, it sends an event. Interested services can subscribe to this event. For example, a service can listen to events and construct a materialized view that is suitable for querying.

- A service that sends events should publish a schema that can be used to automate serializing and deserializing events. Consider JSON schema or a framework like [Microsoft Bond](https://github.com/Microsoft/bond). Think about how you will version the event schema. At high scale, events can become a bottleneck on the system, so consider using aggregation or batching to reduce the total load. 





