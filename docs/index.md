# Introduction

This repository includes a set of documents for best practices around data replication between two Kafka clusters and data consistency.

## Replication with Mirror Maker 2.0

Mirror Maker 2.0 is the new replication feature of Kafka 2.4. It was defined as part of the Kafka Improvement Process - [KIP 382](https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0).

### General concepts

As [Mirror maker 2.0](https://strimzi.io/docs/master/#con-configuring-mirror-maker-deployment-configuration-kafka-mirror-maker) is using Kafka Connect framework, we recommend to review our summary of Kafka Connect [in this note](https://ibm-cloud-architecture.github.io/refarch-eda/kafka/connect/).

The figure below illustrates the MirrorMaker 2.0 internal components running within Kafka Connect.

![Kafka Connect](images/mm-k-connect.png)

In distributed mode, MirrorMaker 2.0 creates the following topics on the target cluster:

* mm2-configs.source.internal: This topic is used to store the connector and task configuration.
* mm2-offsets.source.internal: This topic is used to store offsets for Kafka Connect.
* mm2-status.source.internal: This topic is used to store status updates of connectors and tasks.
* source.heartbeats
* source.checkpoints.internal

A typical MirrorMaker 2.0 configuration is done via a property file and defines the replication source and target clusters with their connection properties and the replication flow definition. Here is a simple example for a local cluster replicating to a remote IBM Event Streams cluster using TLS v1.2 for connection encryption and SASL authentication protocol.  IBM Event Streams is a support, enterprise version of Apache Kafka by IBM.

```properties
clusters=source, target
source.bootstrap.servers=${KAFKA_SOURCE_BROKERS}
target.bootstrap.servers=${KAFKA_TARGET_BROKERS}
target.security.protocol=SASL_SSL
target.ssl.protocol=TLSv1.2
target.ssl.endpoint.identification.algorithm=https
target.sasl.mechanism=PLAIN
target.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="token" password=${KAFKA_TARGET_APIKEY};
# enable and configure individual replication flows
source->target.enabled=true
source->target.topics=products
tasks.max=10
```

* Topics are configured to be replicated or not using a _whitelist_ and _blacklist_ concept
* White listed topics are set with the `source->target.topics` attribute of the replication flow and uses [Java regular expression](https://www.vogella.com/tutorials/JavaRegularExpressions/article.html) syntax.
* Blacklisted topics: by default the following pattern is applied:

```properties
blacklist = [follower\.replication\.throttled\.replicas, leader\.replication\.throttled\.replicas, message\.timestamp\.difference\.max\.ms, message\.timestamp\.type, unclean\.leader\.election\.enable, min\.insync\.replicas]
```

We can also define the _blacklist_ with the properties: `topics.blacklist`. Comma-separated lists and Java Regular Expressions are supported.

Internally, `MirrorSourceConnector` and `MirrorCheckpointConnector` will create multiple Kafka tasks (up to the value of `tasks.max` property), and `MirrorHeartbeatConnector` creates an additional task. `MirrorSourceConnector` will have one task per topic-partition combination to replicate, while `MirrorCheckpointConnector` will have one task per consumer group. The Kafka Connect framework uses the coordinator API, with the `assign()` API, so there is no consumer group used while fetching data from source topics. There is no call to `commit()` either; rebalancing occurs only when there is a new topic created that matches the _whitelist_ pattern.

## Requirements to address

### Environments

We propose two approaches to run the _on-premise_ Kafka cluster:
 
* [Kafka 2.4 cluster using the Strimzi Operator deployed on Openshift](#scenario-2-run-mirror-maker-2-cluster-close-to-target-cluster) - This approach to runnig the local cluster leverages the Strimzi Kubernetes Operator running on the OpenShift Container Platform.
* Event Streams on Cloud cluster, we recommend to create your own using an IBM Cloud account. The product [documentation is here](https://cloud.ibm.com/registration?target=catalog/services/event-streams).

The enviroments are summarized in the table below:

| Environment | Source                 | Target                 | Connect |
|-------------|------------------------|------------------------|:-------:|
| 1           | Strimzi on OCP         | Event Streams on Cloud | OCP / Roks |
| 2           | Event Streams on Cloud | Strimzi on OCP         | OCP/ Roks |
| 3           | Event Streams on OCP   | Event Streams on Cloud | OCP / Roks |

The connect column defines where the MirrorMaker 2 connect to.


### Provisioning Connectors (MirrorMaker 2)

Thinking of our goals as Agile user stories, we list our stories and some notes and requirements below.

1. As an SRE, I want to provision and deploy MirrorMaker 2 connector to existing Openshift cluster without exposing passwords and keys so replication can start working. This will use Kubernetes secrets for configuration parameters.

    * We describe the MM2 deployment with secrets in [this section](mm2-provisioning/#deploying-using-strimzi-mirror-maker-operator).

1. As an SRE I want to understand the CLI commands used to assess how the provisioning process can be automated.

    * We did not show how to automate the deployment, but as all deployments are done with CLI and configuration files given, we could [consider using Ansible](mm2-provisioning#provisioning-automation) for automation. 

1. As an SRE, I want to understand the server sizing for the Mirror Maker environment so that I can understand the leanest resources for minimal needs.

    * We talk about capacity planning in [this section](mm2-provisioning#capacity-planning) and performance tests [in a separate note](perf-tests).

1. As an SRE I want to understand if Mirroring can run from older Kafka cluster version (1.1 +) to newer kafka cluster version.

    * We do not have an old Kafka 1.1 cluster available to us for testing, but after discussion with development and doing deeper analysis, the Kafka Connect framework was already working with Kafka 1.1 and use the bootstrap server to connect to the cluster. So using an existing topic defined in the Kafka 1.1 cluster as source for mirroring will work.
    
*Note that, there is no specific user interface for MirrorMaker 2.*

### Version-to-Version Migration


1. As a SRE, I want to understand how to perform a version-to-version migration for the MirrorMaker 2 product so that existing streaming replication is not impacted by the upgrade.
    
    * See some rolling upgrade recommendations in [this section](mm2-provisioning#version-migration).

1. As a developer I want to deploy configuration updates to modify the topic replication white or black lists so that newly added topics are replicated.

    * Mirror maker 2 configuration is done with yaml files and applied using `kubectl or oc` CLI. The Strimzi cluster operator is managing the redeployment of the mirror maker deployment.


### Security

1. As an SRE, I want to understand how client applications authenticate to source and target Kafka clusters.

1. As a developer, I want to design MirrorMaker 2 based replication solution to support different lines of businesses who should not connect to topics and data not related to their business and security scope.

Those subjects are addressed in [the security note](security.md).

### Monitoring

1. As an SRE, I want to get MirrorMaker 2 metrics for Prometheus so that it fits in my current metrics processing practices.

    * The explanation of how to set up Prometheus metrics for MirrorMaker 2.0 is documented [in the monitoring note](monitoring.md).

1. As an SRE, I want to be able to add new dashboards into Grafana to visualize the MirrorMaker 2 metrics.

1. As an SRE, I want to define rules for alert reporting and configure a Slack channel for alerting.

1. [Removed] As an SRE, I want to get the MirrorMaker 2 logs into our Splunk logging platform.

### Best Practices

1. As a developer I want to understand how MirrorMaker 2 based replication addresses the record duplication.

    * Here is [a note on records duplication](consideration#record-duplication).

1. As a developer I want to design the MirrorMaker 2 Kafka topic replication solution to use minimal resources but also be able to scale-up if I observe data replication lag.

    * Some lag will always be present due to the the fact that MirrorMaker 2 does asynchronous replication, but it is possible to scale MirrorMaker 2 vertically and horizontally to minimize the lag

1. As a developer I want to understand what are the conditions under which messages may be lost. We are detailing the exactly once and ensuring delivery in [those notes: producer](https://ibm-cloud-architecture.github.io/refarch-eda/kafka/producers#how-to-support-exactly-once-delivery) and [consumer](https://ibm-cloud-architecture.github.io/refarch-eda/kafka/consumers).


### Performance  tests

1. As a developer, I want to understand how to measure latency / lag in data replication.

    * IBM Event streams offers an event producers to do stress testing. The performance tests are documented [in this chapter](perf-tests.md).

1. As a SRE I want to understand current throughput for the replication solution.

    * Kafka brokers are reporting the number of request per second, and the byte in per sec as part of the metrics visible in Prometheus. Those metrics can be at the producer Fetch consumer or fetch follower level.

