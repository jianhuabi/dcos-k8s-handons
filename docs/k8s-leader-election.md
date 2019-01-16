# Deep dive into Kubernetes Simple Leader Election

I always think Kubernetes Controller Manager or Scheduler components are leveraging **etcd** to perform leader election ever since I learned these components should have their leader in HA mode. But recently, when I tried to review Kubernetes Controller Manager
**config.yaml**  I suddenly noticed there is actualy no such command flag for adding **etcd connectstring**. 

I decided to ask Google for any information around mechanism of K8s control plane components leader election. There are good
stuffs online such as [Simple leader election with Kubernetes and Docker](https://kubernetes.io/blog/2016/01/simple-leader-election-with-kubernetes/)
but the leader election mechanism that kubernetes performs is confusing. Following statements were what I copied from the 
formentioned blog. 

```
## Implementing leader election in Kubernetes

The first requirement in leader election is the specification of the set of candidates for becoming 
the leader. 

Kubernetes already uses Endpoints to represent a replicated set of pods that comprise a service, 
so we will re-use this same object. (aside: You might have thought that we would use 
ReplicationControllers, but they are tied to a specific binary, and generally you want to have a 
single leader even if you are in the process of performing a rolling update)

To perform leader election, we use two properties of all Kubernetes API objects:

    ResourceVersions - Every API object has a unique ResourceVersion, and you can use these 
    versions to perform compare-and-swap on Kubernetes objects
    Annotations - Every API object can be annotated with arbitrary key/value 
    pairs to be used by clients.

```

That was interesting but how comes a ResourceVersion & Endpoint & Annotation can be a leader election mechanism for service in k8s?

I coundn't find any fine explaination online, that is a shame. So I decide to find the answer in leader election code. 

## Who is leader of K8s controller manager. 

Before jumping into code implemetation, I learned from the blog on how to inspect who is the leader of a leader election enabled service like k8s controller manager.

In kubernetes, any leader enabled service will generate a EndPoint with a annotation suggests leadership in the service. Take k8s controller manager as example. 

```
> kubectl describe ep -n kube-system kube-controller-manager

Name:         kube-controller-manager
Namespace:    kube-system
Labels:       <none>
Annotations:  control-plane.alpha.kubernetes.io/leader:
                {"holderIdentity":"a3e0b5e2-e869-488d-9c14-49a60f3878df_a69decf6-192d-11e9-8a88-e6202bae2e50","leaseDurationSeconds":15,"acquireTime":"201...
Subsets:
Events:  <none>
```
The endpoint annotation suggests **Annotations: control-plane.alpha.kubernetes.io/leader** that instance who's identity or hostname is **a3e0b5e2-e869** currently is the leader. 

```
{
  "holderIdentity": "a3e0b5e2-e869-488d-9c14-49a60f3878df_a69decf6-192d-11e9-8a88-e6202bae2e50",
  "leaseDurationSeconds": 15,
  "acquireTime": "2019-01-16T01:25:47Z",
  "renewTime": "2019-01-16T07:30:31Z",
  "leaderTransitions": 0
}
```

That JSON object give us some clue on k8s leader election. 

## K8S leaderelection.go

K8s leader election package is hosted at [leaderelection.go](https://raw.githubusercontent.com/kubernetes/contrib/master/election/vendor/k8s.io/kubernetes/pkg/client/leaderelection/leaderelection.go)

```

// keep in mind, this struct will be added to a service endpoint as value of  
// Annotations: control-plane.alpha.kubernetes.io/leader 

leaderElectionRecord := LeaderElectionRecord{
    HolderIdentity:       le.config.Identity,
    LeaseDurationSeconds: int(le.config.LeaseDuration / time.Second),
    RenewTime:            now,
    AcquireTime:          now,
}

....
// Instance want to check if a endpoint is created/existed in certain Namespace.

e, err := le.config.Client.Endpoints(le.config.EndpointsMeta.Namespace).Get(le.config.EndpointsMeta.Name)
if err != nil {
    if !errors.IsNotFound(err) {
        glog.Errorf("error retrieving endpoint: %v", err)
        return false
    }

    leaderElectionRecordBytes, err := json.Marshal(leaderElectionRecord)
    if err != nil {
        return false
    }
    _, err = le.config.Client.Endpoints(le.config.EndpointsMeta.Namespace).Create(&api.Endpoints{
        ObjectMeta: api.ObjectMeta{
            Name:      le.config.EndpointsMeta.Name,
            Namespace: le.config.EndpointsMeta.Namespace,
            Annotations: map[string]string{
                LeaderElectionRecordAnnotationKey: string(leaderElectionRecordBytes),
            },
        },
    })
    if err != nil {
        glog.Errorf("error initially creating endpoints: %v", err)
        return false
    }
    le.observedRecord = leaderElectionRecord
    le.observedTime = time.Now()
    return true
}

...
leaderElectionRecordBytes, err := json.Marshal(leaderElectionRecord)

if err != nil {
    glog.Errorf("err marshaling leader election record: %v", err)
    return false
}
//
// const (
//	JitterFactor = 1.2
//
//	LeaderElectionRecordAnnotationKey = "control-plane.alpha.kubernetes.io/leader"
//
//	DefaultLeaseDuration = 15 * time.Second
//	DefaultRenewDeadline = 10 * time.Second
//	DefaultRetryPeriod   = 2 * time.Second
// )

e.Annotations[LeaderElectionRecordAnnotationKey] = string(leaderElectionRecordBytes)

// Instance create or Update Endpoints to claim leader role.
_, err = le.config.Client.Endpoints(le.config.EndpointsMeta.Namespace).Update(e)
if err != nil {
    glog.Errorf("err: %v", err)
    return false
}
le.observedRecord = leaderElectionRecord
le.observedTime = time.Now()
...
```

This code snippet gives me some clue on how k8s implement leader election. It obviously leverage k8s endpoint resource as **LOCK primitive**.  If a service is required to elect a leader in k8s, the instance in this service will compete to LOCK (via Create/Update) an EndPoint resource of this service. 

```
// LeaderElectionRecordAnnotationKey = "control-plane.alpha.kubernetes.io/leader"
e.Annotations[LeaderElectionRecordAnnotationKey] = string(leaderElectionRecordBytes)

le.config.Client.Endpoints(le.config.EndpointsMeta.Namespace).Update(e)

le.observedRecord = leaderElectionRecord
le.observedTime = time.Now()

...
```

So that make sense to me now, an instance in a service claims leader role by create an endpoint and add an annotation of **control-plane.alpha.kubernetes.io/leader**. The service endpoint is used as a Locker to prevent any follower to create same endpoint in this same Namespace. 

That is awesome!!

## Summary of K8S simple leader election

Implementation is boring. Simply put, K8s simple leader election follows below steps. 

1. Instance who firstly creates a service endpoint is the leader of this service at very beginning. This instance then adds **control-plane.alpha.kubernetes.io/leader** annotation to expose leader information to other followers or application in the cluster. 

2. The leader instances shall constantly renew its lease time to indicate its existence. In below, the leaseDuration is 15 seconds. Leader will update **renewTime** before lease duration expired.

```
{
  "holderIdentity": "a3e0b5e2-e869-488d-9c14-49a60f3878df_a69decf6-192d-11e9-8a88-e6202bae2e50",
  "leaseDurationSeconds": 15,
  "acquireTime": "2019-01-16T01:25:47Z",
  "renewTime": "2019-01-16T07:30:31Z",
  "leaderTransitions": 0
}
```

3. Followers will constantly check existence of service endpoint and if it is created already by leader, the followers will do 
additional **renewTime** check against current time. If **renewTime** is older than Now which means leader failed to update its leaseDuration, hence suggests Leader is crashed or something bad happened otherwise. In that case, a new leader election process is started until a follower successfully claims leadership by update endpoint with its Identity and leaseDuration. 


Probably there are other edge case in implemetation details but it doesn't concerns me just now.

If you think there are something wrong in this article, please let me know. 

Thanks in advance. 

