# RabbitMQ

We use RabbitMQ for:

* **Message queues (also called "work queues")** (e.g., computation tasks): 1 durable queue where messages are processed only once (in nominal cases), even with multiple consumers.
* **Event buses (also called "pub/sub")** (e.g., cancelling a computation): as many exclusive auto-delete queues as there are consumers.

Regardless of specific concrete issues, we decided to use **Spring Cloud Stream** to add an abstraction layer since we are in the Spring ecosystem, hoping to gain the following advantages:

* **RabbitMQ / Kafka abstraction:** (almost) the exact same code can run on either broker. *(NOTE: We even ran Gridsuite on Kafka back in 2021 and it worked. However, in 2024 we added RabbitMQ-specific features like Quorum Queues, so it won't work out of the box anymore).*
* **Higher-level, simpler, and better Spring integration** compared to using AMQP directly (even though `spring-amqp` exists... and in fact, Spring Cloud Stream builds on top of `spring-amqp`). For instance:
  * **Event bus:** standard default mode in Spring Cloud Stream.
  * **Work queues:** handled via Spring Cloud Stream `consumergroup`.
  * **Declarative configuration:** queues defined in YAML, auto-provisioned in the broker.
  * **Ultralight API:** just a single bean to consume, and a simple `StreamBridge` call to produce.
  * **Unit testing:** based on an in-memory test binder / mock broker.
  * **Pre-configured default behavior** (e.g., concurrency settings) *Note: As of 2026, this is actually backfiring and causing load-balancing issues (see the following sections).*

## Sections

- [RabbitMQ Work Queue Load Balancing](rabbit-consumer-load-balancing.md)
- [RabbitMQ Error Handling](error-handling.md)
