MQLess is a lightweight actor model framework on top of AWS Lambda.

## What is Actor Model?

The Actor Model provides a higher level of abstraction for writing concurrent and distributed systems. It alleviates the developer from having to deal with explicit locking and thread management, making it easier to write correct concurrent and parallel systems.

## What MQLess adds on top of AWS Lambda?

With Actor Model, every actor has a mailbox of messages, and it processes the messages one by one.
With AWS Lambda doesn't have a mailbox, and AWS process the requests in parallel.

MQLess implement the mailboxes in front of AWS Lambda, guaranteeing that each actor process one message at a time.
