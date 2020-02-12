
## Attach an AMQ Broker

The previous message flow was broker-less, it requires both producer and consumer to be active for the traffic to flow.

The above works well for systems designed to run synchronously. You might however require to accommodate asynchronous clients in need to send messages when no active consumer is running.

You can attach an AMQ Broker to the routing layer to enable store & forward messaging. This would enable the messaging layer to persist messages and retain them until a consumer connects to the network and consumes them.

![](./images/interconnect-brokered.png "An AMQ Broker is attached to the mesh")

In the instructions below you will be adhering an AMQ Broker to Region-2 where messages will be persisted before they cross to Region-1. Please note that the entire *Routing Mesh* acts as a single logical cluster, therefore, the location of where the AMQ Broker is attached is not critical.

<br/>

1. #### Install AMQ's Broker Operator

	As a standard user, ensure you're working in `Cluster-2` where we will deploy the broker.
	
	>**Note**: the broker could be connected anywhere in the *Interconnect* layer.

	   oc project amq-cluster2


	Install AMQ's Broker Operator:

	- Web Console ➡ Operators ➡ OperatorHub ➡ AMQ Broker ➡ Install 

		(at the time of writing: v0.9.1)

	Select `amq-cluster2` as the target namespace, and click '*Subscribe*'.

	>**Be patient:** this action may take some time as *OpenShift* may need to pull the *Operator*'s image from a remote repository.

	This will trigger the *Operator*'s installation. To view the running pod execute:

	   oc get pods -n amq-cluster2

	You should see something similar to:

	```
	NAME                                    READY   STATUS    RESTARTS   AGE
	amq-broker-operator-84f7fcc8b-2x225     1/1     Running   0          28s
	```

<br/>

1. #### Deploy a Broker instance

	Once the *Operator* is running, deploy the Broker:

	From namespace `amq-cluster2`:

	- Operators ➡ Installed Operators ➡ AMQ Broker ➡ AMQ Broker ➡ Create Active MQArtemis

	<br/>

	Review the default YAML definition:

	```yaml
	metadata: name: broker1
	```
	By default, the broker does not include an AMQP acceptor, we need to explicitly define one:

	```yaml
	spec:
	  acceptors:
	    - name: amqp
	      port: 5672
	      protocols: amqp
	```

	The YAML result should look like:
	```yaml
	apiVersion: broker.amq.io/v2alpha1
	kind: ActiveMQArtemis
	metadata:
	  name: broker1
	  namespace: amq-cluster2
	spec:
	  deploymentPlan:
	    size: 1
	    image: 'registry.redhat.io/amq7/amq-broker:7.5'
	  acceptors:
	    - name: amqp
	      port: 5672
	      protocols: amqp
	```

	Click '*Create*' to kick off the deployment.

	>**Be patient:** this action may take some time as *OpenShift* may need to pull the *Broker*'s image from a remote repository.

	The *Operator* will deploy the Broker as per the definition above, and also create other necessary elements. A service will be created for clients to access the broker with the following pattern:

	   {broker-name}-hdls-svc

	If you defined the name `broker1` then the service name would be: 

	   broker1-hdls-svc

	>**OPTIONAL**: If you'd like to access the Broker's web console you can create a Route, navigate to:
	- Web Console ➡ Networking ➡ Routes ➡ Create Route
		```
		Name:        amq-console
		Service:     broker1-hdls-svc
		Target Port: 8161 -> 8161 (TCP)
		```
		Click '*Create*'.

<br/>

1. #### Attach the broker to the routing layer

	From namespace `amq-cluster2` navigate to:

	- Web Console ➡ Operators ➡ Installed Operators ➡ AMQ Interconnect ➡ AMQ Interconnect ➡ cluster2-router-mesh ➡ YAML

	<br/>

	Review the default YAML definition:
	```yaml
	spec: listeners: - port 5671
	```
	Include the following elements:
	```yaml
	spec:
	  connectors:
	    - name: my-broker
	      host: broker1-hdls-svc.amq-cluster2.svc.cluster.local
	      port: 5672
	      routeContainer: true
	  linkRoutes:
	    - prefix: test
	      direction: in
	      connection: my-broker
	    - prefix: test
	      direction: out
	      connection: my-broker
	```
	>**Note**: the field `host` represents the fully service DNS address pointing to the broker's service.

	Click '*Save*'. The *Operator* watching the cluster will reconfigure *Interconnect* to open a connection to the defined broker.

<br/>

## Test Brokered Interconnect flow:

Since we've attached a Broker to Interconnect, the same messaging flow will now go through the broker. To demonstrate that execute the following steps:

1. #### Run the producer

	Ensure both the client and server are stopped. Then start only the producer (client).

	You will observe this round the producer immediately starts sending messages, the attached broker is providing credits to the client.

	After some messages are sent, stop the client.


1. #### Run the consumer

	Now, start the consumer (server). You will observe how it is able to consume all the messages stored in the Broker until there's none left.

