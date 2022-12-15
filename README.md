# Intermediate Microservice Orchestration Process template

This Repository hosts the implementation of the intermediate Microservice Orchestration template which is provided within [Camunda Platform 8](https://accounts.cloud.camunda.io/signup)

[] ADD final diagram

## How to deploy the process

There are several ways how the template could be deployed to your Camunda Platform 8 Cluster.

### Via the process templates catalog available in Camunda Web Modeler (as part of Camunda Platform 8 SaaS) (Web browser)

The easiest option to deploy the process template is by doing so via the process template picker available in Camunda Platform 8. Follow this [documentation](https://docs.camunda.io/docs/next/components/modeler/web-modeler/launch-cloud-modeler/) and choose the option to "Create model from template"
 
### Via zbctl (Terminal / Command line)

zbctl is the command line interface to interact with Camunda Platform 8. It offers to deploy resources to a Camunda Platform 8 Cluster. Follow the [installation documentation](https://docs.camunda.io/docs/next/apis-clients/cli-client/) to properly set it up and connect it to your Cluster.

Then deploy the BPMN file via:

```
zbctl deploy order-fulfillment-process.bpmn
```

### Via Camunda Desktop Modeler (Desktop application)

- [Install](https://docs.camunda.io/docs/next/components/modeler/desktop-modeler/install-the-modeler/) the Desktop Modeler

- Open the process diagram with the Desktop Modeler

- Follow our [deployment guide](https://docs.camunda.io/docs/next/components/modeler/desktop-modeler/connect-to-camunda-cloud/).

## How to start a process

Typically, an order fulfillment process like the one at hand would be started by a customer that places an order via some kind of user interface. For example this could be a website (like an online shop), that serves as the frontend for customers.

In this example, we are going without having such a frontend at hand and use the zbctl CLI instead. In general, zbctl is the command line interface to interact with Camunda Platform 8 that allows users to create and read resources of all kinds inside the Zeebe broker that executes the process. One function zbctl offers is to publish messages inside the Zeebe cluster. This is a great feature that helps us to get quickly started and this way being able to mock the user frontend mentioned above.

To start a process instance via the message start event "Order received", use the following zbctl command

```
zbctl publish message Msg_OrderReceived --correlationKey "1" --variables "{\"orderId\": \"order1\",\"articleId\": \"article1\", \"articleQuantity\": 3, \"articlePrice\": 15}"
```

This publishes the message labeled with "Order received" in our model with the message name "Msg_OrderReceived". To specify the payload, in our case the articles and their quantities, we can make use of the --variables-flag.

The correlationKey-flag allows us to tell Zeebe for which order an instance should be created

However, if you are interested in developing a custom frontend have a look at these resources:

- Mention Bernds project as an alternative and implement (https://github.com/camunda-community-hub/camunda-8-examples/tree/main/start-form-springboot)


## How to use the workers

## Retrieve Payment worker (Java)

Configure your Camunda Platform 8 connection.

Connections to the Camunda SaaS can be easily configured, create the following entries in your 

`src/main/resources/application.properties`:

```properties
zeebe.client.cloud.cluster-id=xxx
zeebe.client.cloud.client-id=xxx
zeebe.client.cloud.client-secret=xxx
zeebe.client.cloud.region=bru-2
```

You can also configure the connection to a self-managed Zeebe broker:

```properties
zeebe.client.broker.gateway-address=127.0.0.1:26500
zeebe.client.security.plaintext=true
```

Run the application via
```
./mvnw spring-boot:run
```

### How the let the worker fail (i.e. how to throw the BPMN error)

In order to simulate that the payment failed while retrieving the payment, we need the Java job worker for the "Retrieve payment" task to throw a BPMN error.

Therefore you can set a flag when starting a process instance. Just set a variable `paymentShouldFail` with the value `true`.

The corresponding zbctl command would then be:

```
zbctl publish message Msg_OrderReceived --correlationKey "1" --variables "{\"orderId\": \"order1\",\"articleId\": \"article1\", \"articleQuantity\": 3, \"articlePrice\": 15, \"paymentShouldFail\":true}"
```

This way to worker will eventually throw and trigger hte "Payment failed" BPMN Error via 

```Java
throw new ZeebeBpmnError("paymentFailed", "The payment failed due to an error");
```

which leads to the user task "Inform customer about failed payment"

[Camunda Tasklist](https://docs.camunda.io/docs/next/components/tasklist/introduction-to-tasklist/) can be used to work on this user task.

### Bonus: Java job workers for "Fetch goods" and "Ship goods"

There are two more job workers defined, but disabled. They could be activated and used to complete the whole order fulfilment process via Java workers.
In order to activate them, just enable them by setting the respective Job worker annotation to true:

```
@JobWorker(enabled = true)
```

### How to write own Java job workers

The Java job workers in this template were created with the [Spring Zeebe Client](https://github.com/camunda-community-hub/spring-zeebe/). Spring Zeebe is a wrapper around the [Zeebe Java Client](https://docs.camunda.io/docs/apis-clients/java-client/). Have a look at the [Spring Zeebe readme and examples](https://github.com/camunda-community-hub/spring-zeebe/) to learn what Spring Zeebe offers to write own Java job workers

## Fetch Goods worker (Go)

Set the connection settings and client credentials as environment variables:

```bash
export ZEEBE_ADDRESS='[Zeebe API]'
export ZEEBE_CLIENT_ID='[Client ID]'
export ZEEBE_CLIENT_SECRET='[Client Secret]'
export ZEEBE_AUTHORIZATION_SERVER_URL='[OAuth API]'
```

Run the worker via

```bash
go run main.go
```

### How to write own Go job workers

The Go job worker in this template was created with the [Go Client](https://docs.camunda.io/docs/next/apis-clients/go-client/). 

Have a look at the [Getting started with the Go client](https://docs.camunda.io/docs/next/apis-clients/go-client/go-get-started/) tutorial to learn how to use and write own Go job workers with it.

### How to time-out the "Fetch goods" task (i.e. make use of the attached timer event)

Even though the label in the process model says "2 days past", the technical attribute behind the [non-interrupting attached boundary event](https://docs.camunda.io/docs/next/components/modeler/bpmn/timer-events/#timer-boundary-events) is only set to 5 minutes (=PT5M), in order to be able to easily simulate the timeout of the task.

So in order to simulate it, you don`t need to do something actively but just wait for 5 minutes.

Learn more about setting [timer durations](https://docs.camunda.io/docs/next/components/modeler/bpmn/timer-events/#time-duration)

The correct timer duration for a 2 day timer would be `PT2D`.

After the timer was fired, the process still waits for the completion of the "Fetch goods" task and addtionally creates the user task "Inform customer about delay"

[Camunda Tasklist](https://docs.camunda.io/docs/next/components/tasklist/introduction-to-tasklist/) can be used to work on this user task.

## Ship Goods worker

