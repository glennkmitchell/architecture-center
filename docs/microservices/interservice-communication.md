# Interservice communication

There are two main ways that services can communicate with other services:

- **Synchronously** by calling APIs on the service.
- **Asynchronously** by sending messages. 

AsyMost queue solutions have reliability built-in - guaranteed delivery of durable messages

## When to use asynchronous message queuing

Asynchronous message queues have some advantages that can be very useful in a microservices architecture.

- Reduced coupling between services. The message sender does not need to know about the consumer. 
- Multiple subscribers. Using a pub/sub model, multiple consumers can subscribe to receive events. See [Event-driven architecture style](/azure/architecture/guide/architecture-styles/event-driven).
- Failure isolation. If the consumer goes down, the sender can still continue to send messages. They will be picked up when the consumer recovers. (Depending on the scenario, you may need logic to handle stale messages.)
- Asynchronous operations. The message sender does not have to wait for the consumer to respond. This is especially useful in a microservices architecture. If there is a chain of service dependencies (service A calls B, which calls C, and so on), waiting on synchronous calls can add unacceptable amounts of latency.
- Load leveling. A queue can act as a buffer to level the workload. 
- Workflows. Queues can be used to manage a workflow, by check-pointing the message after each step in the workflow.

However, there are some tradeoffs with using messages to communicate. Here are some potential challenges:

- Using a particular messaging infrastructure may cause tight coupling. It will be difficult to switch to another messaging infrastructure later.
- The messaging infrastructure incurs additional cost. At high throughputs, the cost could become significant.
- Handling asynchronous messaging is not a trivial task. For example, you must be able handle duplicated messages, either by duplicating or by making operations idempotent. 
- A message queue can become a bottleneck in the system. Each operation requests at least on queue and dequeue operation, and queue semantics generally require some kind of locking inside the messaging infrastructure. If the queue is a managed service, there may be additional latency because the queue endpoint is outside of the cluster’s virtual network. You can mitigate these issues by batching messages, but that complicates the code. 
- Message queues don't work well for request-response semantics. You need to configure a reply queue, with a message correlation ID.

## API design

When you think about designing APIs for a microservice, it's important to distinguish between two types of API:

- A public API that client applications call. 
- Backend APIs that are used for interservice communication.

These two use cases have somewhat different requirements. The public API must be compatible with client applications, typically browser applications or native mobile applications. Most of the time, that means the public API will be REST over HTTP. For the backend APIs, however, you need to take network performance into account. Depending on the granularity of your services, interservice communication can result in a lot of network traffic. Services can quickly become I/O bound. For that reason, considerations such as serialization speed and payload size become more important.

Technology choices. You have to consider several aspects of how an API is implemented:

- **REST or RPC interface**. For a RESTful interface, the most common choice is REST over HTTP using JSON. For an RPC-style interface, there are several popular frameworks, including gRPC, Appache Avro, and Apache Thrift.  

- **Interface definition language (IDL)**. An IDL is used to define the methods, parameters, and return values of an API. An IDL can be used to generate client code, serialiation code, API documentation, or to run automated API tests. Frameworks such as gRPC, Avro, and Thrift define their own IDL specifications. REST over HTTP does not have a standard IDL format, but a common choice is OpenAPI (formerly Swagger). You can also create an HTTP REST API without using a formal defininition language, but then you lose the benefits of code generation and automated testing.

- **Serialization format**. This defines how are objects are serialized over the wire. Options include JSON and XML, which are text-based, or binary formats such as protocol buffer. 

- **Transport**. The underlying transport protocol is usually either HTTP or HTTP/2.

In some cases, you can mix and match options. For example, by default gRPC uses protocol buffers for serialization, but it can use other formats such as JSON.

| &nbsp; | HTTP REST | gRPC | Apache Avro | Apache Thrift |
|--------|-----------|------|-------------|---------------|
| **IDL** | OpenAPI | .proto file | Avro schema (JSON) | Thrift IDL |
| **Serialization** | protocol buffer (default) | JSON, XML, other media types | Binary, JSON | Binary, JSON |
| **Transport** | HTTP/2 | HTTP, HTTP/2 | HTTP | HTTP |

Considerations:

- Tradeoffs between a REST-style interface or an RPC-style interface.
- Does the serialization format require a fixed schema? If so, do you need to compile a schema file?
- Framework and language support. HTTP is supported in nearly every framework and language. gRPC, Avro, and Thrift all have libraries for C++, C#, Java, and Python. Thrift and gRPC also support Go.
- Tooling for generating client code, serializer code, API documentation, etc. 
- Serialization efficiency in terms of speed, memory, and payload size.
-  If you are using a service mesh, is the protocol compatible? For example, linkerd has built-in support for gRPC.
- How will you version the APIs and data schemas?
- If you choose a protocol like gRPC, you may need a protocol translation layer between the public API and the back end. A gateway can perform that function.

Our recommendation is to choose REST over HTTP as a baseline, unless you need the performance of a binary protocol. Advantages of REST over HTTP include: 

- It requires no special libraries. 
- It is the most decoupled option, because callers do not need a client stub to communicate with the service.
- It can be used by browser clients, so you don’t need protocol translation between the client and the backend.

