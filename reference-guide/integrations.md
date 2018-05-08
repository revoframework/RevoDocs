# Integrations

## RabbitMQ with Rebus

When using the **Revo.Integrations.Rebus** module, the framework automatically hooks with

```text
<connectionStrings><add name="RabbitMQ" connectionString=
"Url=amqp://guest:guest@localhost:5672;InputQueue=Revo" /></connectionStrings>
```

Using this integration, the application is able to deliver events to other services connected to the same exchange and also receive the events published by those services to them. Further-more, it also makes it possible to offload some of the command and query handling to exter-nal services. As long as the system is configured to route a command or query type to an external exchange, it will always prefer the external transport over the use of local command handler \(if there are registered any\).

The use of this integrations is especially useful when building application using a micro-service architecture. For queries, it is also possible to use worker queues, effectively scaling the load of their processing to multiple query service instances.


