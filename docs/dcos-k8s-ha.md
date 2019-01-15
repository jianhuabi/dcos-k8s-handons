# How Kubernetes Cluster on DC/OS implement High Availability

DCOS kubernetes cluster deployment offers customer with production grade high available environment by default. In this article, I am trying to demonstrate how DCOS kubernetes deployment addresses control plane HA in production deployment. 



## Kubernetes Cluster HA design in DCOS ##

The High Availability deployment of K8s cluster is an effort of a series of clustering technologies based upon the k8s control plane components.

Let’s review kubernetes cluster control plane components design in below.


![image.png](https://s3-us-west-2.amazonaws.com/github-png/k8s-cluster.png)

The kubernetes cluster control plane are implemented with Etcd service, kube apiserver, kube controller manager and kube sheduler. Hence, the kubernetes cluster control plane high availability really is relying on each of those components high availability deployment.




## Etcd service in DCOS ##

Etcd is a consensus-based distributed system, whose core state is maintained via the raft algorithm.  To achieve high availability of the system the cluster must be deployed into an odd number of servers typically three or five in an HA configuration.  As of today, most deployments are statically configured via some Configuration Management tooling.


DCOS K8s cluster HA deployment automatically launches 3 etcd instances by default with placement constraints set as UNIQUE to make sure etcd instances are landed on different hosts resources.

![image.png](https://s3-us-west-2.amazonaws.com/github-png/etcd-ha-0.png)

Each Etcd instance will get a mesos-dns with its form in below.

```
etcd-<id>-peer.<kubernetes-cluster-name>.autoip.dcos.thisdcos.directory
```

```
dcos node ssh --master-proxy --leader 'nslookup etcd-0-peer.kubernetes-cluster1.autoip.dcos.thisdcos.directory'

;; Got recursion not available from 198.51.100.1, trying next server
;; Got recursion not available from 198.51.100.2, trying next server
Server:		198.51.100.3
Address:	198.51.100.3#53

Name:	etcd-0-peer.kubernetes-cluster1.autoip.dcos.thisdcos.directory
Address: 9.0.1.2
```

The k8s apiservice will be configured with Etcd instances against to its mesos-dns name, a example is listed in below.

```
--etcd-servers=
https://etcd-0-peer.kubernetes-cluster1.autoip.dcos.thisdcos.directory:2379,
https://etcd-1-peer.kubernetes-cluster1.autoip.dcos.thisdcos.directory:2379,
https://etcd-2-peer.kubernetes-cluster1.autoip.dcos.thisdcos.directory:2379
```

So the apiservice will remain running even if a etcd instance gets replaced by a newly created instance in case of a failure or upgrade action.

## API-Server in DCOS: ##

The apiserver will stay in active-active mode, primarily due to fixes that have occurred in the etcd-3.1 timeframe around quorum reads.  But we will dig into some details regarding load-balancing constraints.


Nodes joining the cluster leverage a front end or client-side load-balancer for a couple of reasons.  First and foremost is to prevent the nodes from needing to maintain connection awareness of the api-servers.  This is also done for certificate simplicity as nodes are only aware of a single cert with the ip-address/hostname, or vip, of the load-balancer.


In DCOS k8s, we leverage client-side load-balancer which is IPVS to expose apiserver to other components including the controller manager, scheduler and kubelet nodes.

![image.png](https://s3-us-west-2.amazonaws.com/github-png/apiserver-ha-0.png)

As depicted in the diagram, DCOS will assign IP address as well as mesos-dns record to each kube control plan instance. Upon receiving metadata of kube control plan instance, the DCOS minuteman will generate IPVS record for apiserver instances which is advertised in the form as below.


```
apiserver.kubernetes-cluster1.l4lb.thisdcos.directory
```

Kubenetes components like scheduler, controller manager as well as kubelet nodes shall reference this DNS name to access kubernetes apiserver serivce.

## Scheduler and Controller manager in DCOS: ##

Kubernetes scheduler and Controller manager are deployed as static pod and hence managed by kubernetes cluster itself.

Scheduler and Controller manager HA design, therefore, adheres to kubernetes original architecture design proposed by the community.
