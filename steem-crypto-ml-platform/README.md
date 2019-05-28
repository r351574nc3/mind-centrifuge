# Steem Cryptocurrency Event Stream

## Motivation

Steem proof-of-stake cryptocurrency platform uses something called a *witness server* to process events. The *witness server* also provides a microservices backend for querying and streaming events. It is possible to aggregate data from the microservices backend to another store for deep learning analysis.

## Goals

* Develop methodology for predicting and recognizing bot behavior on the platform.
* Use deep learning to predict what content will earn the most on the platform.

## Assumptions


## Proposed Approach


### Overview



#### Use Cases

## System Technical Design

### Overview

Stream events from *witness server*. Each event is packaged for the platform and sent through a producer. 

### Producers

### Consumers

1. Consumer for consuming events and sending to elasticsearch document store
1. 


## Technology Decisions

* Weka/Moa

### Overview

TBD

Doc Version: `1.0.0`

[diagram1]: ./images/RoutingActivity.png
[diagram2]: ./images/PotentialDocumentRoutingInteraction.png
[diagram3]: ./images/AuthorizationRoutingInteraction.png
[diagram4]: ./images/PotentialDocumentRoutingActivity.png
[diagram5]: ./images/AuthorizationRoutingActivity.png