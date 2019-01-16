# Deep dive into Kubernetes Simple Leader Election

I always think Kubernetes Controller Manager or Scheduler components are leveraging etcd to perform leader election ever since
the I learn these component should have leader in HA mode. But recently, when I tried to review Kubernetes Controller Manager
**config.yaml**  I suddenly noticed there is actualy no command flag for adding **ectd connectstring**. 

I decided to ask Google for any information around mechanism of K8s control plane component leader election. There are good
stuff online such as [Simple leader election with Kubernetes and Docker](https://kubernetes.io/blog/2016/01/simple-leader-election-with-kubernetes/)
but the leader election mechanism kubernetes performs is confusing. Following statements were what I copied from the 
formentioned blog. 

```
## Implementing leader election in Kubernetes

The first requirement in leader election is the specification of the set of candidates for becoming the leader. 
Kubernetes already uses Endpoints to represent a replicated set of pods that comprise a service, so we will 
re-use this same object. (aside: You might have thought that we would use ReplicationControllers, but they are 
tied to a specific binary, and generally you want to have a single leader even if you are in the process of 
performing a rolling update)

To perform leader election, we use two properties of all Kubernetes API objects:

    ResourceVersions - Every API object has a unique ResourceVersion, and you can use these versions to perform 
    compare-and-swap on Kubernetes objects
    Annotations - Every API object can be annotated with arbitrary key/value pairs to be used by clients.

```
