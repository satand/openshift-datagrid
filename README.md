# Datagrid

## Modify the number of the cluster nodes and "stop and start" the cluster

With the cluster in running state we can change dynamically the number of cluster nodes using patch command.
Here we are decreasing to a single node:
```bash
oc patch infinispan infinispan --type='merge' -p '{"spec": {"replicas": 1}}'
```
The PVC relative to the terminated instance will be removed

Then we can increase to two nodes:
```bash
oc patch infinispan infinispan --type='merge' -p '{"spec": {"replicas": 2}}'
```
The PVC relative to the created instance will be created

Similarly, we can stop a cluster with:
```bash
oc patch infinispan infinispan --type='merge' -p '{"spec": {"replicas": 0}}'
```
In this case the relative PVCs are not deleted, but they remain waiting the cluster restart.
To restart the cluster correctly its important to check the number of previous cluster nodes using this command (it gives a result if the cluster is stopped):
```bash
oc get infinispan infinispan -o=jsonpath='{.status.replicasWantedAtRestart}'
```
then we can use that result to patch the cluster replicas:
```bash
oc patch infinispan infinispan --type='merge' -p "{\"spec\": {\"replicas\": $(oc get infinispan infinispan -o=jsonpath='{.status.replicasWantedAtRestart}')}}"
```

## Apply a Datagrid custom configuration
Using a configMap with a key infinispan-config.\[xml|yaml|json\] we can add a Datagrid "user" custom configuration to the default Datagrid "operator" configuration. This default configuration is in the configMap named infinispan-configuration and it's managed by the Datagrid operator. When we add a user custom configuration we have to pay attention to that configuration because if it contains setting outside the cacheContainer section it could affect the operator configuration on other sections such as endpoints, security realms, and cluster transport.

To add the user custom configuration we have to add in the Infinispan CR the field configMapName with value the name of the custom configuration configMap.
Watching the node pod definition we can see the utilization of both configMaps (user and operation):

```yaml
apiVersion: v1
kind: Pod
metadata:
  ...
  labels:
    app: infinispan-pod
    clusterName: infinispan
    ...
  name: infinispan-0
  namespace: datagrid-demo
spec:
  ...
  containers:
  - args:
    - -l
    - /opt/infinispan/server/conf/operator/log4j.xml
    - -c
    - operator/infinispan-base.xml
    - -c
    - user/infinispan-config.yaml
    - -c
    - operator/infinispan-admin.xml
    name: infinispan
    volumeMounts:
    - mountPath: /opt/infinispan/server/conf/operator
      name: config-volume
    - mountPath: /opt/infinispan/server/conf/user
      name: user-conf-volume
    ...
  volumes:
  - configMap:
      name: infinispan-configuration
    name: config-volume
  - configMap:
      name: cluster-config
    name: user-conf-volume
  ...
```

## Modify logging category level

The logging changes in the CR Infinispan don't start an automatic statefulset update, so after the logging configuration change in the CR Infinispan (the log4j.xml key in the configmap infinispan-configuration will be automatically updated), we will have to command a rollout restart of the cluster statefulset to see the new logging settings applied:
```bash
oc rollout restart statefulset infinispan
```

## Persistent or ephemeral storage

When we use the persistent storage (it is the default case), we have to define the dimension of PVCs that will be used by the cluster node pods. So, in the pod definition we are able to see a persistentVolumeClaim volume named data-volume and mounted on '/opt/infinispan/server/data' (the path used by infinispan to store the temporary data for example when the cluster is turned off). In this case if we turn off the cluster, we are able to recover our data after the cluster restart if the persistent storage size is greater then the amount of Datagrid memory.

If we use the ephemeral storage, in the pod definition we are able to see an emptyDir volume named data-volume and always mounted on '/opt/infinispan/server/data'. In this case if we turn off the cluster, we will lost our data completely.

When you start a cluster with a storage type you cannot change that. You can only delete and recreate the cluster with a different storage type.

## User authentication/authorization

By default a secret named 'infinispan-generated-secret' is created and populated with user credentials to access to cluster data. To see the credentials:
```bash
oc -n datagrid-demo get secret infinispan-generated-secret -o jsonpath="{.data.identities\.yaml}" | base64 --decode
```
or
```bash
oc -n datagrid-demo extract secret/infinispan-generated-secret --to=-
```

the secret key identities.yaml contains a developer user, its password and its roles.
```yaml
credentials:
- username: developer
  password: KkpnfqAQxHwVXLTt
  roles:
  - admin
  - controlRole
```

You can create another secret with a key identities.yaml containing the needed users and the relative roles. First you have to create a identities.yaml file with the new user credentials and then you have to create the secret from it:

```bash
oc create secret generic --from-file=identities.yaml my-user-secret
```

Finally you can reference the secret in the Infinispan CR 'spec.security.endpointSecretName' field.

To enable/disable the authentication we have to use the Infinispan CR 'spec.security.endpointAuthentication' field. 

To enable/disable the authorization we have to use the Infinispan CR 'spec.security.authorization.enabled' field.

The default available roles are [here](https://docs.redhat.com/en/documentation/red_hat_data_grid/8.5/html/data_grid_operator_guide/authorization#user-roles-permissions_authorization).

To add custom roles we can follow this [guide](https://docs.redhat.com/en/documentation/red_hat_data_grid/8.5/html/data_grid_operator_guide/authorization#adding-custom-roles-permissions_authorization).

For client certificate authentication we can follow this [guide](https://docs.redhat.com/en/documentation/red_hat_data_grid/8.5/html/data_grid_operator_guide/client-certificates).

To encrypt connections between clients and Datagrid we can use Red Hat OpenShift service certificates (the default case) or custom TLS certificates. The client encryption can be configured using the section 'spec.security.endpointEncryption'. We can disable the client encryption completely using 'spec.security.endpointEncryption.type: None'. To enable it we have to set 'spec.security.endpointEncryption.type: Service'. The Red Hat TLS Certificate are generated and stored in a secret named 'infinispan-cert-secret'. To retrieve the Red Hat TLS Certificate and give them to the clients, we can use this command:
```bash
oc get secret infinispan-cert-secret -o jsonpath='{.data.tls\.crt}' | base64 --decode > tls.crt
```
To use custom encryption certificates, we can follow this [guide](https://docs.redhat.com/en/documentation/red_hat_data_grid/8.5/html/data_grid_operator_guide/configuring-encryption#using-custom-encryption-secrets_tls).

## Monitoring Datagrid
The Datagrid monitoring is enabled by default, but to see the Datagrid metrics on Openshift (enabling the Openshift Prometheus to scrape its metrics) we have to enable the Openshift user workload monitoring using this command:
```bash
oc apply -f resources/monitoring/01.cluster-monitoring-config-cm.yml
```
After that we are able to see for example the metric 'vendor_cache_manager_default_cluster_size' in the Openshift Web Console Observe/Metrics section.

If you want to disable Datagrid monitoring, you have to change the annotation 'infinispan.org/monitoring' value to 'true' in the Infinispan CR metadata.

## Cache types

Two main types:

**Distributed caches**: Maximize capacity by creating fewer copies of each entry across the cluster.

**Replicated caches**: Provide redundancy by creating a copy of all entries on each node in the cluster.

Consider whether your applications perform more write operations or more read operations. In general, distributed caches offer the best performance for writes while replicated caches offer the best performance for reads.

To put k1 in a distributed cache on a cluster of three nodes with two owners, Data Grid writes k1 twice. The same operation in a replicated cache means Data Grid writes k1 three times. The amount of additional network traffic for each write to a replicated cache is equal to the number of nodes in the cluster. A replicated cache on a cluster of ten nodes results in a tenfold increase in traffic for writes and so on. You can minimize traffic by using a UDP stack with multicasting for cluster transport.

To get k1 from a replicated cache, each node can perform the read operation locally. Whereas, to get k1 from a distributed cache, the node that handles the operation might need to retrieve the key from a different node in the cluster, which results in an extra network hop and increases the time for the read operation to complete.

## Cache memory types

Data Grid stores cache entries in JVM heap memory by default. You can configure Data Grid to use off-heap storage, which means that your data occupies native memory outside the managed JVM memory space.

**JVM heap memory**

The heap is divided into young and old generations that help keep referenced Java objects and other application data in memory. The GC process reclaims space from unreachable objects, running more frequently on the young generation memory pool.

When Data Grid stores cache entries in JVM heap memory, GC runs can take longer to complete as you start adding data to your caches. Because GC is an intensive process, longer and more frequent runs can degrade application performance.

**Off-heap memory**

Off-heap memory is native available system memory outside JVM memory management. The JVM memory space diagram shows the Metaspace memory pool that holds class metadata and is allocated from native memory. The diagram also represents a section of native memory that holds Data Grid cache entries.

Off-heap memory:
- Uses less memory per entry.
- Improves overall JVM performance by avoiding Garbage Collector (GC) runs.
- One disadvantage, however, is that JVM heap dumps do not show entries stored in off-heap memory.

## Auto Scaling using HPA

TODO

## Calculating the correct distributed cache size

Firstly, we have to calculate the rough data set size using this formula:

```
Data set size = Number of entries * (Average key size (in its marshalled form) + Average value size (in its marshalled form) + Memory overhead)
```

Memory overhead is additional memory that Data Grid uses to store entries. An approximate estimate for memory overhead is **200 bytes per entry in JVM heap memory** or **60 bytes per entry in off-heap memory**.

It's impossible estimate precisely the memory overhead because its amount depends by many factors. For example, configuring cache boundaries (counters) or cache expiration policies (timestamps) add extra memory overhead. The only way to find any kind of exact amount of memory overhead involves JVM heap dump analysis. Of course JVM heap dumps provide no information for entries that you store in off-heap memory but memory overhead is much lower for off-heap memory than JVM heap memory.

In normal operating conditions, distributed caches store a number of copies for each key/value entry that is equal to the Number of owners that you configure. During cluster rebalancing operations, some entries have an extra copy, so you should calculate Number of owners + 1 to allow for that scenario.

```
Distributed data set size = Data set size * (Number of owners + 1)
```

Distributed caches allow you to increase the data set size either by adding more nodes or by increasing the amount of available memory per node.

```
Distributed data set size <= Available memory per node * Minimum number of nodes

Minimum number of nodes >= Distributed data set size / Available memory per node
```

Distributed caches tolerate the loss of (Number of owners - 1) nodes without losing data, so you can allocate that many extra node in addition to the minimum number of nodes that you need to fit your data set.

```
Planned nodes = Minimum number of nodes + Number of owners - 1
```

so that

```
Planned nodes = Distributed data set size / Available memory per node + Number of owners - 1
```

In case of a Data set size equals to 10GB, a wished Number of owners equals to 3, and an Available memory per node equals to 4GB, the Planned nodes for a distributed cache are:

```
Distributed data set size = Data set size * (Number of owners + 1) = 10GB * (3 + 1) = 40GB
Planned nodes = Distributed data set size / Available memory per node + Number of owners - 1 = 40GB / 4GB + 3 - 1 = 10 + 3 - 1 = 12
```

## Calculating the correct replicated cache size

The Data set size calculation is the same of the distributed cache case:
```
Data set size = Number of entries * (Average key size (in its marshalled form) + Average value size (in its marshalled form) + Memory overhead)
```

A replicated cache replicates every entry in all nodes and for this reason it tolerates the loss of (Number of nodes - 1) nodes without losing data.
```
Replicated data set size = Data set size * Number of nodes
```

Each datagrid node must provide a memory greater than Data set size (you have to consider the base memory used by the datagrid application):
```
Available memory per node > Data set size
```

## Cache encoding

Encoding is the format, identified by a media type, that Data Grid uses to store entries (key/value pairs) in caches.

Data Grid Server stores entries in remote caches with the encoding that is set in the cache configuration.

If the remote cache does not have any encoding configuration, Data Grid Server stores keys and values as generic byte[] without any media type information, which can lead to unexpected results when converting data for clients request different formats.

Data Grid Server returns an error when client requests include a media type that it cannot convert to or from the media type that is set in the cache configuration.

Data Grid recommends always configuring cache encoding with the application/x-protostream media type if you want to use multiple clients, such as Data Grid Console or CLI, Hot Rod, or REST. ProtoStream encoding also lets you use server-side tasks and perform indexed queries on remote caches.

In the Cache CR
- to use the same mediaType for entry key and value
  ```yaml
  ...
        encoding: 
          mediaType: "application/x-protostream"
  ...
  ```
- to use the different mediaTypes for entry key and value
  ```yaml
  ...
        encoding:
          key:
            mediaType: "text/plain"
          value:
            mediaType: "application/x-protostream"
  ...
  ```

Data Grid includes an implementation of the ProtoStream API with native support for frequently used types, including String and Integer. If you want to store custom types in your caches, use ProtoStream marshalling to generate and register serialization contexts with Data Grid so that it can marshall your objects.