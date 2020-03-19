# CCFD demo

![diagram](docs/diagram.png)

# Contents

- [CCFD demo](#ccfd-demo)
- [Contents](#contents)
  - [Setup](#setup)
    - [Running on OpenShift](#running-on-openshift)
      - [OpenDataHub](#opendatahub)
      - [Kafka](#kafka)
      - [Seldon](#seldon)
      - [Kie server](#kie-server)
        - [Execution server](#execution-server)
      - [Notification service](#notification-service)
      - [Camel router](#camel-router)
      - [Kafka producer](#kafka-producer)
  - [Description](#description)
    - [Business processes](#business-processes)
  - [Footnotes](#footnotes)

## Setup

### Running on OpenShift

To deploy all the components in OpenShift, the simplest way is to login using `oc`, e.g.:

```shell
$ oc login -u <USER>
```

Next you can create a project for this demo, such as

```shell
$ oc new-project ccfd
```

#### OpenDataHub

We start by installing OpenDataHub via its operator. Start by cloning the operator:

```shell
$ git clone https://gitlab.com/opendatahub/opendatahub-operator
$ cd opendatahub-operator
```

Next, we deploy ODH and Seldon's CRD. [^0]

```shell
$ oc create -f deploy/crds/opendatahub_v1alpha1_opendatahub_crd.yaml
$ oc create -f deploy/crds/seldon-deployment-crd.yaml
```

Next, create the services and RBAC policy for the service account the operator will run as. This step at a minimum requires namespace admin rights.

```shell
$ oc create -f deploy/service_account.yaml
$ oc create -f deploy/role.yaml
$ oc create -f deploy/role_binding.yaml
$ oc adm policy add-role-to-user admin -z opendatahub-operator
```

Now we can deploy the operator with:

```shell
$ oc create -f deploy/operator.yaml
```

And wait before the pods are ready before continuing. You can verify using:

```shell
$ oc get pods
```

#### Kafka

[Strimzi](https://strimzi.io/) is used to provide Apache Kafka on OpenShift.
Start by making a copy of `deploy/crds/opendatahub_v1alpha1_opendatahub_cr.yaml`, *e.g.*

```shell
$ cp deploy/crds/opendatahub_v1alpha1_opendatahub_cr.yaml frauddetection_cr.yaml
```

and edit the following values:

```yaml
# Seldon Deployment
seldon:
odh_deploy: true

kafka:
odh_deploy: true
kafka_cluster_name: odh-message-bus
kafka_broker_replicas: 3
kafka_zookeeper_replicas: 3
```
Kafka installation requires special setup, the following steps are to configure Kafka. Add your username to the `kafka_admins` list,
by editing `deploy/kafka/vars/vars.yaml`:

```yaml
kafka_admins:
- admin
- system:serviceaccount:{{ NAMESPACE }}:opendatahub-operator
- <INSERT USERNAME>
```

You can now deploy Kafka using:

```shell
$ cd deploy/kafka/
$ pipenv install
$ pipenv run ansible-playbook deploy_kafka_operator.yaml -e kubeconfig=$HOME/.kube/config -e NAMESPACE=<namespace>
```

Deploy the ODH custom resource based on the sample template

```shell
$ oc create -f frauddetection_cr.yaml
```

#### Seldon

To deploy the test Seldon model server, deploy the already built Docker image with

```shell
$ oc new-app ruivieira/ccfd-seldon-model
$ oc expose svc/ccfd-seldon-model
```

#### Kie server

##### Execution server

To deploy the KIE server, the container image `ruivieira/ccd-service` can be used (located [here](https://hub.docker.com/repository/docker/ruivieira/ccd-service)),  deploying it with:

```shell
$ oc new-app ruivieira/ccd-service:1.0-SNAPSHOT \
    -e SELDON_URL=ccfd-seldon-model:5000 \
    -e NEXUS_URL=http://nexus:8081 \
    -e CUSTOMER_NOTIFICATION_TOPIC=ccd-customer-outgoing \
    -e BROKER_URL=odh-message-bus-kafka-brokers:9092
```

If the Seldon server requires an authentication token, this can be passed to the KIE server by adding the following environment variable:

```shell
-e SELDON_TOKEN=<SELDON_TOKEN>
```

If you want to interact with the KIE server's REST interface from outside OpenShift, you can expose its service with

```shell
$ oc expose svc/ccd-service
```

#### Notification service

The notification service is an event-driven micro-service responsible for relaying notifications to the customer and customer responses. 

If a message is sent to a "customer outgoing" Kafka topic, a notification is sent to the customer asking whether the transaction was legitimate or not. For this demo, the micro-service simulates customer interaction, but different communication methods can be built on top of it (email, SMS, *etc*).

If the customer replies (in both scenarios: they either made the transaction or not), a message is written to a "customer response" topic. The router (described below) subscribes to messages in this topic, and signals the business process with the customer response.

To deploy the notification service, we use the image `ruivieira/ccfd-notification-service` (available [here](https://hub.docker.com/repository/docker/ruivieira/ccfd-notification-service)), by running:

```shell
$ oc new-app ruivieira/ccfd-notification-service:1.0-SNAPSHOT \
    -e BROKER_URL=odh-message-bus-kafka-brokers:9092
```

#### Camel router

The Camel router is responsible consume messages arriving in specific topics, requesting a prediction to the Seldon model, and then triggering different REST endpoints according to that prediction.
The route is selected depending on whether a transaction is predicted as fraudulent or not. Depending on the model's prediction a specific business process will be triggered on the KIE server.
To deploy a router with listens to the topic `KAFKA_TOPIC` from Kafka's broker `BROKER_URL` and starts a process instance on the KIE server at `KIE_SERVER_URL`, we can use the built image `ruimvieira/ccd-fuse` (available [here](https://hub.docker.com/repository/docker/ruivieira/ccd-fuse)):

```shell
$ oc new-app ruivieira/ccd-fuse:1.0-SNAPSHOT \
    -e BROKER_URL=odh-message-bus-kafka-brokers:9092 \
    -e KAFKA_TOPIC=ccd \
    -e KIE_SERVER_URL=http://ccd-service:8090 \
    -e SELDON_URL=ccfd-http://seldon-model:5000 \
    -e CUSTOMER_NOTIFICATION_TOPIC=ccd-customer-outgoing \
    -e CUSTOMER_RESPONSE_TOPIC=ccd-customer-response
```

Also optionally, a Seldon token can be provided:

```shell
-e SELDON_TOKEN=<SELDON_TOKEN>
```

#### Kafka producer

To start the Kafka producer (which simulates the transaction events) run:

```shell
$ oc new-app ruivieira/ccfd-kafka-producer \
    -e BROKER_URL=odh-message-bus-kafka-brokers:9092 \
    -e KAFKA_TOPIC=ccd
```


## Description

### Business processes

![](docs/process-fraud.png)

The Business Process (BP) corresponding to a potential fraudulent transaction consists of the following flow:

* The process is instantiated with the transaction's data
* The `CustomerNotification` node sends a message to the `<CUSTOMER-OUTGOING>` topic with the customer's `id` and the transaction's `id`
* At this point, either one of the two branches will be active:
  * If no customer response is receive, after a certain specified time, a timer will trigger the creation of a User Task, assigned to a fraud investigator.
  * If (before the timer expires) a response sent, the process is notified via a signal, containing the customer's response as the payload (either `true`, the customer made the transation or `false`, they did not). From here two additional branches:
    * If the customer acknowledges the transaction, it is automatically approved
    * If not, the transaction is cancelled

The customer notification/response works by:

* Sending a message with the customer and transaction id to `<CUSTOMER-OUTGOING>` topic
* This message is picked by the [notification service](#notification-service), which will send an approriate notification (email, SMS, *etc*) 
* Customer response is sent to the `<CUSTOMER-RESPONSE>` topic, which is picked by the Camel router, which in turn sends it to the appropriate container, using a KIE server REST endpoint, as a signal containing the customer response

## Footnotes

[^0]: Note that this step requires `cluster-admin` permissions.
[^1]: In case you need cluster admin privileges to deploy Strimzi, in which case (in a development setup)  you can run `oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:default:strimzi-cluster-operator`.
[^2]: You might need to add the appropriate server to Maven's `settings.xml` as per 