# RabbitMQ in the Project (Spring AMQP Flow)

## Why & flow
Sync email call slows checkout and fails if mail server is down. Instead the Order Service drops a **message** and moves on.

```
Order Service (Producer)
   │ publish(msg, routingKey="order.created")
   ▼
Exchange "order.exchange" (topic) ── binding "order.*" ──► Queue "email.queue" ──► Email Consumer
                       └───────── binding "order.created" ─► Queue "inventory.queue" ─► Inventory Consumer
Consumer: process → ACK (broker deletes msg)   |   fail → NACK/retry → after N tries → Dead Letter Queue
```
Exchange types: **direct** (exact key), **topic** (patterns `order.*`, `#`), **fanout** (all queues), **headers**. Full theory in your `RabbitMQ.md`.

## Spring AMQP code
```java
@Configuration
class RabbitConfig {
    @Bean TopicExchange exchange() { return new TopicExchange("order.exchange"); }
    @Bean Queue emailQueue() {
        return QueueBuilder.durable("email.queue")
            .withArgument("x-dead-letter-exchange", "dlx.exchange").build();
    }
    @Bean Binding binding(Queue emailQueue, TopicExchange exchange) {
        return BindingBuilder.bind(emailQueue).to(exchange).with("order.created");
    }
    @Bean Jackson2JsonMessageConverter converter() { return new Jackson2JsonMessageConverter(); }
}

// Producer
rabbitTemplate.convertAndSend("order.exchange", "order.created", new OrderCreatedEvent(id, email));

// Consumer
@RabbitListener(queues = "email.queue")
public void handle(OrderCreatedEvent e) {
    if (processedRepo.exists(e.eventId())) return;    // idempotency
    mailService.send(e.email());                       // exception → retry → DLQ
}
```
```yaml
spring.rabbitmq.listener.simple:
  acknowledge-mode: auto        # ack after listener returns without exception
  prefetch: 10                  # fair dispatch
  retry: { enabled: true, max-attempts: 3, initial-interval: 2000 }
```
Reliability checklist: durable queue + persistent messages · publisher confirms · manual/auto ACK correctly · DLQ · idempotent consumer · **outbox pattern** (DB row + message atomic) · monitoring queue depth.
