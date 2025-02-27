# Eclipse Hono

[Eclipse Hono™](https://www.eclipse.org/hono/) provides remote service interfaces for connecting large
numbers of IoT devices to a back end and interacting with them in a uniform way regardless of the device
communication protocol.

This repository contains a *chart* that can be used to install Hono to a Kubernetes cluster using the
[Helm package manager](https://helm.sh).

## Prerequisites

Installing Hono using the chart requires the Helm tool to be installed as described on the
[IoT Packages chart repository prerequisites](https://www.eclipse.org/packages/prereqs/)
page.

In addition, a Kubernetes cluster to deploy to is required.
Hono's [Kubernetes setup guide](https://www.eclipse.org/hono/docs/deployment/create-kubernetes-cluster/)
describes options available for setting up a cluster suitable for running Hono.

The Helm chart is being tested to successfully install on the five most recent Kubernetes versions.

## Installing the chart

Helm can be used to install applications multiple times to the same cluster. Each such
installation is called a *release* in Helm. Each release needs to have a unique name within
a Kubernetes name space.

The instructions below illustrate how Hono can be installed to the `hono` name space
in a Kubernetes cluster using release name `eclipse-hono`. The commands can easily be adapted
to use a different name space or release name.

The target name space in Kubernetes only needs to be created if it doesn't exist yet:

```bash
kubectl create namespace hono
```

The chart can then be installed to name space `hono` using release name `eclipse-hono`:

```bash
helm install --dependency-update --wait -n hono eclipse-hono eclipse-iot/hono
```

## Verifying the Installation

Once installation has completed, Hono's external API endpoints are exposed via corresponding
Kubernetes *Services*. The following command lists all services and their endpoints
(replace `hono` with the name space that you have installed to):

```bash
kubectl get service -n hono

NAME                                            TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)
eclipse-hono-adapter-amqp-vertx                 LoadBalancer   10.109.123.153   10.109.123.153   5672:32672/TCP,5671:32671/TCP
eclipse-hono-adapter-http-vertx                 LoadBalancer   10.99.180.137    10.99.180.137    8080:30080/TCP,8443:30443/TCP
eclipse-hono-adapter-mqtt-vertx                 LoadBalancer   10.102.204.69    10.102.204.69    1883:31883/TCP,8883:30883/TCP
eclipse-hono-artemis                            ClusterIP      10.97.31.154     <none>           5671/TCP
eclipse-hono-dispatch-router                    ClusterIP      10.98.111.236    <none>           5673/TCP
eclipse-hono-dispatch-router-ext                LoadBalancer   10.109.220.100   10.109.220.100   15671:30671/TCP,15672:30672/TCP
eclipse-hono-service-auth                       ClusterIP      10.109.97.44     <none>           5671/TCP
eclipse-hono-service-device-registry            ClusterIP      10.105.190.233   <none>           5671/TCP
eclipse-hono-service-device-registry-ext        LoadBalancer   10.101.42.99     10.101.42.99     28080:31080/TCP,28443:31443/TCP
eclipse-hono-service-device-registry-headless   ClusterIP      None             <none>           <none>
```

The listing above has been retrieved from a Minikube cluster that emulates a load balancer via the `minikube tunnel`
command (refer to the [Minikube docs](https://minikube.sigs.k8s.io/docs/tasks/loadbalancer/) for details).
The service endpoints can be accessed at the *EXTERNAL-IP* addresses and corresponding *PORT(S)*, e.g. 8080 for the
HTTP adapter (*hono-adapter-http-vertx*) and 28080 for the device registry (*hono-service-device-registry*).

The following command assigns the IP address of the device registry service to the `REGISTRY_IP` environment
variable so that it can easily be used from the command line:

```bash
export REGISTRY_IP=$(kubectl get service eclipse-hono-service-device-registry-ext --output='jsonpath={.status.loadBalancer.ingress[0].ip}' -n hono)
```

The following command can then be used to check for the existence of the *DEFAULT_TENANT* which is created as part
of the installation:

```bash
curl -sIX GET http://$REGISTRY_IP:28080/v1/tenants/DEFAULT_TENANT
```

the output should look similar to

```
HTTP/1.1 200 OK
etag: 89d40d26-5956-4cc6-b978-b15fda5d1823
content-type: application/json; charset=utf-8
content-length: 260
```

## Uninstalling the Chart

To uninstall/delete the `eclipse-hono` release from the target name space:

```bash
helm uninstall -n hono eclipse-hono
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

## Configuration

The `values.yaml` file contains all possible configuration values along with documentation.

In order to set a property to a non-default value, the `--set key=value[,key=value]` command line parameter can be passed to
`helm install`. For example:

```bash
helm install --dependency-update --wait -n hono --set useLoadBalancer=false eclipse-hono eclipse-iot/hono
```

Alternatively, one or more YAML files that contain the properties can be provided when installing the chart:

```bash
helm install --dependency-update --wait -n hono -f /path/to/config.yaml -f /path/to/other-config.yaml eclipse-hono eclipse-iot/hono
```


## Installing Prometheus and Grafana

The chart supports installation and configuration of an example Prometheus instance for collecting metrics from Hono's
components and a Grafana instance for visualizing the metrics on dashboards in a web browser.

Both Prometheus and Grafana are completely optional and are not required to run Hono. The following configuration
properties can be used to install the Prometheus and Grafana servers along with Hono:

```bash
helm install --dependency-update --wait -n hono --set prometheus.createInstance=true --set grafana.enabled=true eclipse-hono eclipse-iot/hono
```

### Accessing the Example Grafana Dashboard

Hono comes with an example Grafana dashboard which provides some insight into the messages flowing through the protocol
adapters. The following command needs to be run first in order to forward the Grafana service's endpoint to the local host:

```bash
kubectl port-forward service/eclipse-hono-grafana 3000 -n hono
```

The dashboard can then be opened by pointing your browser to `http://localhost:3000` using credentials `admin:admin`.



## Using specific Container Images

The chart can be customized to use container images other than the default ones.
This can be used to install an older version of the images or to install a Hono milestone
using the chart. It can also be used to install custom built images that need to be
pulled from a different (private) container registry.

The `values.yaml` file contains configuration properties for setting the container
image and tag names to use for Hono's components. The easiest way to override the version
of all Hono components in one go is to set the `honoImagesTag` and/or `honoContainerRegistry`
properties to the desired values during installation.

The following command installs Hono using the standard images published on Docker Hub with tag
*1.9.0* instead of the ones indicated by the chart's *appVersion* property:

```bash
helm install --dependency-update --wait -n hono --set honoImagesTag=1.9.0 eclipse-hono eclipse-iot/hono
```

The following command installs Hono using custom built images published on a private registry with tag
*1.9.0-custom* instead of the ones indicated by the chart's *appVersion* property:

```bash
helm install --dependency-update --wait -n hono --set honoImagesTag=1.9.0-custom --set honoContainerRegistry=my-registry:9090 eclipse-hono eclipse-iot/hono
```

It is also possible to define the image and tag names and container registry for each component separately.
The easiest way to do that is to create a YAML file that specifies the particular properties:

```yaml
deviceRegistryExample:
  # pull custom Device Registry image from private container registry
  imageName: my-hono/hono-service-device-registry-custom
  imageTag: 1.0.0
  containerRegistry: my-private-registry

authServer:
  # pull "older" release from Docker Hub
  imageName: eclipse/hono-service-auth
  imageTag: 1.9.0

# pull standard adapter images in version 1.2.3 from Docker Hub
adapters:
  amqp:
    imageName: eclipse/hono-adapter-amqp-vertx
    imageTag: 1.2.3
  coap:
    imageName: eclipse/hono-adapter-coap-vertx
    imageTag: 1.2.3
  http:
    imageName: eclipse/hono-adapter-http-vertx
    imageTag: 1.2.3
  kura:
    imageName: eclipse/hono-adapter-kura
    imageTag: 1.2.3
  mqtt:
    imageName: eclipse/hono-adapter-mqtt-vertx
    imageTag: 1.2.3
  lora:
    imageName: eclipse/hono-adapter-lora-vertx
    imageTag: 1.2.3
```

Assuming that the file is named `customImages.yaml`, the values can then be passed in to the
Helm `install` command as follows:

```bash
helm install --dependency-update --wait -n hono -f /path/to/customImages.yaml eclipse-hono eclipse-iot/hono
```

## Using a production grade AMQP Messaging Network and Device Registry

The Helm chart by default deploys the example Device Registry that comes with Hono. The example registry provides implementations
of the Tenant, Device Registration, Credentials and Device Connection APIs which can be used for example/demo purposes.

The chart also deploys an example AMQP Messaging Network consisting of a single Apache Qpid Dispatch Router and a single
Apache ActiveMQ Artemis broker.

The protocol adapters are configured to connect to the example messaging network and registry by default.

In a production environment, though, usage of the example registry and messaging network is strongly discouraged and more
sophisticated (custom) implementations of the service APIs should be used.

The Helm chart supports configuration of the protocol adapters to connect to other service implementations than the example registry
and messaging network as described in the following sections.

### Integrating with an existing AMQP Messaging Network

The Helm chart can be configured to use an existing AMQP Messaging Network implementation instead of the example implementation.
In order to do so, the protocol adapters need to be configured with information about the AMQP Messaging Network's endpoint address
and connection parameters.

The easiest way to set these properties is by means of putting them into a YAML file with content like this:

```yaml
# do not deploy example AMQP Messaging Network
amqpMessagingNetworkExample:
  enabled: false

# mount (existing) Kubernetes secret which contains
# credentials for connecting to AMQP network
# into Command Router and protocol adapter containers 
commandRouterService:
  extraSecretMounts:
    amqpNetwork:
      secretName: "my-secret"
      mountPath: "/etc/custom"
adapters:
  http:
    extraSecretMounts:
      amqpNetwork:
        secretName: "my-secret"
        mountPath: "/etc/custom"
  mqtt:
    extraSecretMounts:
      amqpNetwork:
        secretName: "my-secret"
        mountPath: "/etc/custom"
  amqp:
    extraSecretMounts:
      amqpNetwork:
        secretName: "my-secret"
        mountPath: "/etc/custom"

  # provide connection params
  # assuming that "my-secret" contains an "amqp-credentials.properties" file
  amqpMessagingNetworkSpec:
    host: my-custom.amqp-network.org
    port: 5672
    credentialsPath: "/etc/custom/amqp-credentials.properties"
  commandAndControlSpec:
    host: my-custom.amqp-network.org
    port: 5672
    credentialsPath: "/etc/custom/amqp-credentials.properties"
```

Both the *amqpMessagingNetworkSpec* and the *commandAndControlSpec* need to contain Hono client configuration properties
as described in the [client admin guide](https://www.eclipse.org/hono/docs/admin-guide/hono-client-configuration/).
Make sure to adapt/add properties as required by the AMQP Messaging Network.

Note that *my-secret* is expected to already exist in the name space that Hono gets installed to, i.e. the Helm chart
will **not** create this secret.

Assuming that the file is named `customAmqpNetwork.yaml`, the values can then be passed in to the Helm `install`
command as follows:

```bash
helm install --dependency-update --wait -n hono -f /path/to/customAmqpNetwork.yaml eclipse-hono eclipse-iot/hono
```

### Integrating with a custom Device Registry

The Helm chart can be configured to use existing implementations of the Tenant, Device Registration, Credentials and Device Connection APIs
instead of the example device registry.
In order to do so, the protocol adapters need to be configured with information about the service endpoints and connection parameters.

The easiest way to set these properties is by means of putting them into a YAML file with the following content:

```yaml
# do not deploy example Device Registry
deviceRegistryExample:
  enabled: false

adapters:

  # mount (existing) Kubernetes secret which contains
  # credentials for connecting to services
  extraSecretMounts:
  - customRegistry:
      secretName: "my-secret"
      mountPath: "/etc/custom"

  # provide connection params
  # assuming that "my-secret" contains credentials files
  tenantSpec:
    host: my-custom.registry.org
    port: 5672
    credentialsPath: "/etc/custom/tenant-service-credentials.properties"
  deviceRegistrationSpec:
    host: my-custom.registry.org
    port: 5672
    credentialsPath: "/etc/custom/registration-service-credentials.properties"
  credentialsSpec:
    host: my-custom.registry.org
    port: 5672
    credentialsPath: "/etc/custom/credentials-service-credentials.properties"
  deviceConnectionSpec:
    host: my-custom.registry.org
    port: 5672
    credentialsPath: "/etc/custom/device-connection-service-credentials.properties"
```

All of the *specs* need to contain Hono client configuration properties
as described in the [client admin guide](https://www.eclipse.org/hono/docs/admin-guide/hono-client-configuration/).
Make sure to adapt/add properties as required by the custom service implementations.
The information contained in the *specs* will then be used by all protocol adapters that get deployed.
As a consequence, it is not possible to use credentials for the services which are specific to the
individual protocol adapters.

Note that *my-secret* is expected to already exist in the name space that Hono gets installed to, i.e. the Helm chart
will **not** create this secret.

Assuming that the file is named `customRegistry.yaml`, the values can then be passed in to the Helm 3 `install` command
as follows:

```bash
helm install --dependency-update --wait -n hono -f /path/to/customRegistry.yaml eclipse-hono eclipse-iot/hono
```

## Configuring Storage for Command Routing Data

In Hono a place is needed where information about the connection status of devices can be stored.
This kind of information is used for determining how [command & control](https://www.eclipse.org/hono/docs/concepts/command-and-control/)
messages, sent by business applications, can be routed to the protocol adapters that the target devices are connected to.

### Alternative A: Using the Command Router API (default)

Hono's protocol adapters can use the [Command Router API](https://www.eclipse.org/hono/docs/api/command-router/) to supply
device connection information with which a Command Router service component can route command & control messages to the
protocol adapters that the target devices are connected to.

Hono comes with a ready to use implementation of the Command Router API which is used by default when
deploying Hono using the Helm chart:

```bash
helm install --dependency-update --wait -n hono eclipse-hono eclipse-iot/hono 
```

#### Using an Embedded Cache

This will let the Command Router service component use an embedded cache for the command routing data. All data is kept
in-memory only and will therefore be lost after restarting the Command Router service.

**NB** With this configuration, only one Command Router service component instance can be used. For a storage
configuration suitable for production, with the possibility to use multiple instances, use the data grid configuration
as described below.

#### Using a Data Grid

The Command Router service component can also be configured to use a data grid for storing the command routing data.
The Helm chart supports deployment of an example data grid which can be used for experimenting by means of setting the
*dataGridExample.enabled* property to `true`:

```bash
helm install --dependency-update --wait -n hono --set dataGridExample.enabled=true eclipse-hono eclipse-iot/hono 
```

This will deploy the data grid based Command Router service component.

The Command Router service component can also be configured to connect to an already existing data grid.
In this case the *dataGridSpec* property needs to be configured with the data grid's connection information.

### Alternative B: Using the deprecated Device Connection API

The [Device Connection API](https://www.eclipse.org/hono/docs/api/device-connection/) defines a service interface
that protocol adapters can use to store, update and retrieve information about the connection status of devices
dynamically during runtime.

The Device Connection API is deprecated and will be replaced by the Command Router API.

#### File based Example Implementation

Hono's file based example Device Registry component contains a simple in-memory implementation of the Device
Connection API. To use this implementation, deploy the example registry as follows:

```bash
helm install --dependency-update --wait -n hono --set useCommandRouter=false --set deviceRegistryExample.type=file eclipse-hono eclipse-iot/hono 
```

#### Data Grid based Implementation

Hono also contains a production ready, data grid based implementation of the Device Connection API which can be deployed
and used instead of the example implementation. The service component can be deployed by means of setting the
*deviceConnectionService.enabled* property to `true`.

The service requires a connection to a data grid for storing the device connection data.
The Helm chart supports deployment of an example data grid which can be used for experimenting by means of setting the
*dataGridExample.enabled* property to `true`:

```bash
helm install --dependency-update --wait -n hono --set useCommandRouter=false --set deviceConnectionService.enabled=true --set dataGridExample.enabled=true eclipse-hono eclipse-iot/hono 
```

This will deploy the data grid based Device Connection service and configure all protocol adapters to use it instead of
the example Device Registry implementation.

The Device Connection service can also be configured to connect to an already existing data grid. Use the *dataGridSpec*
property for this.

Setting the *deviceConnectionService.enabled* property to `true` and neither setting *dataGridExample.enabled* to `true`
nor configuring an already existing data grid using the *dataGridSpec* property will result in the Device Connection
service using an embedded cache for storage. This is a lightweight deployment option but not suitable for production
purposes.

## Enabling or disabling Protocol Adapters

The Helm chart by default installs the HTTP, MQTT and AMQP protocol adapters.
However, the chart also supports deployment of additional protocol adapters which are still considered experimental or
have been deprecated.
The following table provides an overview of the corresponding configuration properties that need to be set during installation.

| Property                     | Default  | Description                              |
| :--------------------------- | :------- | :--------------------------------------- |
| *adapters.amqp.enabled*      | `true`    | Indicates if the AMQP protocol adapter should be deployed. |
| *adapters.coap.enabled*      | `false`   | Indicates if the CoAP protocol adapter should be deployed. |
| *adapters.http.enabled*      | `true`    | Indicates if the HTTP protocol adapter should be deployed. |
| *adapters.kura.enabled*      | `false`   | Indicates if the deprecated Kura protocol adapter should be deployed. |
| *adapters.lora.enabled*      | `false`   | Indicates if the (experimental) LoRa WAN protocol adapter should be deployed. |
| *adapters.mqtt.enabled*      | `true`    | Indicates if the MQTT protocol adapter should be deployed. |

The following command will deploy the LoRa adapter along with Hono's standard adapters (AMQP, HTTP and MQTT):

```bash
helm install --dependency-update --wait -n hono --set adapters.lora.enabled=true eclipse-hono eclipse-iot/hono
```

## Jaeger Tracing

Hono's components are instrumented using OpenTracing to allow tracking of the distributed processing of messages flowing
through the system. The Hono chart can be configured to report tracing information to the
[Jaeger tracing system](https://www.jaegertracing.io/). The *Spans* reported by the components can then be viewed in a
web browser.

The chart can be configured to deploy and use an example Jaeger back end by means of setting the
*jaegerBackendExample.enabled* property to `true` when running Helm:

~~~sh
helm install --dependency-update --wait -n hono --set jaegerBackendExample.enabled=true eclipse-hono eclipse-iot/hono
~~~

This will create a Jaeger back end instance suitable for testing purposes and will configure all deployed Hono
components to use the Jaeger back end.

Note that this can only be used with the standard Hono images published on Docker Hub with version 1.5.0 or later.
Custom built images need to include the Jaeger client in order to be able to report data to the Jaeger back end.
For the standard Hono components this can be achieved by means of activating the `jaeger` Maven profile during
the build process.

The following command can then be used to return the IP address at which the Jaeger UI can be accessed in a
browser (ensure `minikube tunnel` is running when using minikube):

~~~sh
kubectl get service eclipse-hono-jaeger-query --output="jsonpath={.status.loadBalancer.ingress[0]['hostname','ip']}" -n hono
~~~

If no example Jaeger back end should be deployed but instead an existing Jaeger installation should be used,
the chart's *jaegerAgentConf* property can be set to environment variables which are passed in to
the Jaeger Agent that is deployed with each of Hono's components.

By default, the Jaeger Agent deployed with each of Hono's components is configured to retrieve its sampling strategy
from the Jaeger back end's *Collector* service. The service loads the strategies from the file that the
`SAMPLING_STRATEGIES_FILE` environment variable points to. The example Jaeger back end server's environment variables
can be set via the chart's *jaegerBackendExample.env* property.

By default, the `SAMPLING_STRATEGIES_FILE` variable points to a file which configures all components to sample every span.
A custom file can be used by creating a Kubernetes secret containing the custom file and then configuring the chart's
*jaegerBackendExample.extraSecretMounts* property to mount the secret's files into the Jaeger container where it then
can be used by setting the `SAMPLING_STRATEGIES_FILE` variable accordingly.

Please refer to the [Jaeger documentation](https://www.jaegertracing.io/docs/sampling/#collector-sampling-configuration)
for details regarding the configuration of the sampling strategies.

Note that usage of the sampling strategy of the Collector service is currently not supported when using the quarkus-native
Hono images. In that case the Jaeger Agent deployed with each of Hono's components is configured to sample all traces.

## Using Native Executable Images

The Hono container images that are used by default contain Java byte code that is being executed using a standard Java VM.
For most of the images there also exists a variant that contains a native OS executable that has been created from the
Java byte code using the [Graal project](https://www.graalvm.org)'s *native-image* compiler. These images start up
more quickly than their corresponding JVM based counterparts. However, for these images no *just-in-time* compilation
is taking place during runtime because the byte code has been compiled *ahead of time* already. Consequently, the code
can also not be optimized during runtime which may result in a reduced performance when compared to the JVM based images.

The Helm chart can be configured to use these *native* images by means of setting the `honoImagesType` property
to `quarkus-native` during installation:

```bash
helm install --dependency-update --wait -n hono --set honoImagesType=quarkus-native eclipse-hono eclipse-iot/hono
```

## Using Kafka based Messaging

The chart can be configured to use Kafka as the messaging network instead of an AMQP 1.0 messaging network.
The configuration `messagingNetworkTypes[0]=kafka` deploys Hono configured to use Kafka for messaging.
It is possible to enable both AMQP _and_ Kafka based messaging at the same time using command line parameters
`--set messagingNetworkTypes[0]=amqp --set messagingNetworkTypes[1]=kafka`. Each tenant in Hono can then be
[configured](https://www.eclipse.org/hono/docs/admin-guide/hono-kafka-client-configuration/#configure-for-kafka-based-messaging)
separately to use either Kafka _or_ AMQP for messaging.

The following command provides a quick start for Kafka based messaging (ensure `minikube tunnel` is running when using
Minikube):

```bash
helm install --dependency-update --wait -n hono --set messagingNetworkTypes[0]=kafka --set kafkaMessagingClusterExample.enabled=true --set amqpMessagingNetworkExample.enabled=false eclipse-hono eclipse-iot/hono
```

The parameters enable the deployment of an example Kafka cluster, disable the deployment of the AMQP 1.0 messaging
network and configure adapters and services to use Kafka based messaging.

To use the service type `NodePort` instead of `LoadBalancer`, the following parameters must be added:
`--set useLoadBalancer=false --set kafka.externalAccess.service.type=NodePort`.

### Using a production grade Kafka cluster

If Kafka based messaging is enabled by adding `kafka` to `messagingNetworkTypes`, the Kafka clients need to
be configured with connection information for a Kafka cluster. The Helm chart can deploy an example Kafka cluster.
This is enabled by setting `kafkaMessagingClusterExample.enabled` to `true`. With this setting the chart
deploys a Kafka cluster consisting of a single broker and a single Zookeeper instance and configures the 
protocol adapters to connect to the example cluster.

However, usage of the example Kafka cluster in a production environment is strongly discouraged as it is a single
node cluster which does not provide any redundancy.

The Helm chart can be configured to use an existing Kafka cluster instead of the example deployment.
In order to do so, the protocol adapters need to be configured with information about the bootstrap server addresses
and configuration properties.

The easiest way to set these properties is by means of putting them into a YAML file with content like this:

```yaml
# configure protocol adapters for Kafka messaging
messagingNetworkTypes:
- kafka

# do not deploy example AMQP Messaging Network
amqpMessagingNetworkExample:
  enabled: false

# do not deploy example Kafka cluster
kafkaMessagingClusterExample:
  enabled: false

adapters:
  # provide connection params
  kafkaMessagingSpec:
    commonClientConfig:
      bootstrap.servers: "broker0.my-custom-kafka.org:9092,broker1.my-custom-kafka.org:9092"
```

*adapters.kafkaMessagingSpec* needs to contain configuration properties as described in Hono's
[Kafka client admin guide](https://www.eclipse.org/hono/docs/admin-guide/hono-kafka-client-configuration/).
Make sure to adapt/add properties as required by the Kafka cluster.

Assuming that the file is named `customKafkaCluster.yaml`, the values can then be passed in to the Helm `install`
command as follows:

```bash
helm install --dependency-update --wait -n hono -f /path/to/customKafkaCluster.yaml eclipse-hono eclipse-iot/hono
```
