# Wortel

Wortel is a framework that takes client requests as input and synchronously publishes events to both Pulsar and MongoDB.  This allows developers to implement Event Sourcing architectures using these technologies without worrying about synchronisation problems between the event store and event bus. If Event Sourcing is new to you and you want to understand what it’s useful for, Derek Comartin provides a very good introduction with examples.


- [Enviroment](#enviroment)
- [Background](#background)
- [Trigger and Striker](#trigger-and-striker)
- [Validation and Error Handling](#validation-and-error-handling)
- [Getting Started](#getting-started)
- [Using Wortel in an Application](#using-wortel-in-an-application)
    * [Creating Aggregates](#creating-aggregates)

## Enviroment

Apache Pulsar 2.8 or above

Requires at least Java 17 and Spring Boot 2.5.6

Docker (for tests)



## Background

Wortel was born out of a problem we encountered when trying to implement Event Sourcing and CQRS in one of our applications.  Event Sourcing architecture has two main components; an event store, basically a log of all the events that happened in a database, and an event bus, which routes events to the consumers that need to know about them.

We’d identified Pulsar as the best platform for doing Event Sourcing. Pulsar is a message broker – its primary function is to shunt messages between microservices.  Its message brokering capacity made it the perfect event bus and it had infinite message retention, meaning it could function as an event store in principle.  We really liked the fact that Pulsar supported transactions. Often in enterprise applications, a database update involves a synchronised group of operations, all of which must happen in order for the update to go ahead.

For example, imagine you transfer $20 between two bank accounts.  For this to be a valid transfer, your account must be debited $20 and the recipient account must be credited by exactly the same amount.  The fact that Pulsar supported transactions made it ideal for our application.

There was just one problem. You could only query messages by time. This meant that Pulsar couldn’t meaningfully store Aggregates. Aggregate is Event Sourcing jargon for an entity in a database model, such as Pet or Customer.  Because database queries are always asking questions about relations between entities (all the Pets bought by a Customer), an Event Sourcing application needs to sort events by Aggregate, not just time.

We bolted on MongoDB to function as an Aggregate store.  This led to another problem. Mongo DB was effectively doing duty as our event store while Pulsar was the event bus. We needed a way to ensure that whenever one was updated, the other was too. Without this, there was the possibility that events would be added to the store, but not published to the bus.

Since the end users query databases that listen to the event bus for updates, their view of the data would become out of date.  Think of it like Thomas the Tank Engine pulling his coaches Annie and Clarabel. Without a coupling, Thomas will carry right on but all his passengers will be left behind! Wortel is the coupling.

## Trigger and Striker

Wortel consists of two frameworks, Trigger and Striker.  Trigger accepts http requests from a client and converts them to commands, which it publishes to Pulsar. Striker picks up Trigger’s commands from Pulsar and converts them to events. Striker publishes these to both Pulsar, where they are routed through the event bus, and MongoDB, where they are stored in Aggregate form.

In Event Sourcing, commands are like items on a to-do list. They’re typically named in the imperative, e.g a command called CreatePet tells Striker to create a Pet object. Conversely, events are like diary entries, providing a record of operations that have been performed on a database. Once the aforementioned Pet object has been created, there is a PetCreated event in the event store.

In computer terms (yes – we have to use them at some point) both commands and events consist of a CUD operation and an Aggregate.  Wortel represents the three CUD operations as Pulsar Topics.  Topics are categories, bins that commands and events of a particular type can be routed through.

Aggregates are represented as entities in MongoDB’s data model. In Wortel, all Aggregates contain three members:

- Aggregate id – every Aggregate has a unique id which Striker can use to create commands

- Aggregate version – this keeps track of the Aggregate ‘snapshots’

- Request id – this is the id of the latest request that was made to an Aggregate.  Striker uses it to identify the originating request of an event.

To make a command, Wortel takes the id of one of Pulsar’s Create Update or Delete Topics and pairs it with an id of a given Aggregate. Events are similar, but they contain additional information about the request that was fed into Trigger. This helps with validation and error handling.

Let’s look at how Wortel would create a Pet in a Pet Store application. First, Trigger receives a Post request to add a new Pet to the database.  Trigger reads this request and generates a command from the Pet Aggregate id and the id of Pulsar’s Create Topic.  To validate the command publication process there is a special Reply-to Topic. Trigger checks this to make sure the command has been published to Pulsar.  If it has, Trigger prints the message "command accepted {commandKeyNam}" to the log.

Pulsar views the command as a message. Its API has a Message Listener, which Striker uses to listen for the CreatePet command. After performing validation checks, it publishes a PetCreated event to both Pulsar, where it’s routed through the appropriate Topics and MongoDB, where it’s stored as a ‘snapshot’ of the Pet Aggregate.



## Validation and Error Handling



When Striker receives message, it performs a number of sanity checks. First Striker checks that CreatePet is a valid command.  If it’s not, Striker prints a message to the log listing the constraint violations.

If this test passes, Striker checks that the CreatePet hasn’t already been published as an event. If it has Striker returns an error response with the message validation.request.duplicated.

## Getting Started



Wortel can be installed through Maven. Paste the dependencies for Trigger and Striker into the dependencies section in your POM. The Maven dependency for Trigger is:
```xml

    <dependency>
        <groupId>com.transportexchangegroup</groupId>
        <artifactId>wortel-trigger</artifactId>
        <version>2.3</version>
    </dependency>
```

The Maven dependencies for Striker are:
```xml
    <dependency>
        <groupId>com.transportexchangegroup</groupId>
        <artifactId>wortel-striker</artifactId>
        <version>2.3</version>
    </dependency>
    <dependency>
        <groupId>com.transportexchangegroup</groupId>
        <artifactId>wortel-striker-storage-mongo</artifactId>
        <version>0.1</version>
    </dependency>
```



Make sure Pulsar is configured correctly in your concrete services as:

pulsar.service-url=pulsar://localhost:6650

To use the framework in your chosen IDE write the following import statement:

com.transportexchangegroup.wortel.*;



Now you’re ready to start coding!



## Using Wortel in an Application

Wortel is an extensible framework. It lets you use annotations to make your own Aggregates and commands.  For example, if you wanted to make a pet store app, Wortel gives you the flexibility to create a Pet Aggregate and custom commands to add and remove Pet objects.

### Creating Aggregates

Aggregates in Wortel need to extend the AbstractAggregate class and have an @Aggregate annotation to define their properties.  Here is the code for creating a Pet Aggregate:
```java
@Aggregate(name = "pet", eventClass = PetEvent.class)
public class PetAggregate extends AbstractAggregate {
}
```

The `@Aggregate` annotation is highly versatile and contains a number of different parameters.

`name` - aggregate name, used to provide sensible defaults to other parameters; consumer subscription name and event producer name are also derived from it;

`commandTopicName` - topic name for reading commands, persistent://public/default/{name}-commands by default;

`eventTopicName` - topic name for publishing events, persistent://public/default/{name}-events by default;

`eventClass` - ancestor class of all events the aggregate emits and consumes, Event by default;

`aggregateCollectionName` - name of the MongoDB collection the aggregate is saved to, aggregate class name by default;

`eventCollectionName` - name of the MongoDB collection the aggregate events are saved to, event class name by default.