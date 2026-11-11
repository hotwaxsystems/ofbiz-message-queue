# Message queue design

How the plugin meets the requirements with Apache OFBiz trunk.
It learns from Apache Camel's SQS and Pub/Sub components and their fixes, and does not use Camel.

## How a message flows

Incoming:
1. A scheduled job reads new messages from each open incoming channel.
2. It saves each message as a `QueueMessage`, skipping one it already has.
3. It confirms the saved messages with the provider.
4. A second job processes saved messages. It finds the service for each message type and calls it.
5. A failure is tried again later, up to a limit. Then the message is marked failed.

Outgoing:
1. A service calls `sendQueueMessage` with a channel, a message type and a body.
2. The plugin saves the message, then sends it.
3. A job sends again any message that could not be sent, up to a limit.

Saving first means a slow handler never makes the provider send a message again.
It also means every message has a record, whatever happens at the provider.

## Records

### From trunk, used as they are

| Record | Use |
| --- | --- |
| `Enumeration` | Providers, directions and message types |
| `StatusItem`, `StatusValidChange` | The statuses of a message |
| `CustomMethod`, `CustomMethodType` | Name a service in data, for each provider operation and each message type |
| `JobSandbox` | The receive, process and resend jobs |

### Added by the plugin

`QueueChannel`, one queue OFBiz reads from or writes to:

| Field | Meaning |
| --- | --- |
| `queueChannelId` | Id (key) |
| `queueProviderEnumId` | `QP_AWS_SQS`, later `QP_GCP_PUBSUB` |
| `directionEnumId` | `QCD_IN` or `QCD_OUT` |
| `description` | Name shown to people |
| `defaultMessageTypeId` | The message type when a message does not say its own |
| `messageTypeAttribute` | The message property that gives the type, such as `type` |
| `fromDate`, `thruDate` | When the channel is open |

`QueueProviderMethod` names the service for one provider and one operation:

| Field | Meaning |
| --- | --- |
| `queueProviderEnumId` | The provider (key) |
| `customMethodTypeId` | `QUEUE_SEND`, `QUEUE_RECEIVE`, `QUEUE_CONFIRM` or `QUEUE_CHECK` (key) |
| `customMethodId` | The `CustomMethod` naming the provider's service |

`QueueMessageType` names the handler for one kind of message:

| Field | Meaning |
| --- | --- |
| `queueMessageTypeId` | Such as `ORDER_CREATE` (key) |
| `description` | Name shown to people |
| `customMethodId` | The `CustomMethod` naming the handler service, for incoming messages |

`QueueMessage`, one message in or out:

| Field | Meaning |
| --- | --- |
| `queueMessageId` | Id (key). Also the duplicate id sent to providers that need one |
| `queueChannelId` | The channel |
| `queueMessageTypeId` | The kind of message |
| `providerMessageId` | The provider's id. Unique together with the channel |
| `statusId` | See statuses below |
| `body` | The message text, usually JSON |
| `attributes` | The message properties, as JSON |
| `orderingKey` | Messages with the same key stay in order, such as an order id |
| `tryCount` | How many times it was handled or sent |
| `nextTryDate` | When to try again |
| `lastError` | The last error |
| `receivedDate`, `sentDate`, `doneDate` | When it arrived, was sent, or was handled |

Statuses:

| Status | Meaning |
| --- | --- |
| `QMSG_RECEIVED` | Saved, waiting to be handled |
| `QMSG_HANDLED` | Its handler service succeeded |
| `QMSG_TO_SEND` | Saved, waiting to be sent |
| `QMSG_SENT` | The provider accepted it |
| `QMSG_FAILED` | Stopped after the try limit. Support can send it through again |
| `QMSG_CANCELLED` | Support stopped it |

Each provider adds a login record keyed by the channel, such as `QueueChannelAwsSqs`. Secrets use `encrypt="true"`.

## Services

### Called by applications

| Service | In | Does |
| --- | --- | --- |
| `sendQueueMessage` | `queueChannelId`, `queueMessageTypeId`, `body`, `attributes`, `orderingKey` | Saves the message as `QMSG_TO_SEND`, then sends it |
| `retryQueueMessage` | `queueMessageId` | Sends a failed message through again, with its tries reset |
| `checkQueueChannel` | `queueChannelId` | Calls the provider's check. Returns passed and the provider's reason |

### Jobs

| Service | Does |
| --- | --- |
| `receiveQueueMessages` | Reads each open incoming channel, saves new messages, confirms them |
| `processQueueMessages` | Handles saved messages that are due, oldest first, in order by ordering key |
| `resendQueueMessages` | Sends messages still waiting to be sent that are due |

Each job runs one at a time, with the service `semaphore="wait"`.
Each channel runs in its own transaction, so one channel's failure never stops another.

### Contracts every provider follows

| Interface | Ins | Outs |
| --- | --- | --- |
| `queueProviderReceiveInterface` | `queueChannelId`, `maxMessages` | A list of messages: provider message id, body, attributes, confirm handle, delivery attempt |
| `queueProviderConfirmInterface` | `queueChannelId`, confirm handles | The handles that failed |
| `queueProviderSendInterface` | `queueChannelId`, a list of messages | For each message: sent with the provider's id, or failed with a reason |
| `queueProviderCheckInterface` | `queueChannelId` | Passed, and the provider's reason |

The provider's own fields are mapped to these common names.
The receive and send services work on lists, because providers work in groups, such as 10 for SQS.

### Message properties

- The plugin adds `messageType` and `queueMessageId` to every message it sends.
- Internal OFBiz fields never leave with a message.
- Properties follow CloudEvents names where they fit: the id is the provider's message id, the source is the channel, the type is the message type.

## Rules from Camel's experience

- Partial failure of a group is normal. Every send and confirm returns the failed items, and only they are tried again.
- Never hide a "queue does not exist" error. It fails the channel check and is logged.
- Back off after an error, so a broken channel is not called in a tight loop.
- Close provider clients when OFBiz stops. Camel had a memory leak from a long-running reader.
- A message that keeps failing stops at a limit. Camel added this after messages looped forever.

## Security

- Services need the plugin's `MESSAGEQUEUE` permission.
- Secrets are encrypted, never returned, never logged and never searched by.
- Nothing in the plugin knows who owns a channel. An application that needs owners, such as a warehouse serving many merchants, adds that itself.

## Tests

- Automatic tests start a local copy of the provider in a Docker container. Camel tests the same way.
- For SQS, an AWS look-alike container. For Pub/Sub, Google's emulator container.
- No test calls a real cloud account.

## Points for the design review

1. Message type: a channel default, a message property, or both, as above.
2. Try limit and wait between tries: settings per channel, or one setting for all.
3. Order: processing by ordering key keeps order, but one failing message holds back the rest with the same key.
4. Library: the provider's own Java SDK is used for each provider, such as the AWS SDK for SQS. Only that provider's module is added to the build.
5. Job timing: how often the receive job runs, and how long each read waits for messages.
