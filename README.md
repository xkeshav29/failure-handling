# Problem

When processing the message from any user fails, the entire queue is blocked which means none of the users recieve response from the Job Finder. This implies downtime for all the users.

Once the error is fixed, the consumption from the queue is resumed resulting in delivering the next instruction for all messages in the queue.

This has following downsides:
1. Bad User Experience waiting for next instruction for long
2. Job seekers might drop off from the point when the response took too long.

# Desired State

Error processing message from some Users shoudl not cause downtime for all users.


# Proposed Solution

```
Let's define *Blocked Users* as list of Users whose message processing encountered error that can be fixed on retry.

Add these 2 componenets in the infra:
  1. RETRY QUEUE: Contains the messages from Blocked Users in the same order in which they arrived.
  2. REDIS CACHE: Contains all user ids of all Blocked Users

In order to unblock the queue when any error is encountered while processing a message, we push the message to the retry queue and save the user id in redis.
```

The consumer of Primary Queue does the following on cosuming a message:

```javascript
if the message is from a Blocked User {
  RETRY_QUEUE.push(message)
  EXIT
}
try {
  process(message)
} catch(recoverable error) {
  REDIS CACHE.add(message.user_id)
  RETRY_QUEUE.push(message)
} catch(irrecoverable error) {
  send DEFAULT_INSTRUCTION(I couldn't understand you)
}
```
  
The consumer of Retry Queue does the following on consuming a message:

```javascript
try {
  process(message)
  REDIS CACHE.remove(message.user_id)
} catch(recoverable error) {
  RETRY_QUEUE.push(message)
} catch(irrecoverable error) {
  send DEFAULT_INSTRUCTION(I couldn't understand you)
}
```

# Architechture Diagram

![https://raw.githubusercontent.com/xkeshav29/failure-handling/master/Architechture.png]


