# Eventing-Kafka Distributed Channel

This is the Kafka Channel implementation, originally contributed by
[SAP's Kyma project](https://github.com/kyma-project).

> See <https://github.com/knative/eventing-contrib/issues/1070> for discussion
> of the donation process.

This repo falls under the
[Knative Code of Conduct](https://github.com/knative/community/blob/master/CODE-OF-CONDUCT.md)

This project is a Knative Eventing implementation of a Kafka backed channel
which provides a more granular architecture as an alternative to what the
original "[consolidated](../consolidated)" implementation offers. Specifically
it deploys a single/separate Receiver, and one Dispatcher per KafkaChannel.

## Deployment

1. Setup
   [Knative Eventing](https://knative.dev/docs/admin/install/eventing/install-eventing-with-yaml/)


2. Install an Apache Kafka cluster:

   A simple in-cluster Kafka installation can be setup using the
   [Strimzi Kafka Operator](http://strimzi.io). Its installation
   [guides](http://strimzi.io/quickstarts/) provide content for Kubernetes and
   Openshift. The `KafkaChannel` is not limited to Apache Kafka installations on
   Kubernetes, and it is also possible to use an off-cluster Apache Kafka
   installation as long as the Kafka broker networking is in place.


3. Point the KafkaChannel at your Kafka cluster (brokers):

   Now that Apache Kafka is installed, you need to configure the
   `brokers` value in the `config-kafka` ConfigMap, located inside the
   [300-eventing-kafka-config.yaml](../../../config/channel/distributed/300-eventing-kafka-configmap.yaml)
   file.

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: config-kafka
     namespace: knative-eventing
   data:
     eventing-kafka: |
      kafka:
        brokers: <kafka-brokers-urls-csv>
   ```

   **Note:** Additional Kafka client configuration, such as TLS, SASL and other
   Kafka behaviors, is possible via the `sarama` section of
   the `config-kafka` [ConfigMap](../../../config/channel/distributed/300-eventing-kafka-configmap.yaml)
   which is detailed in the
   configuration [README](../../../config/channel/distributed/README.md).


4. Apply the KafkaChannel configuration:

   ```sh
   ko apply -f config/channel/distributed
   ```

5. Create a `KafkaChannel` custom object:

   ```yaml
   apiVersion: messaging.knative.dev/v1beta1
   kind: KafkaChannel
   metadata:
     name: my-kafka-channel
   spec:
     numPartitions: 1
     replicationFactor: 1
     retentionDuration: PT168H
   ```

   You can configure the number of partitions with `numPartitions`, as well as
   the replication factor with `replicationFactor`, and the Kafka message
   retention with `retentionDuration`. If not set, these will be defaulted by
   the WebHook to `1`, `1`, and `PT168H` respectively.


6. Create a `Subscription` to the `KafkaChannel`:

   ```yaml
   apiVersion: messaging.knative.dev/v1
   kind: Subscription
   spec:
     channel:
       apiVersion: messaging.knative.dev/v1beta1
       kind: KafkaChannel
       name: my-kafka-channel
   delivery:
     backoffDelay: PT0.5S
     backoffPolicy: exponential
     retry: 5
   subscriber:
     uri: <subscriber-uri>
   ```

## Rationale

The Knative "consolidated" KafkaChannel already provides a Kafka backed Channel
implementation, so why invest the time in building another one? At the time this
project was begun, and still today, the reference Kafka implementation does not
provide the scaling characteristics required by a large and varied use case with
many Topics and Consumers. That implementation is based on a single choke point
that could allow one Topic's traffic to impact the throughput of another Topic.

We also had the need to support a variety of Kafka providers, including Azure
EventHubs in Kafka compatibility mode as well as exposing the ability to
customize the Kafka Topic management. Finally, the ability to expose Kafka
configuration was very limited, and we needed the ability to customize certain
aspects of the Kafka Topics / Producers / Consumers.

## Status

Significant work has recently gone into aligning the two implementations from a
CRD, configuration, authorization, and code-sharing perspective, in order to
standardize the user experience as well as maximize code reuse. While the
runtime architectures will always be different
(the "raison d'etre" for having multiple implementations), the goal is to
continue this sharing. The eventual goal might be to have a single KafkaChannel
implementation that can deploy either runtime architecture as desired.

## Architecture

As mentioned in the "Rationale" section above, the desire was to implement
different levels of granularity to achieve improved segregation and scaling
characteristics. Our original implementation was extremely granular in that
there was a separate Receiver/Producer Deployment for every `KafkaChannel`
(Kafka Topic), and a separate Dispatcher/Consumer Deployment for every Knative
Subscription. This allowed the highest level of segregation and the ability to
tweak K8S resources at the finest level.

The downside of this approach, however, is the large resource consumption
related to the sheer number of Deployments in the K8S cluster, as well as the
inherent inefficiencies of low traffic rate Channels / Subscriptions being
underutilized. Adding in a service-mesh (such as Istio) further exacerbates the
problem by adding side-cars to every Deployment. Therefore, we've taken a step
back and aggregated the Receivers/Producers together into a single Deployment
per Kafka authorization, and the Dispatchers/Consumers into a single Deployment
per
`KafkaChannel` (Topic). The implementations of each are horizontally scalable
which provides a reasonable compromise between resource consumption and
segregation / scaling.

### Project Structure

The "distributed" KafkaChannel consists of three distinct runtime K8S
deployments as follows...

- [controller](controller/README.md) - This component implements the
  `KafkaChannel` Controller, and is using the current knative-eventing "Shared
  Main" approach based directly on K8S informers / listers. The controller
  utilizes the shared `KafkaChannel` CRD, [apis/](../../../pkg/apis), and
  [client](../../../pkg/client) implementations in this repository. This
  component also implements the `ResetOffset` Controller in order to support the
  ability to reposition the ConsumerGroup Offsets of a particular Subscription
  to a specific timestamp.

- [dispatcher](dispatcher/README.md) - This component runs the Kafka
  ConsumerGroups responsible for processing messages from the corresponding
  Kafka Topic. This is the "Consumer" from the Kafka perspective. A separate
  dispatcher Deployment will be created for each unique `KafkaChannel` (Kafka
  Topic), and will contain a distinct Kafka Consumer Group for each Subscription
  to the `KafkaChannel`.

- [receiver](receiver/README.md) - The event receiver to which all inbound
  messages are sent. An HTTP server which accepts messages that conform to the
  CloudEvent specification, and then writes those messages to the corresponding
  Kafka Topic. This is the "Producer" from the Kafka perspective. A single
  receiver Deployment is created to service all KafkaChannels in the cluster.

- [config](../../../config/channel/distributed/README.md) - Eventing-kafka
  **ko** installable YAML files for installation.

- [webhook](../../../cmd/webhook) - Eventing-Kafka Webhook will set defaults and
  perform validation of KafkaChannels.

### Control Plane

The control plane for the Kafka Channels is managed by the
[eventing-kafka-controller](controller/README.md) which is installed in the
knative-eventing namespace. `KafkaChannel` Custom Resource instances can be
created in any user namespace. The eventing-kafka-controller will guarantee that
the Data Plane is configured to support the flow of events as defined by
[Subscriptions](https://knative.dev/docs/reference/eventing/#messaging.knative.dev/v1alpha1.Subscription)
to a KafkaChannel. The underlying Kafka infrastructure to be used is defined in
a specially labeled
[K8S Secret](../../../config/channel/distributed/README.md#Credentials) in the
knative-eventing namespace. Eventing-kafka supports several Kafka (and
Kafka-like)
[infrastructures](../../../config/channel/distributed/README.md#Kafka%20Providers)
.

### Data Plane

The data plane for all `KafkaChannels` runs in the knative-eventing namespace.
There is a single Deployment for the receiver side of all channels which accepts
CloudEvents and writes them to Kafka Topics. Each `KafkaChannel` is backed by a
separate Kafka Topic. This Deployment supports horizontal scaling with linearly
increasing performance characteristics.

Each `KafkaChannel` has a separate Deployment for the dispatcher side which
reads from the Kafka Topic and sends to subscribers. Each subscriber has its own
Kafka consumer group. This Deployment can be scaled up to a replica count
equalling the number of partitions in the Kafka Topic.

### Messaging Guarantees

An event sent to a `KafkaChannel` is guaranteed to be persisted and processed if
a 202 response is received by the sender.

The CloudEvent is partitioned based on the
[CloudEvent partitioning extension](https://github.com/cloudevents/spec/blob/master/extensions/partitioning.md)
field called `partitionkey`. If the `partitionkey` is not present, then the
`subject` field will be used. Finally, if neither is available, it will
fall-back to random partitioning.

Events in each partition are processed in order, with an **at-least-once**
guarantee. If a full cycle of retries for a given subscription fails, the event
is ignored, or sent to the _Dead-Letter-Sink_ according to the Subscription's `DeliverySpec`
and processing continues with the next event.

## Offset Repositioning

The ConsumerGroup Offsets of a specific Knative Subscription can be
repositioned (backwards or forwards within the Topic's retention window) via the
[ResetOffset](../../../config/command/resetoffset/README.md) Custom Resource, to
allow events to be "replayed" in failure recovery scenarios.

## Installation

For installation and configuration instructions please see the config files
[README](../../../config/channel/distributed/README.md).
