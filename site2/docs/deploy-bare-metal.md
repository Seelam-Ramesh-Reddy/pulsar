---
id: deploy-bare-metal
title: Deploy a cluster on bare metal
sidebar_label: "Bare metal"
---

:::tip

1. You can use single-cluster Pulsar installation in most use cases, such as experimenting with Pulsar or using Pulsar in a startup or in a single team. If you need to run a multi-cluster Pulsar instance, see the [guide](deploy-bare-metal-multi-cluster).
2. If you want to use all built-in [Pulsar IO](io-overview.md) connectors, you need to download `apache-pulsar-io-connectors`package and install `apache-pulsar-io-connectors` under `connectors` directory in the pulsar directory on every broker node or on every function-worker node if you have run a separate cluster of function workers for [Pulsar Functions](functions-overview).
3. If you want to use [Tiered Storage](concepts-tiered-storage.md) feature in your Pulsar deployment, you need to download `apache-pulsar-offloaders`package and install `apache-pulsar-offloaders` under `offloaders` directory in the Pulsar directory on every broker node. For more details of how to configure this feature, you can refer to the [Tiered storage cookbook](cookbooks-tiered-storage).

:::

Deploying a Pulsar cluster consists of the following steps:

1. Deploy a [ZooKeeper](#deploy-a-zookeeper-cluster) cluster (optional)  
2. Initialize [cluster metadata](#initialize-cluster-metadata)  
3. Deploy a [BookKeeper](#deploy-a-bookkeeper-cluster) cluster  
4. Deploy one or more Pulsar [brokers](#deploy-pulsar-brokers)  

## Preparation

### Requirements

Currently, Pulsar is available for 64-bit **macOS**, **Linux**, and **Windows**. To use Pulsar, you need to install 64-bit JRE/JDK.
For the runtime Java version, please refer to [Pulsar Runtime Java Version Recommendation](https://github.com/apache/pulsar/blob/master/README.md#pulsar-runtime-java-version-recommendation) according to your target Pulsar version.

:::tip

You can reuse existing Zookeeper clusters.

:::

To run Pulsar on bare metal, the following configuration is recommended:

* At least 6 Linux machines or VMs
  * 3 for running [ZooKeeper](https://zookeeper.apache.org)
  * 3 for running a Pulsar broker, and a [BookKeeper](https://bookkeeper.apache.org) bookie
* A single [DNS](https://en.wikipedia.org/wiki/Domain_Name_System) name covering all of the Pulsar broker hosts

:::note

* Broker is only supported on 64-bit JVM.
* If you do not have enough machines, or you want to test Pulsar in cluster mode (and expand the cluster later), You can fully deploy Pulsar on a node on which ZooKeeper, bookie and broker run.
* If you do not have a DNS server, you can use the multi-host format in the service URL instead.
* Each machine in your cluster needs to have the recommended Java version installed (e.g. [Java 17](https://adoptium.net/?variant=openjdk17)). Please refer to [Pulsar Runtime Java Version Recommendation](https://github.com/apache/pulsar/blob/master/README.md#pulsar-runtime-java-version-recommendation) according to your target Pulsar version.

:::

The following is a diagram showing the basic setup:

![alt-text](/assets/pulsar-basic-setup.png)

In this diagram, connecting clients need to communicate with the Pulsar cluster using a single URL. In this case, `pulsar-cluster.acme.com` abstracts over all of the message-handling brokers. Pulsar message brokers run on machines alongside BookKeeper bookies; brokers and bookies, in turn, rely on ZooKeeper.

### Hardware considerations

If you deploy a Pulsar cluster, keep in mind the following basic better choices when you do the capacity planning.

#### ZooKeeper

For machines running ZooKeeper, it is recommended to use less powerful machines or VMs. Pulsar uses ZooKeeper only for periodic coordination-related and configuration-related tasks, not for basic operations. If you run Pulsar on [Amazon Web Services](https://aws.amazon.com/) (AWS), for example, a [t2.small](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/t2-instances.html) instance might likely suffice.

#### Bookies and Brokers

For machines running a bookie and a Pulsar broker, more powerful machines are required. For an AWS deployment, for example, [i3.4xlarge](https://aws.amazon.com/blogs/aws/now-available-i3-instances-for-demanding-io-intensive-applications/) instances may be appropriate. On those machines you can use the following:

* Fast CPUs and 10Gbps [NIC](https://en.wikipedia.org/wiki/Network_interface_controller) (for Pulsar brokers)
* Small and fast [solid-state drives](https://en.wikipedia.org/wiki/Solid-state_drive) (SSDs) or [hard disk drives](https://en.wikipedia.org/wiki/Hard_disk_drive) (HDDs) with a [RAID](https://en.wikipedia.org/wiki/RAID) controller and a battery-backed write cache (for BookKeeper bookies)

#### Hardware recommendations

To start a Pulsar instance, below are the minimum and the recommended hardware settings.

A cluster consists of 3 broker nodes, 3 bookie nodes, and 3 ZooKeeper nodes. The following recommendation is suitable for one node.

- The minimum hardware settings (**250 Pulsar topics**)
   
   Component | CPU|Memory|Storage|Throughput |Rate
   |---|---|---|---|---|---
   Broker|0.2|256 MB|/|Write throughput: 3 MB/s<br /><br />Read throughput: 6 MB/s<br /><br />|Write rate: 350 entries/s<br /><br />Read rate: 650 entries/s
   Bookie|0.2|256 MB|Journal: 8 GB<br /><br />PD-SSDLedger: 16 GB, PD-STANDARD|Write throughput: 2 MB/s<br /><br />Read throughput: 2 MB/s<br /><br />|Write rate: 200 entries/s<br /><br />Read rate: 200 entries/s
   ZooKeeper|0.05|256 MB|Log: 8 GB, PD-SSD<br /><br />Data: 2 GB, PD-STANDARD|/|/

- The recommended hardware settings (**1000 Pulsar topics**)

   Component | CPU|Memory|Storage|Throughput |Rate
   |---|---|---|---|---|---
   Broker|8|8 GB|/|Write throughput: 100 MB/s<br /><br />Read throughput: 200 MB/s<br /><br />|Write rate: 10,000 entries/s<br /><br />Read rate: 20,000 entries/s
   Bookie|4|8GB|Journal: 256 GB<br /><br />PD-SSDLedger: 2 TB, PD-STANDARD|Write throughput: 75 MB/s<br /><br />Read throughput: 75 MB/s<br /><br />|Write rate: 7,500 entries/s<br /><br />Read rate: 7,500 entries/s
   ZooKeeper|1|2 GB|Log: 64 GB, PD-SSD<br /><br />Data: 256 GB, PD-STANDARD|/|/

## Install the Pulsar binary package

> You need to install the Pulsar binary package on each machine in the cluster, including machines running ZooKeeper and BookKeeper.

To get started deploying a Pulsar cluster on bare metal, you need to download a binary tarball release in one of the following ways:

* By clicking on the link below directly, which automatically triggers a download:
  * <a href="pulsar:binary_release_url" download>Pulsar @pulsar:version@ binary release</a>
* From the Pulsar [downloads page](pulsar:download_page_url)
* From the Pulsar [releases page](https://github.com/apache/pulsar/releases/latest) on GitHub
* Using [wget](https://www.gnu.org/software/wget):

```bash

$ wget pulsar:binary_release_url

```

Once you download the tarball, untar it and `cd` into the resulting directory:

```bash

$ tar xvzf apache-pulsar-@pulsar:version@-bin.tar.gz
$ cd apache-pulsar-@pulsar:version@

```

The extracted directory contains the following subdirectories:

Directory | Contains
:---------|:--------
`bin` |[command-line tools](reference-cli-tools) of Pulsar, such as [`pulsar`](reference-cli-tools.md#pulsar) and [`pulsar-admin`](/tools/pulsar-admin/)
`conf` | Configuration files for Pulsar, including for [broker configuration](reference-configuration.md#broker), [ZooKeeper configuration](reference-configuration.md#zookeeper), and more
`data` | The data storage directory that ZooKeeper and BookKeeper use
`lib` | The [JAR](https://en.wikipedia.org/wiki/JAR_(file_format)) files that Pulsar uses
`logs` | Logs that the installation creates

## [Install Builtin Connectors (optional)](/standalone.md#install-builtin-connectors-optional)

> Since Pulsar release `2.1.0-incubating`, Pulsar provides a separate binary distribution, containing all the `builtin` connectors.
> To enable the `builtin` connectors (optional), you can follow the instructions below.

To use `builtin` connectors, you need to download the connectors tarball release on every broker node in one of the following ways :

* by clicking the link below and downloading the release from an Apache mirror:

  * <a href="pulsar:connector_release_url" download>Pulsar IO Connectors @pulsar:version@ release</a>

* from the Pulsar [downloads page](pulsar:download_page_url)
* from the Pulsar [releases page](https://github.com/apache/pulsar/releases/latest)
* using [wget](https://www.gnu.org/software/wget):

  ```shell
  
  $ wget pulsar:connector_release_url/{connector}-@pulsar:version@.nar
  
  ```

Once you download the .nar file, copy the file to directory `connectors` in the pulsar directory. 
For example, if you download the connector file `pulsar-io-aerospike-@pulsar:version@.nar`:

```bash

$ mkdir connectors
$ mv pulsar-io-aerospike-@pulsar:version@.nar connectors

$ ls connectors
pulsar-io-aerospike-@pulsar:version@.nar
...

```

## [Install Tiered Storage Offloaders (optional)](/standalone.md#install-tiered-storage-offloaders-optional)

> Since Pulsar release `2.2.0`, Pulsar releases a separate binary distribution, containing the tiered storage offloaders.
> If you want to enable tiered storage feature, you can follow the instructions as below; otherwise you can
> skip this section for now.

To use tiered storage offloaders, you need to download the offloaders tarball release on every broker node in one of the following ways:

* by clicking the link below and downloading the release from an Apache mirror:

  * <a href="pulsar:offloader_release_url" download>Pulsar Tiered Storage Offloaders @pulsar:version@ release</a>

* from the Pulsar [downloads page](pulsar:download_page_url)
* from the Pulsar [releases page](https://github.com/apache/pulsar/releases/latest)
* using [wget](https://www.gnu.org/software/wget):

  ```shell
  
  $ wget pulsar:offloader_release_url
  
  ```

Once you download the tarball, in the Pulsar directory, untar the offloaders package and copy the offloaders as `offloaders` in the Pulsar directory:

```bash

$ tar xvfz apache-pulsar-offloaders-@pulsar:version@-bin.tar.gz

// you can find a directory named `apache-pulsar-offloaders-@pulsar:version@` in the pulsar directory
// then copy the offloaders

$ mv apache-pulsar-offloaders-@pulsar:version@/offloaders offloaders

$ ls offloaders
tiered-storage-jcloud-@pulsar:version@.nar

```

For more details of how to configure tiered storage feature, you can refer to the [Tiered storage cookbook](cookbooks-tiered-storage.md)


## Deploy a ZooKeeper cluster

> If you already have an existing zookeeper cluster and want to use it, you can skip this section.

[ZooKeeper](https://zookeeper.apache.org) manages a variety of essential coordination-related and configuration-related tasks for Pulsar. To deploy a Pulsar cluster, you need to deploy ZooKeeper first. A 3-node ZooKeeper cluster is the recommended configuration. Pulsar does not make heavy use of ZooKeeper, so the lightweight machines or VMs should suffice for running ZooKeeper.

To begin, add all ZooKeeper servers to the configuration specified in [`conf/zookeeper.conf`](reference-configuration.md#zookeeper) (in the Pulsar directory that you create [above](#install-the-pulsar-binary-package)). The following is an example:

```properties

server.1=zk1.us-west.example.com:2888:3888
server.2=zk2.us-west.example.com:2888:3888
server.3=zk3.us-west.example.com:2888:3888

```

> If you only have one machine on which to deploy Pulsar, you only need to add one server entry in the configuration file.

> If your machines are behind NAT use 0.0.0.0 as server entry for the local address. If the node use external IP in configuration for itself, behind NAT, zookeper service won't start because it tries to put a listener on an external ip that the linux box doesn't own. Using 0.0.0.0 start a listener on ALL ip, so that NAT network traffic can reach it.

Example of configuration on _server.3_

```properties

server.1=zk1.us-west.example.com:2888:3888
server.2=zk2.us-west.example.com:2888:3888
server.3=0.0.0.0:2888:3888

```

On each host, you need to specify the ID of the node in the `myid` file, which is in the `data/zookeeper` folder of each server by default (you can change the file location via the [`dataDir`](reference-configuration.md#zookeeper-dataDir) parameter).

> See the [Multi-server setup guide](https://zookeeper.apache.org/doc/r3.4.10/zookeeperAdmin.html#sc_zkMulitServerSetup) in the ZooKeeper documentation for detailed information on `myid` and more.

For example, on a ZooKeeper server like `zk1.us-west.example.com`, you can set the `myid` value as follows:

```bash

$ mkdir -p data/zookeeper
$ echo 1 > data/zookeeper/myid

```

On `zk2.us-west.example.com`, the command is `echo 2 > data/zookeeper/myid` and so on.

Once you add each server to the `zookeeper.conf` configuration and have the appropriate `myid` entry, you can start ZooKeeper on all hosts (in the background, using nohup) with the [`pulsar-daemon`](reference-cli-tools.md#pulsar-daemon) CLI tool:

```bash

$ bin/pulsar-daemon start zookeeper

```

> If you plan to deploy Zookeeper with the Bookie on the same node, you need to start zookeeper by using different stats
> port by configuring the `metricsProvider.httpPort` in zookeeper.conf.

## Initialize cluster metadata

Once you deploy ZooKeeper for your cluster, you need to write some metadata to ZooKeeper. You only need to write this data **once**.

You can initialize this metadata using the [`initialize-cluster-metadata`](reference-cli-tools.md#pulsar-initialize-cluster-metadata) command of the [`pulsar`](reference-cli-tools.md#pulsar) CLI tool. This command can be run on any machine in your Pulsar cluster, so the metadata can be initialized from a ZooKeeper, broker, or bookie machine. The following is an example:

```shell

$ bin/pulsar initialize-cluster-metadata \
  --cluster pulsar-cluster-1 \
  --metadata-store zk:zk1.us-west.example.com:2181,zk2.us-west.example.com:2181/my-chroot-path \
  --configuration-metadata-store zk:zk1.us-west.example.com:2181,zk2.us-west.example.com:2181/my-chroot-path \
  --web-service-url http://pulsar.us-west.example.com:8080 \
  --web-service-url-tls https://pulsar.us-west.example.com:8443 \
  --broker-service-url pulsar://pulsar.us-west.example.com:6650 \
  --broker-service-url-tls pulsar+ssl://pulsar.us-west.example.com:6651

```

As you can see from the example above, you will need to specify the following:

Flag | Description
:----|:-----------
`--cluster` | A name for the cluster
`--metadata-store` | A "local" metadata store connection string for the cluster. This connection string only needs to include *one* machine in the ZooKeeper cluster.
`--configuration-metadata-store` | The configuration metadata store connection string for the entire instance. As with the `--metadata-store` flag, this connection string only needs to include *one* machine in the ZooKeeper cluster.
`--web-service-url` | The web service URL for the cluster, plus a port. This URL should be a standard DNS name. The default port is 8080 (you had better not use a different port).
`--web-service-url-tls` | If you use [TLS](security-tls-transport), you also need to specify a TLS web service URL for the cluster. The default port is 8443 (you had better not use a different port).
`--broker-service-url` | A broker service URL enabling interaction with the brokers in the cluster. This URL should not use the same DNS name as the web service URL but should use the `pulsar` scheme instead. The default port is 6650 (you had better not use a different port).
`--broker-service-url-tls` | If you use [TLS](security-tls-transport), you also need to specify a TLS web service URL for the cluster as well as a TLS broker service URL for the brokers in the cluster. The default port is 6651 (you had better not use a different port).


> If you do not have a DNS server, you can use multi-host format in the service URL with the following settings:
>

> ```shell
> 
> --web-service-url http://host1:8080,host2:8080,host3:8080 \
> --web-service-url-tls https://host1:8443,host2:8443,host3:8443 \
> --broker-service-url pulsar://host1:6650,host2:6650,host3:6650 \
> --broker-service-url-tls pulsar+ssl://host1:6651,host2:6651,host3:6651
>
> 
> ```

>
> If you want to use an existing BookKeeper cluster, you can add the `--existing-bk-metadata-service-uri` flag as follows:
>

> ```shell
> 
> --existing-bk-metadata-service-uri "zk+null://zk1:2181;zk2:2181/ledgers" \
> --web-service-url http://host1:8080,host2:8080,host3:8080 \
> --web-service-url-tls https://host1:8443,host2:8443,host3:8443 \
> --broker-service-url pulsar://host1:6650,host2:6650,host3:6650 \
> --broker-service-url-tls pulsar+ssl://host1:6651,host2:6651,host3:6651
>
> 
> ```

> You can obtain the metadata service URI of the existing BookKeeper cluster by using the `bin/bookkeeper shell whatisinstanceid` command. You must enclose the value in double quotes since the multiple metadata service URIs are separated with semicolons.

## Deploy a BookKeeper cluster

[BookKeeper](https://bookkeeper.apache.org) handles all persistent data storage in Pulsar. You need to deploy a cluster of BookKeeper bookies to use Pulsar. You can choose to run a **3-bookie BookKeeper cluster**.

You can configure BookKeeper bookies using the [`conf/bookkeeper.conf`](reference-configuration.md#bookkeeper) configuration file. The most important step in configuring bookies for our purposes here is ensuring that [`zkServers`](reference-configuration.md#bookkeeper-zkServers) is set to the connection string for the ZooKeeper cluster. The following is an example:

```properties

zkServers=zk1.us-west.example.com:2181,zk2.us-west.example.com:2181,zk3.us-west.example.com:2181

```

Once you appropriately modify the `zkServers` parameter, you can make any other configuration changes that you require. You can find a full listing of the available BookKeeper configuration parameters [here](reference-configuration.md#bookkeeper). However, consulting the [BookKeeper documentation](https://bookkeeper.apache.org/docs/next/reference/config/) for a more in-depth guide might be a better choice.

Once you apply the desired configuration in `conf/bookkeeper.conf`, you can start up a bookie on each of your BookKeeper hosts. You can start up each bookie either in the background, using [nohup](https://en.wikipedia.org/wiki/Nohup), or in the foreground.

To start the bookie in the background, use the [`pulsar-daemon`](reference-cli-tools.md#pulsar-daemon) CLI tool:

```bash

$ bin/pulsar-daemon start bookie

```

To start the bookie in the foreground:

```bash

$ bin/pulsar bookie

```

You can verify that a bookie works properly by running the `bookiesanity` command on the [BookKeeper shell](reference-cli-tools.md#shell):

```bash

$ bin/bookkeeper shell bookiesanity

```

This command creates an ephemeral BookKeeper ledger on the local bookie, writes a few entries, reads them back, and finally deletes the ledger.

After you start all the bookies, you can use `simpletest` command for [BookKeeper shell](reference-cli-tools.md#shell) on any bookie node, to verify all the bookies in the cluster are up running.

```bash

$ bin/bookkeeper shell simpletest --ensemble <num-bookies> --writeQuorum <num-bookies> --ackQuorum <num-bookies> --numEntries <num-entries>

```

This command creates a `num-bookies` sized ledger on the cluster, writes a few entries, and finally deletes the ledger.


## Deploy Pulsar brokers

Pulsar brokers are the last thing you need to deploy in your Pulsar cluster. Brokers handle Pulsar messages and provide the administrative interface of Pulsar. A good choice is to run **3 brokers**, one for each machine that already runs a BookKeeper bookie.

### Configure Brokers

The most important element of broker configuration is ensuring that each broker is aware of the ZooKeeper cluster that you have deployed. Ensure that the [`metadataStoreUrl`](reference-configuration.md#broker) and [`configurationMetadataStoreUrl`](reference-configuration.md#broker) parameters are correct. In this case, since you only have 1 cluster and no configuration store setup, the `configurationMetadataStoreUrl` point to the same `metadataStoreUrl`.

```properties

metadataStoreUrl=zk1.us-west.example.com:2181,zk2.us-west.example.com:2181,zk3.us-west.example.com:2181
configurationMetadataStoreUrl=zk1.us-west.example.com:2181,zk2.us-west.example.com:2181,zk3.us-west.example.com:2181

```

You also need to specify the cluster name (matching the name that you provided when you [initialize the metadata of the cluster](#initialize-cluster-metadata)):

```properties

clusterName=pulsar-cluster-1

```

In addition, you need to match the broker and web service ports provided when you initialize the metadata of the cluster (especially when you use a different port than the default):

```properties

brokerServicePort=6650
brokerServicePortTls=6651
webServicePort=8080
webServicePortTls=8443

```

> If you deploy Pulsar in a one-node cluster, you should update the replication settings in `conf/broker.conf` to `1`.
>

> ```properties
> 
> # Number of bookies to use when creating a ledger
> managedLedgerDefaultEnsembleSize=1
>
> # Number of copies to store for each message
> managedLedgerDefaultWriteQuorum=1
> 
> # Number of guaranteed copies (acks to wait before write is complete)
> managedLedgerDefaultAckQuorum=1
>
> 
> ```


### Enable Pulsar Functions (optional)

If you want to enable [Pulsar Functions](functions-overview), you can follow the instructions as below:

1. Edit `conf/broker.conf` to enable functions worker, by setting `functionsWorkerEnabled` to `true`.

   ```conf
   
   functionsWorkerEnabled=true
   
   ```

2. Edit `conf/functions_worker.yml` and set `pulsarFunctionsCluster` to the cluster name that you provide when you [initialize the metadata of the cluster](#initialize-cluster-metadata). 

   ```conf
   
   pulsarFunctionsCluster: pulsar-cluster-1
   
   ```

If you want to learn more options about deploying the functions worker, check out [Deploy and manage functions worker](functions-worker).

### Start Brokers

You can then provide any other configuration changes that you want in the [`conf/broker.conf`](reference-configuration.md#broker) file. Once you decide on a configuration, you can start up the brokers for your Pulsar cluster. Like ZooKeeper and BookKeeper, you can start brokers either in the foreground or in the background, using nohup.

You can start a broker in the foreground using the [`pulsar broker`](reference-cli-tools.md#pulsar-broker) command:

```bash

$ bin/pulsar broker

```

You can start a broker in the background using the [`pulsar-daemon`](reference-cli-tools.md#pulsar-daemon) CLI tool:

```bash

$ bin/pulsar-daemon start broker

```

Once you successfully start up all the brokers that you intend to use, your Pulsar cluster should be ready to go!

## Connect to the running cluster

Once your Pulsar cluster is up and running, you should be able to connect with it using Pulsar clients. One such client is the [`pulsar-client`](reference-cli-tools.md#pulsar-client) tool, which is included with the Pulsar binary package. The `pulsar-client` tool can publish messages to and consume messages from Pulsar topics and thus provide a simple way to make sure that your cluster runs properly.

To use the `pulsar-client` tool, first modify the client configuration file in [`conf/client.conf`](reference-configuration.md#client) in your binary package. You need to change the values for `webServiceUrl` and `brokerServiceUrl`, substituting `localhost` (which is the default), with the DNS name that you assign to your broker/bookie hosts. The following is an example:

```properties

webServiceUrl=http://us-west.example.com:8080
brokerServiceurl=pulsar://us-west.example.com:6650

```

> If you do not have a DNS server, you can specify multi-host in service URL as follows:
>

> ```properties
> 
> webServiceUrl=http://host1:8080,host2:8080,host3:8080
> brokerServiceurl=pulsar://host1:6650,host2:6650,host3:6650
>
> 
> ```


Once that is complete, you can publish a message to the Pulsar topic:

```bash

$ bin/pulsar-client produce \
  persistent://public/default/test \
  -n 1 \
  -m "Hello Pulsar"

```

This command publishes a single message to the Pulsar topic. In addition, you can subscribe to the Pulsar topic in a different terminal before publishing messages as below:

```bash

$ bin/pulsar-client consume \
  persistent://public/default/test \
  -n 100 \
  -s "consumer-test" \
  -t "Exclusive"

```

Once you successfully publish the above message to the topic, you should see it in the standard output:

```bash

----- got message -----
Hello Pulsar

```

## Run Functions

> If you have [enabled](#enable-pulsar-functions-optional) Pulsar Functions, you can try out the Pulsar Functions now.

Create an ExclamationFunction `exclamation`.

```bash

bin/pulsar-admin functions create \
  --jar examples/api-examples.jar \
  --classname org.apache.pulsar.functions.api.examples.ExclamationFunction \
  --inputs persistent://public/default/exclamation-input \
  --output persistent://public/default/exclamation-output \
  --tenant public \
  --namespace default \
  --name exclamation

```

Check whether the function runs as expected by [triggering](functions-deploy-trigger.md) the function.

```bash

bin/pulsar-admin functions trigger --name exclamation --trigger-value "hello world"

```

You should see the following output:

```shell

hello world!

```

