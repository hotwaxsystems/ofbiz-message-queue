# Implementation plan

How we build the message queue plugin, AWS SQS first.
Each numbered task is one GitHub issue.

The plan has two parts:

1. **Shared frame.** Built once, used by every provider.
2. **Provider template.** Repeated for each provider. AWS SQS is the first.

It builds on the [requirements](requirements.md) and the [design](design.md).

## Rules for every task

These come from Apache Camel's SQS and Pub/Sub components, their fixes, and earlier HotWax queue work.

- Save every message before it is confirmed, handled or sent.
- Treat every message as one that may arrive twice. The channel and provider message id find a duplicate.
- Expect part of a group to fail. Retry only the failed items.
- Stop a failing message at the try limit. Never loop forever.
- Never hide a provider error such as "queue does not exist".
- Back off after an error.
- Close provider clients when OFBiz stops.
- Never write a secret to a log, a message or a test file. Never put keys in seed data.
- Test data uses made-up businesses and queues only.
- Every change goes through a pull request.

## Part 1. Shared frame

### 1. Records and seed data

Build:
- `QueueChannel`, `QueueProviderMethod`, `QueueMessageType` and `QueueMessage`, with a unique index on channel and provider message id.
- Enumerations for providers and directions.
- Message statuses and their allowed changes.
- Operation types `QUEUE_SEND`, `QUEUE_RECEIVE`, `QUEUE_CONFIRM` and `QUEUE_CHECK`, and the handler type, as `CustomMethodType` rows.

Done when the plugin loads on OFBiz trunk, and the records and seed data load with no errors.

### 2. Provider contracts and channel check

Build:
- The four provider interfaces: receive, confirm, send and check.
- `checkQueueChannel`. It finds the provider's check service and calls it.

Done when a test provider, made only for tests, is called through `checkQueueChannel`.

### 3. Receive and process

Build:
- `receiveQueueMessages`: read each open incoming channel, save new messages, skip duplicates, confirm the saved ones.
- `processQueueMessages`: handle due messages in order by ordering key, retry with a wait, stop at the limit.
- The message type from the channel default or the message property.
- Each channel in its own transaction.

Done when tests with the test provider pass for: one message, many messages, a duplicate, a failing handler, and the try limit.

### 4. Send and resend

Build:
- `sendQueueMessage`: save, then send.
- `resendQueueMessages`: send what is still waiting, retry only the failed items, stop at the limit.
- `retryQueueMessage` for Support.
- The `messageType` and `queueMessageId` properties on every sent message.

Done when tests with the test provider pass for: one send, a group with some failed, provider down, and a retry by Support.

### 5. Jobs and test kit

Build:
- `JobSandbox` seed rows for the three jobs, each running one at a time.
- A test kit that starts a local provider in Docker before a test and stops it after.
- Shared test data: a made-up business with one incoming and one outgoing channel.

Done when the jobs run on a local OFBiz, and a test can start and stop a local provider.

## Part 2. Provider template: AWS SQS

Tasks 6 to 11 carry the label `provider template`. The next provider copies them.

### 6. Provider seed data and login record

Build:
- The `QP_AWS_SQS` provider and its `QueueProviderMethod` rows.
- `QueueChannelAwsSqs`: region, queue URL, access key id, secret key (encrypted), strict-order queue flag, and an address for a local copy.
- One SQS client per channel, built from that record. It is closed when OFBiz stops or the channel changes.
- Only the AWS SDK's SQS module is added to the build.

Done when a channel record builds a client that reaches a local SQS copy.

### 7. Check

Build `awsSqsCheckChannel`. It reads the queue's attributes. A missing queue or a wrong key fails the check with AWS's reason.

Done when tests pass for: check passed, wrong secret, queue does not exist.

### 8. Receive and confirm

Build:
- `awsSqsReceive`: read up to 10 messages per call, waiting a few seconds for messages to arrive. Return the message id, body, attributes, receipt handle and receive count.
- `awsSqsConfirm`: delete the saved messages in groups of 10. Return the ones that failed.

Done when tests pass for: one message, 25 messages, a message received twice, and a delete that partly fails.

### 9. Send

Build `awsSqsSend`:
- Send in groups of 10. Return each message as sent with its SQS id, or failed with a reason.
- Send at most 10 properties per message. Fail a message with more, with a clear reason.
- For a strict-order queue, set the group id from the ordering key and the duplicate id from `queueMessageId`. Refuse a message with no ordering key.

Done when tests pass for: one send, 25 sends, part of a group refused, a strict-order queue, and too many properties.

### 10. Scenario tests

Build one test for each scenario in the requirements, against a local SQS copy.

Done when every scenario in the requirements has a passing test.

### 11. Manual test on a real SQS queue

Needs an AWS test account.

Do:
- Set up a test queue and a channel in a local OFBiz.
- Run each scenario against the real queue.
- Note any difference from the local copy, and add a test for it.

Done when each scenario passes against real SQS, and the results are noted in the issue.

## Order

Tasks 1 to 5 come first. Tasks 6 to 10 follow in order. Task 11 waits for an AWS test account.
Google Pub/Sub comes next, with the same template tasks.
