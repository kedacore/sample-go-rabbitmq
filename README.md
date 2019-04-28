# RabbitMQ consumer and sender

A simple docker container that will receive messages from a RabbitMQ queue and scale via KEDA.  The reciever will receive a single message at a time (per instance), and sleep for 1 second to simulate performing work.  When adding a massive amount of queue messages, KEDA will drive the container to scale out according to the event source (RabbitMQ).

## Pre-requisites

* Kubernetes cluster
* [KEDA](https://github.com/kedacore/keda) installed on the cluster

## Setup

This setup will go through creating a RabbitMQ queue on the cluster and deploying this consumer with the `ScaledObject` to scale via KEDA.  If you already have RabbitMQ you can use your existing queues.

First you should clone the project:

```cli
git clone https://github.com/kedacore/sample-go-rabbitmq
cd sample-go-rabbitmq
```

### Creating a RabbitMQ

#### [Install Helm](https://helm.sh/docs/using_helm/)

#### Install RabbitMQ via Helm

```cli
helm install --name rabbitmq --set rabbitmq.username=user,rabbitmq.password=PASSWORD stable/rabbitmq
```

### Deploying this RabbitMQ consumer

#### Deploy the consumer
```cli
kubectl apply -f deploy/deploy-consumer.yaml
```

#### Validate it has deployed
```cli
kubectl get deploy
```

You should see `rabbitmq-consumer` deployment with 0 pods as there currently aren't any queue messages.  It is scale to zero.

```
NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
rabbitmq-consumer   0         0         0            0           3s
```

### Publishing messages to the queue

#### Deploy the publisher job
The following job will publish 600 messages to a "hello" queue the container is listening to. You can modify the exact number of messages in the yaml file.

```cli
kubectl apply -f deploy/deploy-publisher-job.yaml
```

#### Validate the container scales
```cli
kubectl get deploy -w
```

You can watch the pods spin up and start to process queue messages.  As the message length continues to increase, more pods will be pro-actively added.  

You can see the number of messages vs the target per pod as well:
```cli
kubectl get hpa
```

After the queue is empty and the specified cooldown period (a property of the `ScaledObject`) the last replica will scale back down to zero.

## Cleanup resources

```cli
kubectl delete job rabbitmq-publish
kubectl delete deploy rabbitmq-consumer
kubectl detele ScaledObject rabbitmq-consumer
helm delete rabbitmq
```