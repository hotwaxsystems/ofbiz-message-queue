# OFBiz Message Queue

A plugin for Apache OFBiz that sends and receives messages through cloud message queues.
AWS SQS is the first provider. Google Pub/Sub follows the same pattern.

## How it works

- Each queue is a channel record: the provider, the queue name and the direction, in or out.
- Every message in or out is saved in OFBiz first. That gives a history, retries and a trace.
- An incoming message calls the OFBiz service named for its message type.
- Every provider follows one shared set of services: send, receive, confirm and check.
- It does not change OFBiz core code.

## Install

Clone this repository into the `plugins` folder of an OFBiz trunk checkout:

```
cd ofbiz-framework/plugins
git clone https://github.com/hotwaxsystems/ofbiz-message-queue.git messagequeue
```

## Contributing

Every change goes through a pull request.

## License

Apache License 2.0. See `LICENSE` and `NOTICE`.
