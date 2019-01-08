# Starlark Social Builds

## Motivation

Social builds is a new concept where builds can communicate with each other across domain and applications. Most builds are based on a concept of pipelines where a linear workflow is followed until completion. This is not always possible though. Further, it leads to a pattern where artifacts are affectively thrown over a wall and discarded.

Pipelines follow a paradigm of IFTTT (If this, then that) where conditions govern the path of the workflow through its linear course. Consider instead of conditions we have events and events represent the alteration or transition of one state to another. We can then surmise that workflow can be expressed in terms of existing state, current state, and desired state. Transitions between states are non-linear. Now we can observe builds as finite state machines instead of pipelines.

With builds as finite state machines, builds can now communicate socially via shared events. Builds could then share and understand each other's states. For example, if one build fails another will become aware of it. Another example, if one build is changed, it will force a dependent build (previously referred to as downstream build) to initiate.


## Goals

* **Build discovery** builds will discover each other through build descriptors (one build declares itself dependent upon another build or an application describes that it is dependent on another)
* **Queriable dependency graph** high-level and low-level views of build process can be observed.
* **Organic notifications/events** builds are tied to identities that can be notified or even required to take action.
* **Self-defining workflow** discovery and other features allow workflow to define itself based on dependencies and requisite event structure.

## Assumptions

* **Identity Integration** consumers and producers will have access to an identity management system that exposes details about access controls and secrets.
* **Graph Structured Data**
* **Common Build Tools**
* **Continuous Integration**

## Proposed Approach

### Producers

1. Producer notifies on build creation.
1. Producer notifies on build change.
    1. Pricipals (un)assigned to a build
    1. Dependencies added/removed to a build
1. Producer notifies on build removal.
1. Producer notifies on build state
    1. Build state change
        1. Start
        1. Warning
        1. Failure
        1. Transition
        1. Acknowledgement
        1. Approval
        1. Disapproval
        1. FYI

### Consumers

1. Build event consumer
    * There is one event consumer that updates the dependency graph based on build events.
1. Build state consumer(s)
    * Each build has a state consumer that watches and reacts to the state of other builds.
1. Vault consumer
    * Vault has a consumer that captures new build events to promote access control registration.
1. Graph consumer
    * Neptune graph database consumes events from builds to update the build graph.

### Messaging Model

Messaging model is described through json-schema.

Limit so that messages are only decrypted if the message is specifically meant for the consumer. That is the consumer first finds messages its interested in, then determines whether it can read them instead of reading all messages. How does a consumer know what messages it needs?

* Type 
    * Verify that this object is one that the consumer can consume.
* Schema
    * Verify that this object follows a schema showing that the consumer can consume it.
* Key id
    * Verify that this object has rights to the message content. If the consumer does not have the correct decryption key, it will not be able to read the message. Decryption keys are obtained through Vault identity framework. Each producer uniquely encrypts messages. It is possible that a producer uses multiple keys based on ACL. Keys are a way to verify authorization without having to continuously ping identity services.

```javascript
{
    "schema": "http://r351574nc3.github.io/mind-centrifuge/starlark-social-builds/event.schema.json",
    "message_type": "Build Event",
    "message_id": "",
    "Key": "metaknight-12345",
    "message_body": "ewogICAgIiRzY2hlbWEiOiAiaHR0cDovL2pzb24tc2NoZW1hLm9yZy9kcmFmdC0wNy9zY2hlbWEjIiwKICAgICIkaWQiOiAiaHR0cDovL3IzNTE1NzRuYzMuZ2l0aHViLmlvL21pbmQtY2VudHJpZnVnZS9za3lsYXJrLXNvY2lhbC1idWlsZHMvZXZlbnQuc2NoZW1hLmpzb24iLAogICAgInRpdGxlIjogIkJ1aWxkIiwKICAgICJkZXNjcmlwdGlvbiI6ICJCdWlsZCBFdmVudCIsCiAgICAidHlwZSI6ICJvYmplY3QiLAogICAgInByb3BlcnRpZXMiOiB7CiAgICAgICJwcm9kdWN0SWQiOiB7CiAgICAgICAgImRlc2NyaXB0aW9uIjogIlRoZSB1bmlxdWUgaWRlbnRpZmllciBmb3IgYSBwcm9kdWN0IiwKICAgICAgICAidHlwZSI6ICJpbnRlZ2VyIgogICAgICB9CiAgICB9LAogICAgInJlcXVpcmVkIjogWyAicHJvZHVjdElkIiBdCiAgfQ=="

}
```
> **Above** Sample encrypted message

```
{
    "event_type", "Build Created",
    "object": {  // Build object
        "id": 1, // Build id
        "project": 0,
        "tenant": 0,
        "imports": [
            "github.com/r351574nc3/mind-centrifuge.git", // Source repository id
        ], // projects depended on
    }
}
```
> **Above** Sample actual message body (unencrypted)

```
{
    "event_type", "Build Failed",
    "object": {  // Build object
        "id": 1, // Build id
        "project": 0,
        "tenant": 0,
        "imports": [
            "github.com/r351574nc3/mind-centrifuge.git", // Source repository id
        ], // projects depended on
    }
}
```
> **Above** Sample actual message body (unencrypted)

### Multitenancy

Tenant information is encoded into the id (hashid) of the message. The `message_id` is composed using `hashids` of different unique identifiers:
* `event_id`
    * Unique identifier for the event or message
* `tenant_id`
    * Unique identifier for multitenancy because each tenant will have its own set of messages that can be retrieved.
* `producer_id`
    * Unique identifier for the producer that published the message. This id is created by the producer at initialization or registration with the platform.
* `epoch timestamp`
    * Integer that represents the exact time the message was sent. This is used for managing state.

The unique identifiers are composited together using a salt. The resulting composite is a hash that is the `message_id`. 
> The `message_id` result is an alphanumeric string that resembles something like `3kTMd`.


### Security

Any consumer can subscribe to messages. To prevent consumers from accessing information that is not intended for them, there are two layers of security added.

#### hashids

> **Isla de Muertes**

`event_id`, `tenant_id`, `producer_id`, and `timestamp` are secured through hashing. A salt is required to decode a composite id (`3kTMd`) into its decomposed parts. This prevents information like the origin of the message, what it contains, and who it is intended for are kept secret except for by those that expect it.

#### Public/Private Key Encryption

Message bodies are encrypted via private key. Private keys are maintained by the message producers, but public keys are maintained/stored in a secret store. Consumers that wish to gain access to message contents can gain access to the public key from the secret store.

These are considered layered security measures because a public key is coupled to a producer. That is it cannot be reused across producers. It is unknown which public key will decrypt the message without knowing the producer that created it. The producer identity is composited in the hashid; therefore, in order to decrypt the message contents, the producer must be determined from the hashid by decrypting it first.

#### Performance

Messages do not require being encrypted. Encryption is only required if there is a possibility that sensitive information may be passed in the message contents. To give an example, public builds will probably not be encrypted while private builds will.

Normally, large amounts of decryption can lead to performance declines; however, the following must be considered:
1. Not all messages are encrypted
1. Only messages that a consumer can understand will be decrypted.

The consumer will first validate the message schema to show this is a message the consumer can consume. Next, the consumer must use the salt (supplied through secret store) to decode the hashid. If the hashid can decode correctly and the schema validation passes, we can safely assume the message is intended for this consumer.

Based on the above, decryption should only happen on messages that are encrypted and messages that are intended for the consumer.


### Processes

Steps and flows for what happens

#### New Project/Build (Build Discovery)

![New Build Activity][diagram2]

1. User creates new project
1. Producer is instantiated notify new project creation.
1. Build generator (Consumer/Producer) receives notification.
    1. New build is created
        1. New `id` is created
        1. Build file is scanned for dependency projects and builds.
        1. Build producer created
            * Each build has its own producer to send events
        1. Build consumer created
            * Each build has its own consumer to handle events.
    1. Notification of new build is created and sent including reults of build file scan and producer details
    1. Vault consumer gets event for new build and submits necessary access requests for a human being to grant permission to necessary build keys

#### New Dependencies

1. Build consumer gets event for new build with dependency information.
1. Build consumer requests necessary keys from Vault
1. Build instantiated
    1. Build started notification is sent
    1. Build ended notification is sent

#### Build Failure

1. Build failed
1. Build failure event sent from producer
1. Build consumers get failure event
1. Build mark current build as expected to fail



## System Technical Design

### Overview

## Technology Decisions

* Kafka
* Bazel build tool
* Starlark python dialect in Bazel for business rules
* Vault for secrets management and identities
* Hashids used for ids to embed meta information within an id that also makes it unique
* Apollo Server GraphQL integration
* Kubernetes for turnkey container cluster management and deployments.

### Infrastructure

![Infrastructure Overview][diagram1]

AWS EC2 Nodes are the foundation for a K8S cluster where the KaaS will reside. A large Kubernetes cluster exists with pods that deployments and statefulsets can be made to.

#### Apollo Server

Apollo Server is installed within the kubernetes cluster and has 1 replica.

#### AWS Neptune 

AWS Neptune is managed through AWS and exists outside the kubernetes cluster.

#### Vault

Vault will be managed externally to the cluster.

#### Kafka and Zookeeper

Kafka and Zookeeper is installed on `StatefulSets` with 3 replicas each.

#### CI/CD 

Jenkins is used for CI/CD, but it could be Gitlab or any other system for instantiating builds. The CI/CD system is deployed to a pod with no replicas.


Doc Version: `1.0.0`

[schema1]: ./schema/build.schema.json
[schema2]: ./schema/event.schema.json
[diagram1]: ./images/KaaSInfrastructureOverview.png
[diagram2]: ./images/NewBuild_Activity.png
[diagram3]: ./images/AuthorizationRoutingInteraction.png
[diagram4]: ./images/PotentialDocumentRoutingActivity.png
[diagram5]: ./images/AuthorizationRoutingActivity.png