---
title: SQLServer / NHibernate / Outbox sample
summary: 'How to integrate SQLServer transfer with NHibernate persistence using outbox'
tags:
- SQLServer
- NHibernate
- Outbox
---

 1. Make sure you have SQL Server Express installed and accessible as `.\SQLEXPRESS`. Create two databases: `sender` and `receiver`.
 2. Start the Sender project (right-click on the project, select the `Debug > Start new instance` option). 
 3. Start the Receiver project.
 4. If you see `DtcRunningWarning` log message in the console, it means you have a Distributed Transaction Coordinator (DTC) service running. The Outbox feature is designed to provide *exactly once* delivery guarantees without DTC. We believe it is better to disable the DTC service to avoid confusion when you use Outbox.
 5. In the Sender's console you should see `Press <enter> to send a message` text when the app is ready. 
 6. Hit <enter>.
 7. On the Receiver console you should see that order was submitted.
 8. On the Sender console you should see that the order was accepted.
 9. Finally, after a couple of seconds, on the Receiver console you should see that the timeout message has been received.
 10. Open SQL Server Management Studio and go to the receiver database. Verify that there is a row in saga state table (`dbo.OrderLifecycleSagaData`) and in the orders table (`dbo.Orders`)

## Code walk-through

This sample contains three projects: 

 * Messages - A class library containing the messages.
 * Sender - A console application responsible for sending the initial `OrderSubmitted` message and processing the follow-up `OrderAccepted` message.
 * Receiver - A console application responsible for processing the order message.

Sender and Receiver use different databases, just like in a production scenario where two systems are integrated using NServiceBus. Each database contains, apart from business data, queues for the NServiceBus endpoint and tables for NServiceBus persistence.

### Sender project
 
The Sender does not store any data. It mimics the front-end system where orders are submitted by the users and passed via the bus to the back-end. It is configured to use SQLServer transport with NHibernate persistence and Outbox.

<!-- import SenderConfiguration -->

The Sender uses a configuration file to tell NServiceBus where the messages 
addressed to the Receiver should be sent

<!-- import SenderConnectionStrings -->

### Receiver project

The Receiver mimics a back-end system. It is also configured to use SQLServer transport with NHibernate persistence and Outbox but uses V2.1 code-based connection information API.

<!-- import ReceiverConfiguration -->

In order for the Outbox to work, the business data has to reuse the same connection string as NServiceBus' persistence

<!-- import NHibernate -->

When the message arrives at the Receiver, it is dequeued using a native SQLServer transaction. Then a `TransactionScope` is created that encompasses
 * persisting business data,

<!-- import StoreUserData -->

 * persisting saga data of `OrderLifecycleSaga` ,
 * storing the reply message and the timeout request in the outbox.

<!-- import Reply -->

<!-- import Timeout -->

Finally the messages in the outbox are pushed to their destinations. The timeout message gets stored in NServiceBus timeout store and is sent back to the saga after requested delay of five seconds.