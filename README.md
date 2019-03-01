# Ascend.Queue
Ascend.Queue is a library intended to interface with preferred queuing technologies used by Ascend Learning.  The current implementation v.0.0.1.275 communicates strictly with Kafka
## Terms

 - **Subscriber** - The application that is listening or checking for updates from the queue.  In Kafka terms, it is synonymous with Consumer.  More about Kafka Consumers can be found [here](https://kafka.apache.org/documentation/#intro_consumers).
 - **Publisher** - The application that will be writing information to the queue.  In Kafka terms, it is synonymous with Producer.  More about Kafka Producers can be found [here](https://kafka.apache.org/documentation/#intro_producers)

## Installation
Ensure your package
To install Ascend.Queue from within Visual Studio, search for Ascend.Queue in the NuGet Package Manager UI, or run the following command in the Package Manager Console:

    Install-Package Ascend.Queue


## Requirements

## Subscriber
### Subscriber Factory
 - Use the SubscriberFactory in Ascend.Queue.Factories with the
   [Connection](#connection) object to get a Subscriber. 
 - Specify the type
   of key and value you are receiving with the Subscriberyou are
   requesting.  
 - Use the Null object from Ascend.Queue.Models if no key
   is given.

			    var connection = new Connection()
                {
                    ApplicationName = "DemoSubscriber",
                    QueueNames = new List<string> { "foo" },
                    Endpoint = "dev-kafkabroker.ascendlearning.com:9092"
                };
    
                var subscriber = SubscriberFactory.GetSubscriber<string, string>(connection);
### Subscriber Methods

 - **event EventHandler<Log> OnLog** 
	 - Register a method with OnLog to receive queue specific logs.  Depending on the queue, you can throttle the logs coming in using the QueueSettings object of the Connection object.  For Kafka, use [KafkaSettings](#kafkasettings) DebugLogging.
 - **event EventHandler<Message<TKey, TValue>> OnMessage**
	 - Register a method with OnMessage to receive the next message in the queue.  Depending on the queue, you can throttle the logs coming in using the QueueSettings object of the Connection object.  For Kafka, use [KafkaSettings](#kafkasettings) DebugLogging.
 -  **Task<string> CommitAsync()**
	 - Use this method if you are controlling your own commits.  By default, auto commiting is turned on.  To disable auto commit, please refer to [KafkaSettings](#kafkasettings) EnableAutoCommit.
  - **State State { get; }**
	  - Returns the current state of the Consumer.  
	  - Type of Ascend.Queue.Enumerations.State
		  - NotStarted
		  - Running
		  - Paused
  - **void SetConsumerId(string ConsumerId)**
	  - Sets an id to persist on your [Message](#message) object
  - **string GetConsumerId()** 
	  - Gets the id set to persist on your [Message](#message) object
  - **void Start()**
	  - Start the Subscriber.  Call after you have subscribed to OnMessage and OnLog.
  - **void Stop()**
	  - Stops the Subscriber.
  - **ISubscriber<TKey, TValue> Clone()**
	  - Get a new Subscriber of the same connection

### Basic Subscriber Example

	using System.Collections.Generic;
	using Ascend.Queue.Factories;
	using Ascend.Queue.Models;

	namespace SubscriberExample
	{
	    class Program
	    {
	        async static void Main(string[] args)
	        {
	            // Build your connection 
	            var connection = new Connection()
	            {
	                ApplicationName = "DemoSubscriber",
	                QueueNames = new List<string> { "foo" },
	                Endpoint = "dev-kafkabroker.ascendlearning.com:9092"
	            };

	            var subscriber = SubscriberFactory.GetSubscriber<string, string>(connection);

	            subscriber.OnLog += (e, log) =>
	            {
	                System.Console.WriteLine(log);
	            };

	            subscriber.OnMessage += (e, message) =>
	            {
	                System.Console.WriteLine(message.Key);
	                System.Console.WriteLine(message.Metadata);
	                System.Console.WriteLine(message.QueueName);
	                System.Console.WriteLine(message.Value);
	            };

	            subscriber.Start();

	            // Keep the thread running
	            while(true) { }
	        }
	    }
	}


## Publisher
### Publisher Factory

 - Use the PublisherFactory in Ascend.Queue.Factories with the
   [Connection](#connection) object to get a Publisher. 
  - Specify the type
   of key and value you are sending with the Publisher you are
   requesting.  
   - Use the Null object from Ascend.Queue.Models if no key
   is given.

		    var publisherConfiguration = new Connection()
            {
                ApplicationName = "DemoPublisher",
                QueueNames = new List<string> { "foo" },
                Endpoint = "dev-kafkabroker.ascendlearning.com:9092"
            };

            var publisher = PublisherFactory.GetPublisher<string, string>(publisherConfiguration);

### Publisher Methods
 - **Task<Message<Null, TValue>> SendMessageAsync(TValue message)** 
 - **Task<Message<TKey, TValue>> SendMessageAsync(TKey key, TValue message)** 
	 - Parameters
		 - **Key** - The type determined when requesting a publish from the factory
		 - **Value** - This is the message body you wish to send.  The type should be of the type determined when requesting a publisher from the factory
	 - Use this method to issue messages to the queue.
 - **event EventHandler<Log> OnLog**
	 - Register a method with OnLog to receive queue specific logs.  Depending on the queue, you can throttle the logs coming in using the QueueSettings object of the Connection object.  For Kafka, use [KafkaSettings](#kafkasettings) DebugLogging.
### Basic Publisher Example
Use the static PublisherFactory to get an instance of your publisher. 

Specify the key and value types you are planning to send.  If you are not planning on sending a key, use the Ascend.Queue.Models.Null object in its place.  Read more about keys [here](https://kafka.apache.org/documentation/#intro_consumers).

You should use the  `SendMessageAsync`  method if you would like to wait for the result of your produce requests before proceeding. You might typically want to do this in highly concurrent scenarios, for example in the context of handling web requests. Behind the scenes, the client will manage optimizing communication with the Queue for you, batching requests as appropriate.

    using System;
    using System.Collections.Generic;
    using Ascend.Queue.Factories;
    using Ascend.Queue.Models;
    
    namespace PublisherExample
    {
        class Program
        {
            async static void Main(string[] args)
            {
                // Build your connection 
                var publisherConfiguration = new Connection()
                {
                    ApplicationName = "DemoPublisher",
                    QueueNames = new List<string> { "foo" },
                    Endpoint = "dev-kafkabroker.ascendlearning.com:9092"
                };
    
                var publisher = PublisherFactory.GetPublisher<string, string>(publisherConfiguration);
    
			    publisher.OnLog += (e, log) =>
	            {
	                System.Console.WriteLine(log);
	            };
                for (var i = 0; i < 10; i++)
                {
                    var key = i;
                    var message = Guid.NewGuid().ToString();
                    var response = await publisher.SendMessageAsync(i, message);
                }
            }
        }
    }



## Connection
The Connection object is used to tell the Subscriber and the Publisher needed information about connecting to the Queue (currently Kafka).    

### Connection Members

 - **ApplicationName (required)** - This is important as it gives the subscriber a specific name to monitor.  When using Kafka, this will be considered the "group.id" Giving all instances of your subscriber the same group name will ensure that only one instance of the group will receive a unique message.
 - **QueueNames (required)** - A list of queues (topics in Kakfa).  
	 - A subscriber can subscribe to multiple queues. The ***Message*** object contains metadata that will differentiate between the topics.  ***There can only be one topic per Publisher.  If multiple are given, only the first topic in the list will be used.*** 
 - **Endpoint (required)** - The queue server your want to connect to
 - **Username** - Required if authentication is used on the queue
 - **Password** - Required if authentication is used on the queue
 - **Settings** - Queue technology specific settings.  For Kafka use the [KafkaSettings](#kafkasettings) object

	   var connection = new Connection()
	         {
	             ApplicationName = "TimsLocalMachine",
	             QueueNames = new List<string> { "foo" },
	             Endpoint = "dev-kafkabroker.ascendlearning.com:9092",
	             Username = "ati.stuqmon.stg-consumer",
	             Settings = new KafkaSettings()
	             {
	                 ConsumerPollIntervalMS = 5000,
	                 AutoCommitIntervalMS = 100,
	                 AutoOffsetReset = Ascend.Queue.Enumerations.OffsetResets.Earliest,
	                 EnableAutoCommit = false,
	                 DebugLogging = new List<Ascend.Queue.Enumerations.Debug>() { Ascend.Queue.Enumerations.Debug.consumer, Ascend.Queue.Enumerations.Debug.msg }
	             }
	         };
	         var subscriber = SubscriberFactory.GetSubscriber<Null, string>(connection);

### KafkaSettings
 - **NumberOfConsumerThreads** - The number of threads running on an application instance.  
	 - This is specific to ***Subscribers*** only.  
	 - Used to scale Kafka Consumers in a Consumer Group.  Read more about that [here](https://kafka.apache.org/documentation/#intro_consumers)
	 - *Default 1*
 - **PollTimeSpanMS** - The time a Consumer waits to timeout on a poll to the Broker. 
	 - *Default 100MS* 
 - **ConsumerPollIntervalMS** - The time between a Consumer poll attempt
	 - *Default 100MS*
 - **EnableAutoCommit** - 
	 - If true, commits will be handled by the framework based on a schedule. 
	 - If false, the Subscriber application will need to use the CommitAsync() method to commit it's own offsets
	 - *Default true*
 - **AutoCommitIntervalMS** - The time between committing a batch of offsets.  ***Only used if EnableAutoCommit is true***
	 - *Default 1000MS*
 - **AutoOffsetReset** - When a Consumer is starting for the first time, and has not committed an offset, it needs to determine whether it wants to read from the beginning of the Queue or start with the next incoming message.  
	 *Default OffsetResets.Earliest*
- **DebugLogging** - Takes a list of Ascend.Queue.Enumerations.Debug.  Read more about each of these debug settings [here](https://docs.confluent.io/4.1.2/clients/librdkafka/CONFIGURATION_8md.html).
 
## Message
### Message Object
- **ConsumerId** - If provide, this id will persist with each message sent with a specific consumer.  This gets set on the [Subscriber](#subscriber) object.
-  **QueueName** - The name of the queue the message is coming from
-  **Key** - The key that is given for the specific message.  If no key was given, this will be null.
- **Value** - The message body of the message in the queue.  This will be the value your application is most likely looking for. 
- **Metadata** - A Dictionary<string, string> of queue specific information.  For Kafka these will be "offset", "partition" and "topic"

## Log 
### Log Object
- **Level** - Type of AScend.Queue.Enumerations.LogLevel
	- Error
	- Warn
	- Info
	- Debug
- **Message** - A string containing the information of the log
