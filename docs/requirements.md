# Message queue requirements

## Scope

This plugin lets an OFBiz business exchange messages with other systems through cloud message queues.
AWS SQS comes first. Google Pub/Sub follows.

A message queue is a mailbox in the cloud. One system drops a message in it. Another system picks it up later.
A channel is one queue that OFBiz reads from or writes to, with its login details.

## Actors

- **Integration Admin** sets up channels and says which OFBiz service handles each kind of message.
- **Partner System** puts messages on a queue, or reads the messages OFBiz sends. Example: a sales channel that sends orders.
- **Queue Provider** (AWS SQS first) holds the messages until they are read.
- **Support** looks at failed messages and sends them through again.
- **Developer** adds a new queue provider to the plugin.

## Business process story

### Set up a channel

1. Integration Admin records a channel: the provider, the queue name, the direction (in or out), and the login details.
2. Company keeps the secret safe. Nobody can read it back after it is saved.
3. Company checks the channel with the provider and records the result.
4. Integration Admin names the kind of message each incoming channel carries, or the message property that gives the kind.
5. Integration Admin names the OFBiz service that handles each kind of message, such as "new order".

### Receive messages

6. Partner System puts a message on its queue.
7. Company reads new messages from each incoming channel every few seconds.
8. Company saves each message, then confirms it with the provider, so the provider does not send it again.
9. Company skips a message it has already saved, so a message sent twice is handled once.
10. Company calls the service for the message's kind and records the result.
11. If the service fails, Company tries again a few times. Then it marks the message failed.
12. Support looks at a failed message, fixes the cause, and sends it through again.

### Send messages

13. An OFBiz service asks Company to send a message of a kind on a channel, such as "shipment shipped".
14. Company saves the message, then sends it to the provider.
15. If the provider refuses or cannot be reached, Company tries again a few times. Then it marks the message failed.
16. Messages that must stay in order, such as all updates of one order, are sent in order.

### Add a provider

17. Developer adds a new provider by following the same shared steps: check, receive, confirm and send.
18. Integration Admin can then set up a channel for the new provider with no other change to OFBiz.

## Rules

- Every message in or out is saved in OFBiz before anything else happens to it.
- A message is confirmed with the provider only after it is saved.
- A message with the same provider message id on the same channel is saved once.
- A secret is never shown, logged or stored in a message.
- A failed message keeps its last error and its number of tries.
- A message that keeps failing stops after a set number of tries. It never loops forever.
- Part of a group of messages can fail while the rest succeed. Only the failed ones are tried again.
- One channel's failure never stops another channel.
- A stopped channel is not read and not sent to.
- Test data uses made-up businesses and queues only.

## Scenarios

| # | Scenario | Example | Outcome |
| --- | --- | --- | --- |
| 1 | Add a channel | Acme adds the SQS queue `acme-orders` as incoming | Channel saved. Check passed |
| 2 | Wrong secret | The secret is typed wrong | Channel saved. Check failed, with the provider's reason |
| 3 | Receive one | One order message arrives on `acme-orders` | Saved, confirmed, handled by the order service |
| 4 | Receive many | 25 messages arrive at once | All 25 saved and handled |
| 5 | Sent twice | The same message arrives twice | Saved and handled once |
| 6 | Handler fails | The order service fails on a missing product | Tried again, then marked failed with the error |
| 7 | Send again | Support fixes the product and sends the message again | Handled. Message marked done |
| 8 | Send one | A shipment ships | One "shipment shipped" message sent on `acme-updates` |
| 9 | Part of a send fails | 25 messages are sent and 2 are refused | 23 sent. Only the 2 are tried again |
| 10 | Provider down | The provider cannot be reached for 10 minutes | Messages wait and are sent when it is back |
| 11 | In order | Three updates of order 1022 are sent | They arrive in the order they were sent |
| 12 | Stopped channel | Acme's channel is stopped | Nothing read or sent on it |

## Out of scope for now

- Google Pub/Sub, Azure Service Bus, Kafka and other providers.
- Creating queues or topics from OFBiz. They are made in the provider's console.
- Messages larger than the provider allows.
- A screen for Support. The services come first.

## Open question

- How many times should a failed message be tried before it is marked failed?
