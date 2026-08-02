# Simple Queue Service (SQS) and Simple Notification Service (SNS)

## Motivation for queueing systems

As you may know, whenever a user sends a message in the PolyAI chat, an HTTP request is sent from the `frontend` to the `agent` service with the full message context (user id, content, timestamp, conversation id, etc...).

If the message contains an image, the agent calls the `yolo` service, which detects the objects in the image and returns the result to the agent, which in turn answers the user.

Besides calling the Yolo service, there are a few more microservices in the system who might be interested in the message, such as:

- Metering service: Tracks user interactions for billing (tokens, processed images).
- Analytics service: Collects and analyzes user interactions for insights.
- Security service: Monitors user interactions for security threats (e.g. prompt injection attempts).

In a **naive** approach, the agent service could directly send HTTP requests to each of these microservices with the event information:

```
                ┌──────────────┐
           ┌───►│     yolo     │
           │    └──────────────┘
           │    ┌──────────────┐
┌───────┐  ├───►│   metering   │
│ agent │──┤    └──────────────┘
└───────┘  │    ┌──────────────┐
           ├───►│  analytics   │
           │    └──────────────┘
           │    ┌──────────────┐
           └───►│   security   │
                └──────────────┘
```

However, this **synchronous communication** pattern introduces several challenges:

- Each HTTP request adds latency, making the system slower and less responsive.
- As the number of microservices grows, the complexity and potential for failure increase, leading to a tightly coupled system that is difficult to scale and maintain.

## Message brokers: The Producer/Consumer model 

To address these issues, we can introduce a **message broker system**: 

```
┌───────┐  produce   ┌────────────────────┐  consume   ┌──────────┐
│ agent │───────────►│ queue userMessage  │───────────►│   yolo   │
└───────┘            └────────────────────┘            │   (xN)   │
                                                       └──────────┘
```

The agent service **produce** the `userMessage` event to a **queue**.
The Yolo microservice(s) can then **consume** the event from the queue **at its own pace**. 

This kind of **asynchronously communication** decouples the services, reduces latency, and simplifies the overall architecture, making the system more scalable, resilient, and easier to manage.

This kind of queuing system is also known as a **message broker**, in our case, we'll use a simple message broker manages by AWS, called **SQS**.

This is a very common design in event driven and microservices architectures which can significantly improve the scalability and reliability of the system:  

- **Scalability:** Multiple consumers can concurrently process messages from the queue (as can be seen in the above figure for the Yolo service), allowing the system to handle high loads efficiently.
- **Fault Tolerance:** If a consumer fails to process an event, the message is returned to the queue for another attempt. 


## Create a standard queue

1. Open the Amazon SQS console at https://console.aws.amazon.com/sqs/

1. Choose **Create queue**.

1. For **Type**, the **Standard** queue type is set by default. 

1. Enter a **Name** for your queue, e.g. `polyai-chat-messages`.

1. Under Configuration, set values for parameters according to the following characteristics:

    - We let consumer `60 seconds` to process a single message.
    - The amount of time a single message will be stored in the queue until it's processed by one of the consumers is `1 day`.
    - We don't want any delay between the time the message is produced to the queue to the time it is consumed.
    - We know that the maximal message size is `256KB`.

2. Choose the **Basic Access policy**.
1. Choose **Create queue**. Amazon SQS creates the queue and displays the queue's Details page.

## Integrate your queue in the PolyAI agent

Here is a simple Python code to produce a message to a given queue: 

```python
import boto3
import json
from botocore.exceptions import ClientError

sqs = boto3.client('sqs', region_name='us-east-1')  # Change region as needed
QUEUE_URL = 'https://sqs.us-east-1.amazonaws.com/123456789012/polyai-chat-messages'

def produce_message(message_body: dict):
    try:
        response = sqs.send_message(QueueUrl=QUEUE_URL, MessageBody=json.dumps(message_body))
        print(f"Message sent successfully. MessageId: {response['MessageId']}")
        
        # send to the client - "your message is being processed...."
        
    except ClientError as e:
        print(f"Error sending message: {e}")
        # send to the client - "Opps, something went wrong. Please try again later."
```

And a example of consumer code: 

```python
import boto3
import time
import json

sqs = boto3.client('sqs', region_name='us-east-1')
QUEUE_URL = 'https://sqs.us-east-1.amazonaws.com/123456789012/polyai-chat-messages'

while True:
    response = sqs.receive_message(
        QueueUrl=QUEUE_URL,
        MaxNumberOfMessages=5,
        WaitTimeSeconds=20
    )
    
    messages = response.get('Messages', [])
    
    for msg in messages:
        msg_body: dict = json.loads(msg['Body'])
        print(f"Handling message: {msg_body}")
        
        # Delete the message when done processing it
        sqs.delete_message(QueueUrl=QUEUE_URL, ReceiptHandle=msg['ReceiptHandle'])
        print(f"Message processed: {msg['MessageId']}")
        
    if not messages:
        # No messages, waiting...
        time.sleep(1)
```

You can even try to run multiple instances of the above consumer, as if the Yolo service is running multiple instances. What is the expected behaviour? 

## Message brokers: The Publisher/Subscriber model 

So far, we've seen how a single microservice consumes messages from the queue.

What about scenarios where multiple services need to consume the same event simultaneously? Why isn't a single SQS queue suitable for this?

Let's address this by dedicating a **separate queue** for each microservice aimed at handling specific events:

```
                         ┌────────────────────┐    ┌──────────────┐
                    ┌───►│ queue: yolo        │───►│     yolo     │
                    │    └────────────────────┘    └──────────────┘
┌───────┐           │    ┌────────────────────┐    ┌──────────────┐
│ agent │───────────┼───►│ queue: metering    │───►│   metering   │
└───────┘           │    └────────────────────┘    └──────────────┘
                    │    ┌────────────────────┐    ┌──────────────┐
                    └───►│ queue: analytics   │───►│  analytics   │
                         └────────────────────┘    └──────────────┘
```

But now how can the agent efficiently produce the same event message to multiple queues?
Additionally, what if a microservice is interested only in a subset of the events?
For example, the Metering service might only be interested in `userImage` events, while the Analytics service might be interested in `userTextMessage` events as well.

The naive approach is to manually send the same event message to each queue, which introduces complexity and potential inconsistencies.

Here **SNS** comes in. 

SNS allows you to **publish** messages to a **topic**. The messages can be delivered to a large number of **subscribers** (a.k.a. **pub/sub**) in parallel.
In our case, the subscribers are the SQS queues. 

```
                                      ┌────────────────────┐   ┌──────────────┐
                                 ┌───►│ queue: yolo        │──►│     yolo     │
                                 │    └────────────────────┘   └──────────────┘
┌───────┐   ┌──────────────────┐ │    ┌────────────────────┐   ┌──────────────┐
│ agent │──►│ SNS topic        │─┼───►│ queue: metering    │──►│   metering   │
└───────┘   │ polyai-messages  │ │    └────────────────────┘   └──────────────┘
            └──────────────────┘ │    ┌────────────────────┐   ┌──────────────┐
                                 └───►│ queue: analytics   │──►│  analytics   │
                                      └────────────────────┘   └──────────────┘
```


> [!NOTE]
> SNS is a fully managed messaging service for both application-to-application (A2A) and application-to-person (A2P) communication. Subscribers can be SQS queues, Lambda functions, HTTP endpoints, and more, ensuring that multiple services can react to the same event in parallel.


## To summarize

We wanted to send the `userMessage` event in our app to multiple microservices.

Using the naive approach, we would have needed to manually initiate an HTTP request to each microservice, which is slow, cumbersome and not scalable.

Now, using a single HTTP request from the PolyAI agent to the SNS topic, we would be able to efficiently publish the event to as many microservices as we want:  

- **SNS**: Used to broadcast the event to multiple SQS queues.
- **SQS**: Each microservice has its own queue to process the events independently.


