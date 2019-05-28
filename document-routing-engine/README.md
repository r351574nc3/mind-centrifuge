# Document Routing Engine


## Background

A document routing engine routes a collection of data (`Document`) by using transitioning states (like in a state machine). Using multiple routes and transitions, a `Document` can make its way from one user to another where that user take some action upon it. The user does not have to be a interactive interface. It could be non-interactive/autonomous.

## Goals

* **Domain Independent** system that can be functionaly scaled infinitely because it does not need to know about any specific domain. `Document` instances that it handles can be from any domain. 

* **Document-Driven** system where events, notifications, and process details are encapsulated in documents.

* **State Machine** where documents are processed to different states along
a route to a final state/destination.

* **Synchronous** routing where routes must continue in series

* **Asynchronous** routing where routes can be handled simultaneously

* **Conditional** routing where previous states determine progress through the route or functional dependencies must be met.
 
* **Interactive** isn't always automated. Can require human interactivity to ensure the transition from one state to the next.

* **Auth Integration** system that is tightly integrated with authorization because states are actually points where users (systems or people) that can take action. Actions that can be taken and users allowed access must be restricted. The only way to do this is tight integration with Authorization.

* **Reactive** uses events to determine state change.

## Assumptions

* Events are delivered through event stream implementation

* Authorization system provides functional extension point for integration

## Proposed Approach

![Routing Activity][diagram1]

### Overview

The basic idea is a system that consumes/produces events to/from external systems and itself. The system should 

1. Take some content `Document` (given via event from an external system)

1. Determine a route to use for it

1. Transition the `Document` from one state to the next by consuming events (either from itself or an external system)

1. Eventually, reach a final state (destination)

#### Use Cases

Cases for these are many. Most of which is when we want to conditionally manage some kind of flow.

**Digital Signoff** if approval needs to be granted on a process, multiple signatures/approvals can be given via a `Document`. For example, granting access to production systems. Signoff can require an entire group (Managers or product owners) to signoff, a single person in an entire group, or just one person in particular.

**Your eyes only** routing can be used to ensure that documents only get to where they are intended.

**Reviewal** design review, code review, marketing review, etc... different reviews have a process and flow where some approval/disapproval is reached. 

**Notifications** some kind of notification where the receiver must `Acknowledge` receipt. One example, ensuring that users finish their HIPAA certifications. 

**Auditing** Document routing offers an detailed set of events for understanding a process with high visibility.


#### Release to Production

This is an example route of a customer releasing software to production.

![Potential Document Routing Interaction][diagram2]

> **Interaction** diagram about component interactions across the **Document Routing Engine**

![Potential Document Routing Activity][diagram4]

> **Activity** diagram illustrating the state machine states and flows.

1. **Product Manager** initiates release by releasing changes through some 
change management tool or issue tracker like **Jira**

1. The change management system submits an event notifying release to interested parties. 

1. Simultaneously two things happen
    * The `Document Routing Engine` consumes and handles the event by starting the routing process. It then begins to wait the next transition to be ready.
    * **Continuous Integration** retrieves the event to start release and begins the release process by creating a release artifact.

1. Once the release artifact is created, the **CI** system submits and event that triggers a new transition.

1. The `document routing engine` consumes the event and 
 1. Creates a new transition for submitting a relase artifact
 1. Sends a notification/request for approval to the **release manager**

1. At this point, the **release manager** can:
 1. Approve the release which will (1. Set a new status on the transition. 2. Send an event for the CI system to push the release artifact. 3. Start a new transition.)

 1. Disapprove the release which will (1. shut down the whole release. 2. Send a event for the CI system to cleanup.)

1. **CI** System will consume the event and push the artifact to the artifact repository

1. **CI** System will send another event notifying the `Document Routing Engine` the completion of the action which will complete one of the conditions for the next transition.

1. The `Document Routing Engine` will send a notification to the **release manager**, **product manager**, and the **cluster manager** to confirm deployment of the newly released artifact to the production cluster.

#### Access to Production

In this scenario, a cluster administrator is unable to access a cluster because the cluster contains PII and requires those that have access to be FERPA certified. The administrator tries to access the cluster which starts a workflow guiding the administrator through an access request process. The process eventually leads to the administrator obtainin access to the cluster.

![Authorization Routing Interaction][diagram3]

> **Interaction** diagram about component interactions across the **Document Routing Engine**

![Authorization Routing Activity][diagram5]

> **Activity** diagram illustrating the state machine states and flows.

1. **Cluster Administrator** connects to cluster dashboard

1. **Authorization** verifies **Cluster Administrator** is missing **FERPA** certification.

1. **Authorization** produces event to start **FERPA** certification routing. 

1. **Document Routing Engine** sends notification to **Cluster Administrator**

1. **Cluster administrator** acknowledges notification

1. **Cluster administrator** takes **FERPA** exam and passes

1. **LMS** produces event for **FERPA** certification

1. **Document Routing Engine** and **Authorization** consume **FERPA** certification event

1. **Document Routing Engine** autoapproves transition based on **FERPA** certification condition.

1. **Cluster Administrator** can now access the cluster


#### Documents

Should have as little domain information as possible. These are JSON collections of data that are essential for the routing/transitioning from one state to the next. For example, if authorization routing, it probably has the following: 
```
    {
        role_urn: ''
        user_urn: ''
        route_urn: '', 
        operation: {}
    }
```

#### Routes

A route describes a set of transitions and states. The best way to illustrate or manipulate routes is via graph. A graph database will be used to compose your route.

#### Transitions

Transitions are basically the conditions that describe a `Document` changing states. For example, when an authorization `Document` engages the `Start` state, that is a startup transition. It then waits for manager's `Approval`. Upon `Disapprove`/`Approve`, the state will change and begin another transition. There are two conditions in this case. One condition is that the approver is a manager. The other condition is the `Approve` or `Disapprove` status of the previous transition. That is, if the startup transition was `Approved` and approver is a manager, then route to that approver. 

#### Actions

These are functions that can be applied to a `Document` at a specific transition. Below is a list of possible actions:

* **Approve** move the `Document` to the `Approved` state for a given transition.

* **Disapprove** move the `Document` to the `Disapproved` state for a given transition.

* **Acknowledge** to acknowledge you have received a notification. 

## System Technical Design

### Overview

This system will be composed of the following:

* **State Machine** to manage states and transitions of `Documents`

* **Document** datastore to manage data

* **Graph** datastore to describe routes, transitions, and states

* **Even Processor** for consuming events and communicating with the state machine on state changes.

## Technology Decisions

### Overview

TBD

Doc Version: `1.0.0`

[diagram1]: ./images/RoutingActivity.png
[diagram2]: ./images/PotentialDocumentRoutingInteraction.png
[diagram3]: ./images/AuthorizationRoutingInteraction.png
[diagram4]: ./images/PotentialDocumentRoutingActivity.png
[diagram5]: ./images/AuthorizationRoutingActivity.png